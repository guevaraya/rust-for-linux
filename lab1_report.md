# 学习记录

\#问题1
代码下载，由于rust-for-linux的官方代码全镜像约为4G，国内网络想下载下来是很难的，因此需要添加--depth=1 并指定分支rust-dev，但是太慢
最终还是用下面命令完成下载：

```
git clone https://github.com/fujita/linux.git -b rust-e1000

```

由于问题3，最后还是从官网下
git clone <https://github.com/Rust-for-Linux/linux> -b rust-dev --depth=1

# 问题2

根据脚本 scripts/min-tool-version.sh分析
rustup override set $(scripts/min-tool-version.sh rustc)  相当于 rustup override set  1.62.0 
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli相当于
cargo install --locked --version 0.56.0 bindgen-cli 安装

# 问题3

menuconfig搞了半天发现没有找到所谓的rust support选项，经过对比config.in发现

```
rust_is_available=n，
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
    Blocking waiting for file lock on package cache
    Updating `ustc` index

vierror: could not find `bindgen-cli` in registry `crates-io` with version `=0.56.0`

```

根因还是bindgen没有安装好，在crates-io官网搜索bindgen，确实最新的版本是0.65.1，但是fujita下载的版本太低，适应官网地址编译，解决了问题

```
cat ~/.cargo/config
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
# registry = "https://mirrors.ustc.edu.cn/crates.io-index"
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
[http]
check-revoke = false

```

配置完成结果

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1699368525435.png?raw=1)
