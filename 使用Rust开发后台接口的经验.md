# 使用Rust开发后台接口的经验

- [使用Rust开发后台接口的经验](#使用rust开发后台接口的经验)
  - [Rust编程语言设计理念](#rust编程语言设计理念)
    - [错误处理](#错误处理)
    - [风险代码](#风险代码)
    - [内存管理](#内存管理)
    - [面向对象](#面向对象)
    - [模板与宏](#模板与宏)
    - [链接](#链接)
    - [执行形式](#执行形式)
    - [包管理](#包管理)
    - [文档与测试](#文档与测试)
    - [多线程](#多线程)
  - [库选用](#库选用)
    - [Actix/Actix-Web](#actixactix-web)
      - [路由](#路由)
    - [Tracing/TracingSubscriber](#tracingtracingsubscriber)
    - [Serde/SerdeJson](#serdeserdejson)
    - [Tokio](#tokio)
    - [AsyncTrait](#asynctrait)
    - [Reqwest](#reqwest)
    - [TypedBuilder](#typedbuilder)
  - [如何处理Panic](#如何处理panic)
    - [主体思路](#主体思路)
    - [CatchUnwind](#catchunwind)
    - [JoinHandle](#joinhandle)
      - [Tokio中的JoinHandle](#tokio中的joinhandle)
      - [std::thread中的JoinHandle](#stdthread中的joinhandle)
    - [PanicHook](#panichook)

## Rust编程语言设计理念

### 错误处理
- 遇到无法处理的错误时，迅速崩溃并且显示错误信息
- 错误类型与返回值类型挂钩，错误数据保存在返回值中
- 强制处理错误，调用方需要选择处理错误或继续向上抛出

### 风险代码
- 与底层交互的代码需要标注unsafe
- 推荐将safe与unsafe分开处理而非混用

### 内存管理
- 不使用自动GC
- 强制处理生命周期问题
- 默认智能指针，裸指针需要开启unsafe模式

### 面向对象
- 不使用类型继承，只使用类似接口约束继承的Trait机制
- 着重Trait机制的可扩展性

### 模板与宏
- 模板专注泛型功能，极不推荐使用模板特化
- 注重宏的功能，过程宏和声明宏的能力强

### 链接
- 默认动态链接libc，静态链接rust库
- 可以静态链接libc，需要使用musl

### 执行形式
- 生成二进制机器码形式可执行文件，直接执行
- 不需要解释器/虚拟机

### 包管理
- 官方下场打造包管理器，力求简化代码库引入流程
- 编译脚本build.rs使用rust自身编写

### 文档与测试
- 提倡注释即文档，通过cargo doc文档工具基于注释生成文档
- 提倡单元测试，编译器提供单元测试工具

### 多线程
- 提倡注重线程安全，编写结构体时需要标注是否符合线程安全
- 不符合线程安全要求的类型在多线程场景会产生编译错误

## 库选用

### Actix/Actix-Web
Actix是Rust最知名的Web库之一，使用Actix是因为我用Actix入门，对Actix较为熟悉。

#### 路由
Actix通过注解形式标注一个函数的Router信息。
```rust
#[get("/gethealthcenter")]
pub async fn service_xxx() -> impl Responder {
    let json_info = serde_json::json!({});
    web::Json(json_info)
}
```
可以通过web::ServiceConfig来将Router信息注册到Server中
```rust
/// Use this function to initialize routers
pub fn router(cfg: &mut web::ServiceConfig) {
    cfg.service(service_xxx)
        .service(service_xxy)
        .service(service_xyx)
        .service(service_xyy)
        .service(service_yxx)
        .service(service_yxy)
        .service(service_yyx)
        .service(service_yyy);
}
```

### Tracing/TracingSubscriber
Tracing是Rust知名的日志库，同时也是Trace库，支持日志和Trace功能。

TracingSubscriber是基于Tracing库的工具，提供方便配置Tracing库的功能。
```rust
use tracing_appender::rolling;
use tracing_subscriber::{filter, fmt, prelude::*, util::SubscriberInitExt};

/// Use this function to initialize tracing.
pub fn tracing_config_initalize() {
    // register all of subscribers
    tracing_subscriber::registry()
        // register daily_logger
        .with({
            fmt::layer()
                .pretty()
                .with_target(true)
                .with_line_number(true)
                .with_ansi(false)
                .with_writer(
                    rolling::daily("log", "tracelog_daily.log")
                        .with_max_level(tracing::Level::INFO),
                )
        })
        // register console logger
        .with({
            fmt::layer()
                .with_target(true)
                .with_line_number(true)
                .with_ansi(false)
                .with_writer(std::io::stdout.with_max_level(tracing::Level::INFO))
        })
        // initialize the tracing subscriber
        .init();
}

```

TracingSubscriber提供了滚动日志的功能，但是不提供删除过期日志的功能。需要我们自己写工具。

```rust
    pub fn cleanup(path: &Path) -> Option<()> {
        let dir = std::fs::read_dir(path).ok()?;
        for entry in dir {
            let Ok(entry) = entry else {
                continue;
            };
            let remove_expired_entry = || {
                let entry_expired = entry
                    .metadata()
                    .ok()?
                    .modified()
                    .ok()?
                    .elapsed()
                    .ok()?
                    .gt(&three_days_duration());
                let entry_path = entry.path().as_os_str().to_owned();
                if entry_expired {
                    match std::fs::remove_file(entry.path()) {
                        Err(error) => {
                            tracing::error!(
                                target: CLEANER,
                                "Error occured while removing entry: {:?}, error is: {}",
                                entry_path,
                                error
                            );
                        }
                        _ => {
                            tracing::info!(
                                target: CLEANER,
                                "Remove entry: {:?} success",
                                entry_path,
                            );
                        }
                    };
                } else {
                    tracing::info!(
                        target: CLEANER,
                        "Checked entry: {:?} not expired",
                        entry_path,
                    );
                }
                Some(())
            };
            remove_expired_entry();
        }
        Some(())
    }
```

### Serde/SerdeJson
Serde是Rust知名的序列化/反序列化库，通过注解形式为结构体提供序列化反序列化功能。

```rust
#[derive(Serialize, Deserialize)]
struct ServiceNode {
    pub ip: String,
    pub idc: String,
    pub status: String,
}
```

需要序列化时，使用序列化库如SerdeJson进行序列化。
```rust
let service_node_str = serde_json::to_string(&service_node)?;
let service_node = serde_json::from_str(&service_node_str)?;
```

### Tokio
异步运行时，别的语言如Golang、C#等内置运行时，而Rust选择将压力给到第三方库，目前异步运行时都是由第三方库提供。

```rust
#[tokio::main]
async fn main() {
}
```

### AsyncTrait
Rust的Trait机制和异步机制结合时写起来不友好，需要使用GAT等高级知识，推荐使用AsyncTrait库，让他们结合时更易写，引入该库有一定性能开销。

```rust
#[async_trait::async_trait]
pub trait AsyncTrait {
    async fn async_trait_fn(self);
}
```

### Reqwest
Reqwest库类似Python的requests库，用于发起HTTP请求。

### TypedBuilder
Rust不支持命名参数（当然C++、Java等也不支持），一般通过建造者模式来模拟命名参数。
Rust不支持默认参数，一般通过建造者模式来模拟默认参数。

```rust
#[derive(TypedBuilder, Debug)]
pub struct Args {
    #[builder(setter(transform = |x: impl ToString| x.to_string()))]
    url: String,
    #[builder(default = "A".to_string())]
    query_type: String,
}

fn main() {
    let query_result = Args::builder()
        .url("http://www.qq.com")
        .build()
        .query();
}
```


## 如何处理Panic

### 主体思路

Panic是Rust中“不可恢复的错误”，实际其实可以做一定的处理。

通过std::panic提供的接口，我们可以捕获一些panic并进行处理。

### CatchUnwind
通过std::panic::catch_unwind函数，我们可以捕获大部分的panic，并进行处理。
```rust
let result = std::panic::catch_unwind(|| {
    println!("hello!");
});
assert!(result.is_ok());

let result = std::panic::catch_unwind(|| {
    panic!("oh no!");
});
assert!(result.is_err());
```

### JoinHandle
当你使用Tokio或者标准库提供的线程启动一个线程或协程，框架会提供一个JoinHandle对象。

对该对象进行Join操作时，框架会调用catch_unwind进行捕获，返回一个Result。

至此，一个Panic已经被转换为可以处理的Err了，不过能进行的处理也很有限，我们知道一个任务已经失败，可以记录失败信息等。

#### Tokio中的JoinHandle
```rust
use tokio::task;
use std::io;
use std::panic;

#[tokio::main]
async fn main() -> io::Result<()> {
    let join_handle: task::JoinHandle<Result<i32, io::Error>> = tokio::spawn(async {
        panic!("boom");
    });

    let err = join_handle.await.unwrap_err();
    assert!(err.is_panic());
    Ok(())
}
```
#### std::thread中的JoinHandle
```rust
use std::thread;

let builder = thread::Builder::new();

let join_handle: thread::JoinHandle<_> = builder.spawn(|| {
    // some work here
}).unwrap();
join_handle.join().expect("Couldn't join on the associated thread");
```

### PanicHook
通过注册PanicHook的方式，我们可以在Panic处理之前插入一段自定义函数，来收集Panic信息，此时一般搭配backtrace，获取调用堆栈。

获取崩溃时调用堆栈之后，我们可以记录更完善的失败信息。
```rust
std::panic::set_hook(Box::new(|panic_info| {
    let stack_trace = std::backtrace::Backtrace::capture();
    tokio::spawn(collect_infomation_of_panic(format!(
        "Panicinfo:\n{panic_info}\n\nStacktrace:\n{stack_trace}"
    )));
}));
```
