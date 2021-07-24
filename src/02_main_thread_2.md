# 主线程 - 处理连接

## TCP accept

worker thread 还没讲到，先跳过收到 event 的处理，假设我们现在收到了连接，并通过信号量被唤醒(`unpark`)。然后就从 `park()` 返回，接着调用 `f.as_mut().poll`，也就是 async main。之前讲过 Rust 会把 future 编译为一个状态机，所以当 async main 这次被调用时，并不会从头开始执行，而是从 Readiness 的 poll 方法开始执行：

```rust
// impl Future for Readiness<'_> {
//     fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
loop {
    match *state {
				State::Waiting => {
						let w = unsafe { &mut *waiter.get() };

		        if w.is_ready {
								*state = State::Done;
		        } else {
								// ...
		        }

						drop(waiters);
				}
				State::Done => {
						let tick = TICK.unpack(scheduled_io.readiness.load(Acquire)) as u8;

            // Safety: State::Done means it is no longer shared
            let w = unsafe { &mut *waiter.get() };

            return Poll::Ready(ReadyEvent {
                tick,
                ready: Ready::from_interest(w.interest),
            });
				}
		}
}
```

`state` 现在已经是 `Waiting`，因为 worker 线程已经把 waiter 的 `is_ready` 改为了 true，所以又修改 `state` 为 `Done` 并继续循环，构造了 `ReadyEvent` ，并返回。 `ReadyEvent` 的 tick 在 #6 中会讲，而 `ready` 则是表示具体是 ready 还是 write ready。现在 `async_io` 中的 `self.readiness(interest).await?` 就可以返回了。

再来看 `async_io` ：

```rust
// async fn async_io:
loop {
    let event = self.readiness(interest).await?;

		match f() {
        Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
            self.clear_readiness(event);
        }
        x => return x,
    }
}

-------------------------------
// f(): （listener.accept 中)
// .async_io(Interest::READABLE,
|| self.io.accept()
// )

-------------------------------
// self.io.accept():
sys::tcp::accept(inner).map(|(stream, addr)| (TcpStream::from_std(stream), addr))
```

`readiness.await` 返回后就表示这个 readiness 已经 *ready*，于是在 `f()` 中读数据（accept）。 `accept`和 bind 类似，调用了 mio 的  `tcp::accept`，基本上是系统调用，并封装为 `std::TcpStream` ，然后用它来构造 mio 的 `TcpStream` ，再用 mio TcpStream 来初始化 tokio 的 `TcpStream`:

```rust
pub async fn accept(&self) -> io::Result<(TcpStream, SocketAddr)> {
    let (mio, addr) = self.io.registration()
        .async_io(Interest::READABLE, || self.io.accept())
        .await?;

    let stream = TcpStream::new(mio)?;
    Ok((stream, addr))
}
```

`TcpStream::new` 和 `TcpListener::new` 几乎是一样的，也是先通过 Slab 申请 ScheduledIO 资源、注册 event poll、返回 `PollEvented<T>`。唯一的不同是 `PollEvented` 泛型的 T(`io`字段) 是 mio TcpStream 而不是 mio TcpListener。`PollEvented` 会被用来注册 IO 事件以及读写数据等， `io` 字段不同就意味着，底层读写数据等的实现不同，但对于 PollEvented 来说都是统一的 read/write 接口，因此可以用泛型来实现。

建立一个连接后，我们会得到一个 TcpStream，之后可以用来在这个 TCP 连接上收发数据。

## spawn task

之后通过 `tokio::spawn` 在一个 task 中处理这个连接：

```rust
tokio::spawn(async move {
  // ...
})
```

这里会先从 thread local 中拿到线程池 spawner 的 handle，把 async block 封装为一个 tokio [task](https://docs.rs/tokio/1.6.1/tokio/task/index.html)，再调度这个 task：

```rust
// tokio::spawn:
let spawn_handle = runtime::context::spawn_handle();
let (task, handle) = task::joinable(future);
spawn_handle.shared.schedule(task, false);
```

 `schedule` 代码如下所示：

```rust
CURRENT.with(|maybe_cx| {
		// ... may schedule to local(only in worker threads)

    // inject: global queue
    self.inject.push(task);
    self.notify_parked();
});

// notify_parked:
if let Some(index) = self.idle.worker_to_notify() {
    self.remotes[index].unpark.unpark();
}
```

task （在这个情况下）会被放到 global queue 中，然后通知 worker 线程。scheduler 会找一个 idle 的线程来通知，并通过调用 `unpark` 来唤醒它，就是 #1 中图里的 wake 2。这里的 `remotes` 就是初始化线程池时创建的，主要用做其他线程和线程池中的线程通信，比如这里是主线程要访问 worker 线程中的 `unpark`：

```rust
// https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/park.rs#L214
match self.state.swap(NOTIFIED, SeqCst) {
    EMPTY => {}    // no one was waiting
    NOTIFIED => {} // already unparked
    PARKED_CONDVAR => self.unpark_condvar(),
    PARKED_DRIVER => self.unpark_driver(),
}
```

因为 worker thread 在 park 时会根据需要情况选择不同的 park 方式，所以 unpark 时也要执行对应的方法。在这里就是 `unpark_driver`，会调用 io driver 的方法，也就是 `inner.waker.wake()`，然后会调用 mio 的 `wake` 方法来通过 IO 事件唤醒响应的 worker 线程：

```rust
// unpark_driver:
fn unpark_driver(&self) {
    self.shared.handle.unpark();
}

-------------------------------------
// https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/io/driver/mod.rs#L292
// self.shared.handle.unpark():
if let Some(inner) = self.inner() {
    // in io::Driver.new:
		// waker = mio::Waker::new(poll.registry(), TOKEN_WAKEUP)?;
    inner.waker.wake().expect("failed to wake I/O driver");
}
```

主线程把处理 TCP 连接的 async task 调度之后，就在循环中又执行 echo 代码中的 `accept`，去 poll readiness，和之前不太一样的是，因为（对于 linux 的 epoll）是边缘触发，readiness 并没有被修改还是 ready 状态，于是继续在 `f()` 中尝试读数据，但如果当前没有连接，就会得到 `WouldBlock`，于是会清除 readiness，然后又 poll readiness，这时返回 Pending，最后继续 block_on 的 loop 并 park。

```rust
loop {
    let event = self.readiness(interest).await?;

    match f() {
        Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
            self.clear_readiness(event);
        }
        x => return x,
    }
}
```

## 总结

这一章比较简单，主线程被唤醒后，先完成 TCP accept，再调度 task，最后又回去继续执行 async main 来等待新连接。

![](./assets/02_main_2.png)
[https://excalidraw.com/#json=4958173064593408,RWwZeg19iXYek188c9wzOg](https://excalidraw.com/#json=4958173064593408,RWwZeg19iXYek188c9wzOg)

至此，我们已经以 TCP 建立连接的过程为例，介绍完了主线程的执行流程、如何注册 IO 事件、如何启动一个异步的 task。接下来，我们将重点介绍 worker 线程如何处理事件和执行 task。
