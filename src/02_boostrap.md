# Tokio runtime 启动

从这章开始，我们来以 echo example 为例详细看一下 Tokio 的源码。代码基于 tag tokio-1.5.0，且为了方便起见关闭了 `time`、`process` 和 `signal` 三个 feature，后边讲到的时候再开启。

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;  // listen

    loop {
        let (mut socket, _) = listener.accept().await?; // async wait for incoming tcp socket

        tokio::spawn(async move {                       // create async task and let Tokio process it
            let mut buf = vec![0; 1024];

            loop {                                      // read and write data back until EOF
                let n = socket.read(&mut buf).await?;   // async wait for incoming data

                if n == 0 { return; }

                socket.write_all(&buf[0..n]).await?;    // async wait socket is ready to write and write data
            }
        });
    }
}
```

这个 async main 主要是靠 `tokio::main` 来生成代码，[文档](https://docs.rs/tokio/1.5.0/tokio/attr.main.html)已经写得很清楚了，这里就不再赘述，只要知道它会被编译成这样就行：

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
				    let listener = TcpListener::bind("127.0.0.1:8080").await?;
            // ...
        })
}
```

## Runtime 初始化

这是 `build()` 方法（做了简化，以后的代码也会适当做简化）：

```rust
let (driver, resources) = driver::Driver::new(self.get_cfg())?;

let (scheduler, launch) = ThreadPool::new(core_threads, Parker::new(driver));
let spawner = Spawner::ThreadPool(scheduler.spawner().clone());

// Create the blocking pool
let blocking_pool = blocking::create_blocking_pool(self, self.max_blocking_threads + core_threads);
let blocking_spawner = blocking_pool.spawner().clone();

// Create the runtime handle
let handle = Handle {
    spawner,
    io_handle: resources.io_handle,
    blocking_spawner,
};

// Spawn the thread pool workers
let _enter = crate::runtime::context::enter(handle.clone());
launch.launch();

Ok(Runtime {
    kind: Kind::ThreadPool(scheduler),
    handle,
    blocking_pool,
})
```

主要初始化了几个资源：

- `runtime::driver::Driver` 表示 event loop，比如 IO event，#1 中的 `poll events` 和 `events` 就是用这个来实现的。
- `ThreadPool` 就是 #1 中的 worker，因为 poll events 是在 worker 中进行的，所以需要把 driver 传进去。当 worker 没有 task 可以执行时就会 `Parker` 的 "park"，park 其实是调用 driver 来等待事件。
- `blocking_poll` 是之前提到的，专门用来运行 blocking 任务的线程池，其中 `core_threads` 个线程是来运行 worker 的线程（因为每个 worker 自身也可以看做一个 blocking 的任务），剩下的是专门运行 blocking 任务的。
- runtime `handle` 和 `runtime` ，包含 driver 和线程池，最后会返回到 main 中。之后的章节，会把运行轻量级的线程称作 worker thread/线程，而把运行其他 blocking 任务的称作 blocking thread/线程。

### IO driver 初始化

我们来重点看一下 driver 的构造过程 `driver::Driver::new`：

```rust
let poll = mio::Poll::new()?;
let waker = mio::Waker::new(poll.registry(), TOKEN_WAKEUP)?;
let registry = poll.registry().try_clone()?;

let slab = Slab::new();
let allocator = slab.allocator();

let io_driver = Driver {
    tick: 0,
    events: Some(mio::Events::with_capacity(1024)),
    poll,
    resources: Some(slab),
    inner: Arc::new(Inner {
        resources: Mutex::new(None),
        registry,
        io_dispatch: allocator,
        waker,
    }),
};

return Resources { io_handle: io_driver.handle(), ... };

----------
// io_driver.handle():
Handle {
    inner: Arc::downgrade(&io_driver.inner),
}
```

这里也初始化了多个资源：

