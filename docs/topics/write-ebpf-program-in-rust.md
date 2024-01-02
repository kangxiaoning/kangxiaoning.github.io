# 使用Rust编写eBPF程序

<show-structure depth="2"/>

参考[Aya项目](https://aya-rs.dev/book/)，使用Rust编写eBPF程序，最吸引人的地方在于，只需要一种语言就可以完成用户态程序和内核态程序的编写，体验比较好。

## 1. 开发环境设置

本文使用的开发环境是`Fedora Server 39`，内核版本是`6.5.6-300`，CPU架构是`aarch64`。

```Shell
[root@localhost clash]# uname -a
Linux localhost 6.5.6-300.fc39.aarch64 #1 SMP PREEMPT_DYNAMIC Fri Oct  6 19:36:57 UTC 2023 aarch64 GNU/Linux
[root@localhost clash]# more /etc/os-release 
NAME="Fedora Linux"
VERSION="39 (Server Edition)"
ID=fedora
VERSION_ID=39
VERSION_CODENAME=""
PLATFORM_ID="platform:f39"
PRETTY_NAME="Fedora Linux 39 (Server Edition)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:39"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/f39/system-administrators-guide/"
SUPPORT_URL="https://ask.fedoraproject.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=39
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=39
SUPPORT_END=2024-05-14
VARIANT="Server Edition"
VARIANT_ID=server
[root@localhost clash]# 
```
{collapsible="true" collapsed-title="Fedora Server" default-state="collapsed"}

### 1.1 OS依赖安装

```Shell
dnf install llvm llvm-devel
```

> 缺少`llvm-devel`会导致`bpf-linker`安装报如下错误，解决方法是安装`llvm-devel`后再安装`bpf-linker`。
{style="warning"}

```Shell
error: No suitable version of LLVM was found system-wide or pointed
              to by LLVM_SYS_160_PREFIX.
       
              Consider using `llvmenv` to compile an appropriate copy of LLVM, and
              refer to the llvm-sys documentation for more information.
       
              llvm-sys: https://crates.io/crates/llvm-sys
              llvmenv: https://crates.io/crates/llvmenv
   --> /root/.cargo/registry/src/index.crates.io-6f17d22bba15001f/llvm-sys-160.1.4/src/lib.rs:490:1
    |
490 | / std::compile_error!(concat!(
491 | |     "No suitable version of LLVM was found system-wide or pointed
492 | |        to by LLVM_SYS_",
493 | |     env!("CARGO_PKG_VERSION_MAJOR"),
...   |
500 | |        llvmenv: https://crates.io/crates/llvmenv"
501 | | ));
    | |__^

error: could not compile `llvm-sys` (lib) due to previous error
warning: build failed, waiting for other jobs to finish...
error: failed to compile `bpf-linker v0.9.10`, intermediate artifacts can be found at `/tmp/cargo-installTRuP1E`.
To reuse those artifacts with a future compilation, set the environment variable `CARGO_TARGET_DIR` to that path.
[root@localhost ~]#
```
{collapsible="true" collapsed-title="error: No suitable version of LLVM was found system-wide or pointed" default-state="collapsed"}

### 1.2 Rust依赖安装

```Shell
rustup install stable
rustup toolchain install nightly --component rust-src

# arm64架构
cargo install --no-default-features bpf-linker
cargo install cargo-generate
```

## 2. 创建eBPF项目

这一步使用`cargo`以及对应的模板生成项目脚手架。

```Shell
[root@localhost ~]# cd workspace/
[root@localhost workspace]# pwd
/root/workspace
[root@localhost workspace]# 
[root@localhost workspace]# cargo generate --name xdp-hello -d program_type=xdp https://github.com/aya-rs/aya-template
⚠️   Favorite `https://github.com/aya-rs/aya-template` not found in config, using it as a git repository: https://github.com/aya-rs/aya-template
🔧   program_type: "xdp" (variable provided via CLI)
🔧   Destination: /root/workspace/xdp-hello ...
🔧   project-name: xdp-hello ...
🔧   Generating template ...
🔧   Moving generated files into: `/root/workspace/xdp-hello`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /root/workspace/xdp-hello
[root@localhost workspace]# cd xdp-hello/
[root@localhost xdp-hello]# tree .
.
├── Cargo.toml
├── README.md
├── xdp-hello
│   ├── Cargo.toml
│   └── src
│       └── main.rs
├── xdp-hello-common
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── xdp-hello-ebpf
│   ├── Cargo.toml
│   ├── rust-toolchain.toml
│   └── src
│       └── main.rs
└── xtask
    ├── Cargo.toml
    └── src
        ├── build_ebpf.rs
        ├── main.rs
        └── run.rs

9 directories, 13 files
[root@localhost xdp-hello]#
```

> 也可以直接使用`cargo generate https://github.com/aya-rs/aya-template` 创建项目，根据提示进行设置。

## 3. 编译运行

这个自动生成的项目可以直接编译运行，但是默认绑定的是`eth0`网卡，如果网卡不一样需要修改下。

### 3.1 修改网卡名称

修改`xdp-hello/src/main.rs`文件中的`eth0`为自己的网卡名称即可。

### 3.2 编译

```Shell
[root@localhost xdp-hello]# cargo xtask run -- -h
warning: virtual workspace defaulting to `resolver = "1"` despite one or more workspace members being on edition 2021 which implies `resolver = "2"`
note: to keep the current resolver, specify `workspace.resolver = "1"` in the workspace root's manifest
note: to use the edition 2021 resolver, specify `workspace.resolver = "2"` in the workspace root's manifest
note: for more details see https://doc.rust-lang.org/cargo/reference/resolver.html#resolver-versions
// 省略编译细节
// 省略编译细节
   Compiling xdp-hello v0.1.0 (/root/workspace/xdp-hello/xdp-hello)
    Finished dev [unoptimized + debuginfo] target(s) in 12.37s
Usage: xdp-hello [OPTIONS]

Options:
  -i, --iface <IFACE>  [default: eth0]
  -h, --help           Print help
[root@localhost xdp-hello]#
```

### 3.3 运行

执行`RUST_LOG=info cargo xtask run`，就会看到很多日志，表示这个`xdp-hello`这个eBPF程序已经正常被加载到内核，并且被接收网络包的事件触发。

```Shell
[root@localhost xdp-hello]# RUST_LOG=info cargo xtask run
warning: virtual workspace defaulting to `resolver = "1"` despite one or more workspace members being on edition 2021 which implies `resolver = "2"`
note: to keep the current resolver, specify `workspace.resolver = "1"` in the workspace root's manifest
note: to use the edition 2021 resolver, specify `workspace.resolver = "2"` in the workspace root's manifest
note: for more details see https://doc.rust-lang.org/cargo/reference/resolver.html#resolver-versions
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/xtask run`
    Finished dev [optimized] target(s) in 0.04s
warning: virtual workspace defaulting to `resolver = "1"` despite one or more workspace members being on edition 2021 which implies `resolver = "2"`
note: to keep the current resolver, specify `workspace.resolver = "1"` in the workspace root's manifest
note: to use the edition 2021 resolver, specify `workspace.resolver = "2"` in the workspace root's manifest
note: for more details see https://doc.rust-lang.org/cargo/reference/resolver.html#resolver-versions
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
[2024-01-02T08:00:55Z INFO  xdp_hello] Waiting for Ctrl-C...
[2024-01-02T08:00:55Z INFO  xdp_hello] received a packet
[2024-01-02T08:00:55Z INFO  xdp_hello] received a packet
[2024-01-02T08:00:55Z INFO  xdp_hello] received a packet
[2024-01-02T08:00:55Z INFO  xdp_hello] received a packet
[2024-01-02T08:00:55Z INFO  xdp_hello] received a packet
```

## 4. 总结

看了[Learning eBPF](https://learning.oreilly.com/library/view/learning-ebpf/9781098135119/)这本书，了解到`aya`这个项目，之前也学习过一段时间Rust，顺手试了下，上手体检还是非常好的。

对于开发生产级别的eBPF程序，个人还是比较推荐使用**C语言和libbpf库**或者**Rust语言和aya库**这种组合。一是单一编程语言可以实现，二是有**CORE**支持，可以兼顾编程体验和“一次编译到处运行”的效果。

**BCC**框架作为比较早期的eBPF框架，为了解决一次编译到处运行的问题，需要在运行环境安装一大堆依赖包，相当于是使用的时候才编译，长期来看这种方案不适合推广。