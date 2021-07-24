# Tokio runtime 启动

这是 [echo example](https://github.com/tokio-rs/tokio/blob/df10b68d47631c3053342c7cf0b1fab8786565b2/examples/echo.rs) 的代码：

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

和普通的同步代码不同，Tokio 需要我们写一个 async 的 main 函数，它主要是靠 `#[tokio::main]` 宏来生成代码，[文档](https://docs.rs/tokio/1.5.0/tokio/attr.main.html)已经写得很清楚了，这里就不再赘述，只要知道它会被编译成下边这样就行。其实我们也可以根据需要在自己的 main 函数中，调用 API 来完成 runtime 初始化，而不通过 Tokio 的默认 macro。

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

这是 `build()` 方法（做了简化，以后的代码示例也会做适当的简化）：

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
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/builder.rs#L540)

主要初始化了几个资源：

- `runtime::driver::Driver` 表示 event loop 的 driver，比如 IO event，[1.2](./01_intro_tokio.md) 中的 `poll events` 和 `events` 就是用这个来实现的。
- `ThreadPool` 就是 [1.2](./01_intro_tokio.md) 中的 worker，因为 poll events 是在 worker 中进行的，所以需要把 driver 传进去。当 worker 没有 task 可以执行时就会 `Parker` 的 "park"，park 其实是调用 driver 来等待事件。
- `blocking_poll` 是专门用来运行 blocking 任务的线程池，其中 `core_threads` 个线程是来运行 worker 的线程（因为每个 worker 自身也可以看做一个 blocking 的任务），剩下的是专门运行 blocking 任务的。
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

return Resources {
    io_handle: Handle {
        inner: Arc::downgrade(&io_driver.inner),
    },
    ...
};
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/io/driver/mod.rs#L114)

这里也初始化了多个资源：

- mio 的 `poll`，底层就是 epoll/kqueue 对象。
- `waker` 是向 `poll` 注册一个特殊的事件 `TOKEN_WAKEUP` 创建的，用来直接唤醒 worker 线程（[1.2](./01_intro_tokio.md) 图的 wake2），被唤醒后可以执行 task。
- `Slab` 类似于 Linux kernel 中的 [Slab](https://en.wikipedia.org/wiki/Slab_allocation)，可以为 object 分配内存，并返回一个地址。目前只有 IO driver 在使用 Slab，会分配 `ScheduledIo`，用来保存 IO 资源的状态和 waker 相关信息。之所以用 Slab 来，是因为当有大量 IO 事件产生和被清除时，Slab 可以减少内存碎片以及提高利用率。（细节在 [3.1](./03_slab_token_readiness.md) 中会讲）
- `io_driver` 包含了之前创建的资源，它的 handle（句柄）会被线程共享，用来访问一些数据，比如 poll、Slab 等等。

### Thread pool 初始化

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
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/thread_pool/mod.rs#L46)

worker 的初始化没有太多要说的，只要留意几个 struct 就好。

- `Core`。表示一个 worker 自己的数据，如 run_queue，park，通过 [`Box`](https://doc.rust-lang.org/std/boxed/index.html) 被分配在 heap 上。
- `Remote`。steal 是 Core 中 run_queue 的 handle 的 copy，用来让其他线程“偷”任务。
- `Shared`。所有 worker 共享，保存了 Remote，和 global 的 run_queue 等。最终会返回作为 `scheduler`。
- `Worker`。包含了之前的 shared 和 core，并放到了 `launch` 中返回。

而 blocking_pool 也是类似地初始化了一个 run queue 和一些状态，不过相对还要更简单一些。

### 启动 Thread pool

在 `runtime` 返回之前，会通过 `let _enter = crate::runtime::context::enter(handle.clone());` 把 thread local 的 `CONTEXT` 赋值为 `handle`，当 `_enter` 被 drop 时， `CONTEXT` 会被恢复回之前的值。之所以这样“绕”了一下，是因为 `runtime::spawn_blocking` 需要从当前 thread local 中获取 runtime，这在其他场景是必要的。之后启动所有的 worker 线程（`launch.launch()`），代码如下：

```rust
pub(crate) fn launch(mut self) {
    for worker in self.0.drain(..) {
        runtime::spawn_blocking(move || run(worker));
    }
}
```
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/thread_pool/worker.rs#L277)

`runtime::spawn_blocking` 调用时， `|| run(worker)` 匿名函数会被传进去，这其实就是 worker 线程要执行的逻辑。

如下，匿名函数会被包装为 `BlockingTask`，并被放在 blocking thread 的 run queue 中，这样当它运行时就会执行这个匿名函数。因为这时没有足够的线程，就会初始化一个新的 OS 线程（如果有 idle 的线程，就会通过 condvar 通知），并开始执行 blocking 线程的逻辑。每个 worker 都占用一个 blocking 线程，并在 blocking 线程中运行直到最后。

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
[link](https://github.com/tokio-rs/tokio/blob/a5ee2f0d3d78daa01e2c6c12d22b82474dc5c32a/tokio/src/runtime/handle.rs#L201)

## 总结

`runtime::Runtime` 现在就被构造好了，包含了 Tokio runtime 运行的几乎所有数据，如 io driver、线程池等等。另外 worker 线程也已经创建好并开始运行了，但让我们暂时放下 worker 线程，在下一章中先看主线程后续的执行，也就是本章第二段代码中 Runtime 的 `block_on` 方法。
