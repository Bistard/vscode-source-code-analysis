---
layout: post
title:  "Take a Glance at Electron"
categories: [VSCode, Tools-and-Technologies]
---


# What is Electron

Electron is an open-source framework that allows developers to create cross-platform desktop applications using web technologies like JavaScript, HTML, and CSS. 
* It achieves this by leveraging the **Node.js** runtime for back-end functionality and **Chromium** for front-end display.
* Chromium is what renders the screens/windows of your app, while Node.js provides OS-level functionality (access to files or databases) for your app.
* It allows developers to create a single application that runs almost seamlessly across different operating systems - Windows, macOS, or Linux.
> Electron is a product of GitHub, which is owned by Microsoft.

# How does Electron support multiple platforms - Chromium
Chromium belongs to Google and is also the codebase of lots of modern browsers considering it is free, powerful, and open-sourced.
* It is also the codebase of Chrome which is also a proprietary product of Google.

Since Electron uses Chromium, we then can have cross-platform support: every operating system will use the same bundled version of Chromium. It roughly guarantees 99% of our code (JavaScript, HTML, CSS) requires no changes when running on different platforms.
* We should thank the Chrome Team, for all the hard work done by them :)
* This feature is one benefit Electron has over [Tauri](https://tauri.app/?ref=debugandrelease.com) (another cross-platform framework) - your app will look and behave exactly the same as any supported operating system.

## Processes
Chromium has two types of processes: the **main process** and the **renderer process**. There can only be one main process and Chromium starts from it, then the renderer process may spawn from it.
* Renderer processes are synonymous with a browser window. 
* The main process holds references to all the renderer processes and can create/delete them when necessary.
* Since they are different processes, they cannot access each other’s data directly. The communication between two types of processes is done by an Inter-Process Communication (IPC) system, which is provided by the Electron framework. This section will be discussed in later blogs.

# Node.js
Node.js is not a programming language, it is an environment that runs on a JavaScript engine (V8 in VSCode's case, which is used in Chromium) and executes JavaScript code outside a browser environment. 

It gives us the ability to read and write files outside of the browser since the browser normally cannot achieve IO tasks without asking permission from the users. 
* Otherwise, imagine that you are browsing the internet, and suddenly one of your secret pictures in your local disk got hacked, just because you accidentally clicked some hackable links.

> That also should raise our attention on security issues about Node.js. Electron official website also has a long list of potential security issues that might happen when processes in Chromium encounter Node.js. I will discuss the security concern in the next blog, which is a big section.

Recall our two types of processes, the main process and the renderer process:
* The main process always has direct access to Node.js, which means it can create/delete/modify files in your operating system just like other applications do.
* The renderer process does not always have access to Node.js which depends on your configurations. However, it always has access to web APIs, which can manipulate the DOM tree.

# Challenges
However, it's important to note that while Electron offers numerous benefits, it's not without its criticisms. Concerns have been raised regarding the performance of Electron-based applications, particularly in terms of memory usage and size.
* The main reason for this situation is due to the Chromium kernel. The browser kernel can be said to be one of the most complex types of software in the world at present, and the same goes for Chromium. 
* Embedding a complete browser kernel will appear to be quite bloated. If one wants to optimize it, because its various enormous systems are all tightly coupled together, most programmers simply can't carry out any form of decoupling, making it very difficult to discard systems that are not needed for Electron. This is a challenge that the Electron team needs to face.

# Conclusion
However, given the trade-off of extensive cross-platform compatibility and reduced development times, many developers consider these compromises acceptable. 

Moreover, with an active community of developers and regular updates, Electron continues to evolve, constantly addressing these concerns and optimizing its performance.

Here is a list of applications that are built by Electron:

# Reference
* https://www.debugandrelease.com/the-ultimate-electron-guide/
* https://www.electronjs.org/docs/latest/tutorial/security
* https://www.chromium.org/chromium-projects/