- `poll` 对象，底层就是 epoll/kqueue 对象
- 这里的 `waker` 是向 `poll` 注册一个特殊的事件 `TOKEN_WAKEUP` 创建的，用来直接唤醒 worker 线程（#1 图的 wake2），而不需要通过 io 事件，被唤醒后可以执行 task。
- `Slab` 类似于 Linux kernel 中的 [Slab](https://en.wikipedia.org/wiki/Slab_allocation)，可以为 object 分配内存，并返回一个地址。目前只有 IO driver 在使用 Slab，会分配 `ScheduledIo`，用来保存 IO 资源的状态和 waker 相关信息。之所以用 Slab 来，是因为当有大量 IO 事件产生和被清除时，Slab 可以减少内存碎片以及提高利用率。（细节在 #7 中会讲）
- `io_driver` 包含了之前创建的资源，它的 handle（句柄）会被线程共享，用来访问一些数据，比如 poll、Slab 等等。

### Thread pool 初始化和启动

```rust
// ThreadPool::new:
cores.push(Box::new(Core {
    run_queue,
		...
    park: Some(park),
}));

remotes.push(Remote {
    steal,
		...
    unpark,
});

let shared = Arc::new(Shared {
    remotes: remotes.into_boxed_slice(),
    inject: queue::Inject::new(),
		...
});

launch.0.push(Arc::new(Worker {
    shared: shared.clone(),
    index,
    core: AtomicCell::new(Some(core)),
}));
```

worker 的初始化没有太多要说的，只要留意几个 struct 就好。

- `Core`。表示一个 worker 自己的数据，如 run_queue，park，被分配在 heap 上。
- `Remote`。steal 是 Core 中 run_queue 的 handle copy，用来让其他线程“偷”任务。
- `Shared`。所有 worker 共享，保存了 Remote，和 global 的 run_queue 等。最终会返回作为 `scheduler`。
- `Worker`。包含了之前的 shared 和 core，并放到了 `launch` 中返回。

而 blocking_pool 也是类似地初始化了一个 run queue 和一些状态，不过相对还要更简单一些。

在 `runtime` 返回之前，会把 thread local 的 `CONTEXT` 赋值为 `handle`（`enter`），然后启动 worker 线程（`launch.launch()`），当 `_enter` 被 drop 时， `CONTEXT` 会被恢复回之前的值。之所以这样“绕”了一下，是因为 `runtime::spawn_blocking` 需要从当前 thread local 中获取 runtime，这在其他场景是必要的。代码如下：

```rust
pub(crate) fn launch(mut self) {
    for worker in self.0.drain(..) {
        runtime::spawn_blocking(move || run(worker));
    }
}
```

`runtime::spawn_blocking` 调用时， `|| run(worker)` 匿名函数会被传进去，这其实就是 worker 线程要执行的逻辑。

如下，匿名函数会被包装为 `BlockingTask`，并被放在 blocking thread 的 run queue 中，这样当它运行时就会执行这个匿名函数。因为这时没有足够的线程，就会初始化一个新的线程（如果有 idle 的线程，就会通过 condvar 通知），并开始执行 blocking 线程的逻辑。每个 worker 都对应一个 blocking 线程，并在 blocking 线程中运行直到最后。

```rust
// runtime::spawn_blocking:
let (task, _handle) = task::joinable(BlockingTask::new(func));

let mut shared = self.inner.shared.lock();
shared.queue.push_back(task);

let mut builder = thread::Builder::new(); // Create OS thread
// run worker thread
builder.spawn(move || {
    rt.blocking_spawner.inner.run(id);
})
```

## 总结

`runtime::Runtime` 现在就被构造好了，包含了 Tokio runtime 运行的几乎所有数据，如 io driver、线程池等等。另外 worker 线程也已经创建好并开始运行了，但让我们暂时放下 worker 线程，在下一章中先看主线程后续的执行，也就是 Runtime 的 `block_on` 方法。
