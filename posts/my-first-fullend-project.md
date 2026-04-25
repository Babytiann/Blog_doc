---
title: "我的第一个全栈项目"
published: 2025-02-10
description: ""
tags: ["前端"]
category: "技术"
lang: "zh-CN"
draft: false
comment: true
---
## 前言

距离上次写博客已经过去一段时间了，一直没找到合适的主题。这次借着字节跳动青训营的机会，我开始了一个新的项目——**TrackPoint**，并决定记录下整个开发过程。

## 背景

在寒假前，我报名参加了字节跳动青训营，希望能通过这个机会完成一个有实际意义的项目。青训营大项目公布后，我选择了 **TrackPoint** 作为我的项目。项目地址：[MeowTrackPoint](https://github.com/Babytiann/MeowTrackPoint)。

最初，我对“埋点 SDK”这个概念还不太清楚，以为它是某个现成的第三方库，结果发现这个 SDK 其实是需要自己开发的，并且还需要配套后端服务进行数据存储和管理，为埋点管理面板提供数据分析和可视化展示。

## 项目架构设计

在确定需求后，我对 **TrackPoint** 进行了架构设计，项目主要分为三个部分：

1. **前端 SDK（埋点库）**：负责埋点数据的采集、批量上报等核心逻辑。
2. **后端服务**：使用 `Express` 搭建，负责存储和管理埋点数据。
3. **埋点管理面板**：基于 `React + TypeScript` 开发，结合 `ECharts` 进行数据可视化。

### 1. 前端 SDK

埋点 SDK 需要解决以下问题：

- **用户行为采集**：监听用户的点击、页面访问、输入等行为。
- **性能监控**：监控用户从在开始访问这个网页到真正看到网页内容时的时间
- **数据上报**：支持 **批量上报** 机制，提高效率并减少服务器压力。
- **错误监控**：捕获JS 异常，包括 async 函数抛出的错误、被拒绝的 Promise 以及资源加载失败的错误，并将这些错误记录在埋点数据中

### 2. 后端服务

后端采用 `Express` + `MySQL`，主要功能包括：

- 接收埋点数据并存储到数据库。

- 提供 API，供前端管理面板获取数据。

  ```
  import express from 'express';
  import baseInfo from "./API/baseInfo";
  import demo from "./API/demo";
  import timing from "./API/timing";
  import error from './API/error'
  
  const app = express();
  
  app.use(express.json());  // 解析 JSON 请求体
  app.use(express.urlencoded({ extended: true }));  // 解析 URL-encoded 请求体
  
  //Router引入
  app.use('/baseInfo', baseInfo);
  app.use('/demo', demo);
  app.use("/timing", timing);
  app.use('/error', error);
  app.use('/check', check);
  
  app.get('/', (req, res) => {
      res.send('Welcome to Meow Backend !✨');
  });
  
  
  //监听端口
  const port = 5927
  export const server = app.listen(port, () => {
      console.log(`Server is running at http://localhost:${port}`);
  });
  ```

### 3. 数据管理面板

管理面板使用 `React` + `TypeScript`，主要功能包括：

- 以表格形式展示埋点数据。
- 通过 `ECharts` 进行数据可视化，如趋势图、用户行为分析等。
- 过滤和查询特定时间范围内的数据。

同时还用了**GSAP**加入了一些动画看着会稍微QQ弹弹一点
![面板截图](https://cdn.gemdzqq.com/img/TrackPoint.png)

## 总结

整个项目从零开始搭建，涉及到了 **前端、后端、数据库、数据可视化** 等多个方面，虽然一开始压力不小，但最终还是顺利完成了核心功能。

后续计划：

- **优化 SDK**，支持更多自定义事件。
- **增加更多分析维度**，如用户路径分析、停留时间统计等。
- 简化后端部分冗余代码，当时写的时候有点傻哈哈哈