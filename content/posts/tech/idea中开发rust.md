---
draft: false
date: 2023-06-20T15:48:09+08:00
title: "idea中开发rust"
slug: "" 
tag: ["idea","rust"]
authors: ["since"]
description: "idea中开发rust"
---

## 前置条件

mac上使用brew安装rust

```
brew install rust
```

安装好之后使用命令查看rustc版本和cargo版本

```
rustc --version
```

创建一个hello.rs

```
// This is a comment
// hello.rs

// main function
fn main() {

    // Print text to the console
    println!("Hello World!");
}
```

使用rustc编译hello.rs

```
rustc hello.rs
```

然后执行编译好的二进制文件

```
./hello
```

## idea中使用rust

### 安装插件

首先需要安装1个插件Intelligent Rust,使用这个插件可以实现语法高亮，debug，内置测试模块等

在插件市场搜索rust，安装之后重启idea

![image-20230620155717113](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306201557238.png)

![image-20230620171345774](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306201713817.png)



### 项目结构

然后rust项目整体结构如下，需要注意的是src下需要有main.rs或者lib的rs，不然编译会报错

![image-20230620203301482](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306202033518.png)



我一开始的src下只有一个hello.rs，编译就报了如下错，需要在Cargo.toml里配置 [[bin]]或者[lib]才能编译运行

```
since@fengchuangdeMacBook-Pro rust-test % cargo build
error: failed to parse manifest at `/Users/since/git/rust-test/Cargo.toml`

Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section must be present
since@fengchuangdeMacBook-Pro rust-test % cargo build
error: failed to parse manifest at `/Users/since/git/rust-test/Cargo.toml`

Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section must be present

```

### 运行

运行main函数，执行debug，如果没有debug库会提示先下载

![](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306201713817.png)

![image-20230620203131997](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306202031033.png)

### 传入命令行参数

不加额外的参数, command是这样

```
run --package rust-test --bin rust-test
```

如果希望可以传入命令行参数的话，可以在command处设置arg，有一个注意点就是如果想传入命令行参数，需要加一个--，然后在--arg1或者-a这样传入参数

```
run --package rust-test --bin rust-test -- -a eat -b san
```

运行输出结果如下

![image-20230620202847407](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202306202028443.png)

## 附录

贴一下对应的文件内容

main.rs

```
use clap::{Arg, Command};

fn main() {
    println!("Hello World!");
    let matches = Command::new("rust-test程序")
        .version("0.1.0")
        .author("since")
        .about("rust-test程序")
        .arg(Arg::new("apple")
            .short('a')
            .long("apple")
            .takes_value(true)
            .help("apple"))
        .arg(Arg::new("bag")
            .short('b')
            .long("bag")
            .takes_value(true)
            .help("bag"))
        .get_matches();

    let apple = matches.value_of("apple").unwrap_or("aaa");
    let bag = matches.value_of("bag").unwrap_or("bbb");
    println!("arg1: {}, arg2: {}", apple, bag)
}
```

Cargo.toml

```
[package]
name = "rust-test"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "3.0.13", features = ["derive"] }

[build]
# 设定默认target
# target = "x86_64-unknown-linux-musl"

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
rustflags = ["-C", "link-arg=-Wl,-undefined,dynamic_lookup"]
```



## 参考链接

插件quick-start文档

https://plugins.jetbrains.com/plugin/8182-rust/docs/rust-quick-start.html

插件debug文档

https://plugins.jetbrains.com/plugin/8182-rust/docs/rust-debugging.html
