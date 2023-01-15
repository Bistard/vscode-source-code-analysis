# VSCode-source-code-analysis
My personal usage for analyzing VSCode (Visual Studio Code) source code.

自己用的，不是给人读的。

> Most reading were done before, tired of rewriting the note.
> 很多源码读的都差不多了，懒得再去记笔记了。

# Outline
* [Fundamentals](#Fundamentals)
* [Main Process](#Main%20Process)
* [Renderer Process](#Renderer%20Process)
* [Microservices](#Microservices)
* [Editor](#Editor)

## Fundamentals
* [x] Dispose Pattern（内存管理）
* [x] Dependency Injection（依赖注入）
* [x] Event Emitter（事件系统）
* IPC通信
  * [x] 主进程与渲染进程
  * [x] preload.js文件
* [ ] VSCode是如何加速"动态加载文件"这个过程的？

## Main Process
* VSCode主进程大框架
  * [x] 大框架的理解
  * [x] IPC通信

## Renderer Process
* VSCode渲染进程大框架
  * [ ] 大框架的理解
  * [x] IPC通信
  * [ ] worker的使用契机

## Microservices
* [x] 太多了，懒得列了

## Editor
* MVVM架构
  * [x] 大框架的理解
  * Model
    * [ ]
  * ViewModel
    * [ ]
  * View
    * [ ]