# 通过实现链表学Rust

[英文版](https://rust-unofficial.github.io/too-many-lists/).

# 编译

编译本书需要用到mdbook，可以从crates.io来安装mdbook

```sh
cargo install mdbook
```

假设你已经将cargo的安装目录`~/.cargo/bin`配置到环境变量中了，要编译本书可以直接用下面的命令：

```sh
mdbook build
```

如果需要在直接在本机上查看离线文档，可以运行：

```sh
mdbook serve
```

---

本书也可以使用[gitbook](https://github.com/GitbookIO/gitbook)来编译，
但是gitbook的方式不是官方支持的，所以可能会存在一些不确定的问题
