+++
title = 'Vue基础-项目的初步搭建和项目结构'
date = 2024-04-22T15:02:33+08:00
draft = false
categories = [
    "前端",
]

tags = [
    "vue",
]
image = "/cover/cover2.jpg"
+++

## vue项目的构建和运行

可以使用以下两种不同的构建方式来构建vue项目

```
npm init vue@latest
```
```
npm init vite@latest
``` 

## 两种构建方式的区别

- npm init vue@latest

    这个命令专门用于快速启动新的 Vue.js 项目。它背后使用的是 Vue 官方推荐的 `create-vue` 工具包，旨在为开发者提供一个简单直接的方式来创建 Vue 项目，包括 Vue 3.x 的最新功能和最佳实践。当你运行 `npm init vue@latest` 时，它将会安装并运行`create-vue`,并引导你通过一个交互式的界面选择不同的配置选项，如：

    - 是否使用 TypeScript
    - 是否添加 Vue Router（用于页面路由管理）
    - 是否添加 Pinia（状态管理，Vue 3 的推荐状态管理库，替代 Vuex）
    - 是否需要 CSS 预处理器（如 SASS）


- npm init vite@latest

    `npm init vite@latest` 命令则是初始化使用 Vite 作为构建工具的项目。Vite 是一个现代化的前端构建工具，它提供了极快的冷启动、快速的热更新和丰富的插件生态系统。Vite 支持多种前端框架，如 Vue、React、Svelte 等，使得它在前端开发中非常通用。

    当你执行 `npm init vite@latest` 命令时，它也会提供一个交互式的界面，让你选择以下配置：

    - 项目名称
    - 想要使用的框架（Vue、React、Svelte 等）
    - 是否使用 TypeScript
    - 是否需要 CSS 预处理器

    **这种方法更加通用，不仅限于 Vue 项目**，适合那些希望利用 Vite 强大功能，同时需要支持多种框架的开发者。

**`npm init vue@latest`** 主要针对那些想要快速开始一个新的 Vue 项目，并利用 Vue 3 和相关生态的最佳实践的开发者,这种构建方式实质上还是使用了vite脚手架，并且会默认配置好`@`的映射地址。

**`npm init vite@latest`** 提供了一个更为通用的解决方案，适用于需要使用 Vite 作为构建工具的各种前端项目，无论是 Vue、React 还是其他框架。

## vue项目结构

构建完项目后会出现以下文件夹

```
node_moudles   --- 依赖文件夹
public         --- 资源文件夹（浏览器图标）
src            --- 源代码文件夹           
index.html     --- 入口html文件
package.json   --- 信息描述文件（保留依赖描述和项目构建命令等）
README.md      --- README.md文件
vite.config.js --- vue的配置文件（对于路径中的@配置写在这里）
```

src文件夹内通常开发者会分成以下内容
```
src/
|-- assets/             # 静态资源如样式、图片等
|-- components/         # Vue 组件（默认存在）
|-- views/              # Vue 页面组件，通常是路由的组件、此文件夹也常命名成page
|-- router/             # 路由配置文件
|-- store/              # Vuex、pinia 状态管理
|-- services/           # 应用级的服务，如 API 客户端
|-- utils/              # 工具类方法
|-- App.vue             # 根组件
|-- main.js             # 入口文件，初始化 Vue 实例、插件等
```

> 不同的开发者会有不同的习惯，上面只是做为一个简单的参照