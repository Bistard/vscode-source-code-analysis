# Introduction to Directory Structure

> 首先这是Visual Studio Code的[仓库链接](https://github.com/microsoft/vscode)。

## Important Root Directory
我们先简单看一下repo下都有哪些重要文件夹：

* `src`: 这里就是存整个软件的所有核心源代码，代码量巨大。同时也包含了基本所有的测试代码（文件后缀为`*.test.ts`）。
* `resources`: 这里存了一些二进制文件，比如图片，字体等等。
* `extensions`: 这里存了所有VSCode的默认插件。因为VSCode有一套自己完整的开发插件系统的流程，所以这些默认插件和市场上的第三方插件本质上没区别，都是通过同一套插件系统进行开发的，所以对于每一个默认插件里面都有相关的`.json`配置文件。
* `build`: 这里存了关于如何构建（build）整个软件的脚本和配置文件。
* `scripts`: 存了各种各样的脚本文件。大部分都是`sh`和`bat`文件。
* `test`: 存了各种测试代码相关的test runners（也可以算一种脚本）。比如在VSCode中，测试代码被分成了`unit`，`integration`和`smoke`。

## Dive Into `src`
// TODO
