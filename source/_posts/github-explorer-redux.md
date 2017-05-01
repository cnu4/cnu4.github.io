---
title: "使用 redux 改写的 github-explorer"
date: 2016-08-22 20:02:21
tags:
	- react
	- redux
---

## 介绍

刚学习了 redux 不久，恰好看到一个优秀的 react 项目 [github-explorer](https://github.com/trungdq88/github-explorer)，该应用使用了 RxJS 去处理数据流，为了巩固学习便有了使用 redux 改写的想法。

<!-- more -->

[源码地址](https://github.com/cnu4/github-explorer-redux)
[DEMO地址](https://cnu4.github.io/github-explorer-redux)

## 自定义中间件

应用中使用了自定义个中间件 **api**，方便编写异步的 action creators。异步 action 可以定义成以下方式

``` js
export function loadUserProfileRepos (username) {
  return {
    types: [USER_PROFILE_REPOS_REQUEST, USER_PROFILE_REPOS_RECEIVED,
      USER_PROFILE_REPOS_FAILURE],
    callAPI: () => api('fechURL')
  }
}
```

中间件接收到这种形式的action，会处理异步请求并在适当的时候 dispatch `types` 中的各项。

## 其他
 
应用的拉取数据的进度条方面，负责拉取状态 reducer 在接收到诸如 `xx_REQUEST` 和 `xx_RECEIVED` 的 actions 后，会更新表示进度条状态的数据。

因为只是巩固redux学习，所以原应用的部分动画效果没有加上。

除了数据流部分，应用大部分都是参考了原应用。


## 依赖

使用了redux后加上的依赖

 - redux
 - react-router-redux

## Development

开发

```
npm install
npm run start
```

打包

```
npm run dist
```

# 参考

 - [github-explorer](https://github.com/trungdq88/github-explorer)
 - [redux 文档](http://cn.redux.js.org/index.html)