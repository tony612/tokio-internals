# Signal, Process 和 Time

我们之前在讲 Tokio runtime 的时候为了简化，特地关掉了 [`time`](https://docs.rs/tokio/1.7.1/tokio/time/index.html), [`process`](https://docs.rs/tokio/1.7.1/tokio/process/index.html) 和 [`signal`](https://docs.rs/tokio/1.7.1/tokio/signal/index.html) 这三个 feature，现在来讲一下，他们分别对应三个 module，用来处理异步的时间、进程管理和信号处理。

还记得在 #2 中讲过的 `runtime::driver::Driver` 吗？它被用来创建 `runtime::Parker`，并且在 #5 中当 worker 没有 task 要处理而 poll events 时就会调用 `Parker` 的 [`park` 方法](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/park.rs#L92)，在其中会进一步调用 `driver` 的 `park`:

```rust
driver.park()
```

这个 driver 就是 `runtime::driver::Driver`，之前只是把它当做 `io::driver::Driver`来讲，但如果开了 `time`, `process` 和 `signal` 这三个 feature 后，Driver 其实是一个嵌套的结构：

```rust
runtime::driver::Driver {
    inner: time::driver::Driver {
        park: process::unix::driver::Driver {
            park: signal::unix::driver::Driver {
                park: io::Driver,
            },
        }
    }
}
```

这里的每一层 Driver 都实现了 `Park` 这个 trait，其中包含了`park` 方法，当 runtime driver 的 `park` 被调用时，不断调用下一层的 `park`，而且逻辑也都差不多，在调用 `park` 后还会调用自己的处理逻辑。

```rust
impl Park for runtime::Driver::Driver {
		fn park(&mut self) -> Result<(), Self::Error> {
        self.inner.park()      // call time driver's park
    }
}

impl<P> Park for time::driver::Driver<P> {
		fn park(&mut self) -> Result<(), Self::Error> {
				// ... preprocess for time
				// may call self.park.park_timeout(duration)?;
				self.park.park()?;     // call process driver's park

				self.handle.process();
    }
}

impl Park for process::unix::driver::Driver {
		fn park(&mut self) -> Result<(), Self::Error> {
        self.park.park()?;      // call signal driver's park
        self.inner.process();
        Ok(())
    }
}

impl Park for signal::unix::driver::Driver {
		fn park(&mut self) -> Result<(), Self::Error> {
        self.park.park()?;      // call io driver's park
        self.process();
        Ok(())
    }
}

impl Park for io::Driver {
		fn park(&mut self) -> io::Result<()> {
        self.turn(None)?;
        Ok(())
    }
}
```

嵌套的 driver 形成了依赖关系，最底层的 io driver 最基础的，signal 依赖 IO，process 依赖 signal，time 有点不太一样，其实是依赖 IO。

## Signal driver

我们从底向上，先来看 signal driver。

### Signal 注册 IO 事件

在 signal driver 创建时，会创建一个 unix stream socket，获得 receiver 和 sender，并且通过 mio 向 epoll/kqueue 注册 io 事件：

```rust
let (receiver, sender) = UnixStream::pair()

let receiver = PollEvented::new_with_interest_and_handle(
    receiver,
    Interest::READABLE | Interest::WRITABLE,
    park.handle(),
)?;
```

当 signal driver `park` 时就会等待这个 unix socket receiver 的 IO 事件。

再来看 signal 订阅事件（类似于 TcpListener 的 `accept` ），以及接收 signal 广播的代码，以文档上的[这个例子](https://docs.rs/tokio/1.7.1/tokio/signal/index.html#examples)为示：

```rust
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // An infinite stream of hangup signals.
    let mut stream = signal(SignalKind::hangup())?;

    // Print whenever a HUP signal is received
    loop {
        stream.recv().await;
        println!("got signal HUP");
    }
}
```

在 `signal(SignalKind)` 中会通过 `signal_hook_registry` 这个 crate 来注册 signal callback：

```rust
signal_hook_registry::register(signal, move || action(globals, signal))

let (tx, rx) = watch::channel(());
Signal {
    inner: RxFuture::new(rx),
}
```

并且会创建一个 `sync::watch::Channel`，channel 的 sender(tx) 会放在 thread local 中，rx 会被放在 `stream`（ `signal::unix::Signal`）中从 `signal` 调用中返回，当调用 `stream.recv().await` 时其实是调用的 `rx.changed().await` 来等待 `tx` 被写入消息。

### Signal 被唤醒

当收到系统 signal 时，回调函数 `action(globals, signal)` 就会被调用，并通过 `sender` 向之前的 unix socket 写数据：

```rust
fn action(globals: Pin<&'static Globals>, signal: c_int) {
    globals.record_event(signal as EventId);

    let mut sender = &globals.sender;
    drop(sender.write(&[1]));
}
```

sender 被写了数据后，reactor 就会被唤醒，io driver 从 `park` 中返回，于是 signal driver 会在 `self.process()` 中通知 signal 的 listeners(`globals().broadcast()`)：

```rust
fn process(&self) {
	let ev = match self.receiver.registration().poll_read_ready(&mut cx) {
	    Poll::Ready(Ok(ev)) => ev,
	}
	self.receiver.registration().clear_readiness(ev);

	// Broadcast any signals which were received
	globals().broadcast();
}
```

 `globals().broadcast()` 中其实就是调用 `tx.send()` 来通知 `stream.recv().await` 继续。

## Process driver

先来看下异步 process 的基本用法：

```rust
let mut child = Command::new("echo").arg("hello").spawn()
        .expect("failed to spawn");

// Await until the command completes
let status = child.wait().await?;
```

Command 用法和标准库中的 Command 基本一致，只是提供了接口来异步地等到子进程执行完成。

process driver 之所以依赖 signal 是因为 process 其实在创建时订阅了 `libc::SIGCHLD` 事件：

```rust
let sigchild = signal_with_handle(SignalKind::child(), park.handle())?;
let inner = CoreDriver {
    sigchild,
    orphan_queue: GlobalOrphanQueue,
};

Ok(Self { park, inner })
```

process 示例中的 `child.wait().await?` 内部其实就是异步等待 system signal，当收到 `SIGCHILD` signal 时，`child.wait().await?` 就会完成。同时，signal driver 会被唤醒而结束 `park`，于是 process driver 可以继续执行，在调用 `self.inner.process()` 时会做一些子进程的回收工作。

### Time driver

Tokio 的 time 模块提供了异步的时间处理函数，比如 `sleep(Duration::from_millis(100)).await`。为了能在未来某个时间点处理相关的任务，time driver 会调用 `park_timeout`，io driver 在 poll events 时会传入 timeout，即使没有收到 io 事件，也会在 timeout 后被唤醒。time driver 的 `park_timeout` 调用返回后，会调用自己的处理方法，来唤醒任务（具体逻辑这里先不展开，本质上和 signal 和 process 的差不多）。
