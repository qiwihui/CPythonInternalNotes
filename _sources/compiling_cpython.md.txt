# 编译 CPython

现在你已经下载了 CPython 开发环境并对其进行了配置，你可以将 CPython 源代码编译成一个可执行的解释器。

与 Python 文件不同，C 源代码每次更改时都必须重新编译。

在前一章中，我们已经设置开发环境，并设置了运行“Build”阶段的选项，该选项将重新编译 CPython。
在构建步骤工作之前，你需要一个 C 编译器和一些构建工具。
使用的工具取决于你使用的操作系统。

> 如果你担心这些步骤中的任何一个会干扰您现有的 CPython 安装，请不要担心。CPython 源目录的行为就像一个虚拟环境。
>
> 对于编译 CPython、修改源代码和标准库，这些都保留在源目录的沙箱中。
>
> 如果要安装自定义版本，本章也将介绍此步骤。

## 在 macOS 系统上编译 CPython

在 macOS 上编译 CPython 需要一些额外的应用程序和库。你首先需要基本的 C 编译器工具包。
“Command Line Development Tools” 是一个可以在 macOS 中通过 App Store 更新的应用程序。
你需要在终端上执行初始安装。

在终端中，通过运行以下命令安装 C 编译器和工具包：

```bash
$ xcode-select --install
```

该命令会弹出一个提示，提示下载并安装一组工具，包括 Git、Make 和 GNU C 编译器。

你还需要一份 [OpenSSL](https://www.openssl.org/) 的工作副本，用于从 PyPi.org 网站获取包。
如果你以后计划使用此构建版本来安装其他软件包，则需要进行 SSL 验证。

在 macOS 上安装 OpenSSL 的最简单方法是使用 [Homebrew](https://brew.sh/)。

可以使用一下命令安装 Homebrew：

```bash
$ /usr/bin/ruby -e "$(curl -fsSL \
 https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

一旦安装完成，你就可以使用 `brew install` 命令来安装所需的工具。

```bash
$ brew install openssl xz zlib gdbm sqlite
```

现在你已经安装了依赖项，你可以运行 `configure` 脚本。
Homebrew 有一个命令 `brew --prefix [package]` ，它将给出安装包的目录。
你将通过编译 Homebrew 使用的位置来启用对 SSL 的支持。

标志 `--with-pydebug` 启用调试挂钩。如果你打算出于开发或测试目的进行调试，请添加此项。

配置阶段只需要运行一次，同时指定 `zlib` 包的位置：

```bash
$ CPPFLAGS="-I$(brew --prefix zlib)/include" \
 LDFLAGS="-L$(brew --prefix zlib)/lib -L$(brew --prefix bzip2)/lib" \
 ./configure --with-openssl=$(brew --prefix openssl) --with-pydebug
```

运行 `configure` 将在存储库的根目录中生成一个 `Makefile`，你可以使用它来自动化构建过程。

你现在可以通过运行一下命令来构建 CPython 二进制文件：

```bash
$ make -j2 -s
```

在构建过程中，你可能会收到一些错误。在构建摘要中，`make` 会通知你并非所有包都已构建。
例如，`ossaudiodev`、`spwd` 和 `_tkinter` 将无法使用这组指令进行构建。
如果你不打算针对这些软件包进行开发，那也没关系。
如果是，请查看[官方开发指南](https://devguide.python.org/)网站以获取更多信息。

构建将需要几分钟并生成一个名为 `python.exe` 的二进制文件。每次对源代码进行更改时，
你都会需要使用相同的标志重新运行 `make`。
`python.exe` 二进制文件是 CPython 的调试二进制文件。
执行 `python.exe` 以查看有效的 REPL：

```bash
$ ./python.exe
Python 3.9.0b1 (tags/v3.9.0b1:97fe9cf, May 19 2020, 10:00:00)
[Clang 10.0.1 (clang-1001.0.46.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

> 是的，没错，macOS 版本的文件扩展名为 `.exe`。 这个扩展*不是*因为它是 Windows 二进制文件！
> 因为 macOS 有一个不区分大小写的文件系统，并且在使用二进制文件时，
> 开发人员不希望人们不小心引用 `Python/` 目录，因此附加了 `.exe` 以避免歧义。
> 如果你稍后运行 `make install` 或 `make altinstall`，
> 它将在将文件安装到你的系统之前将文件重命名回 python。
