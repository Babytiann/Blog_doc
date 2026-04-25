---
title: "服务端渲染 SSR 入门：EventSource 与 Next.js / Nuxt.js 实践"
published: 2025-05-14
description: "从 SEO、首屏性能和实时推送场景出发，简单介绍 SSR、SSE/EventSource，以及 Next.js 与 Nuxt.js 的常见实践。"
tags: ["SSR", "EventSource", "Next.js", "Nuxt.js", "前端"]
category: "技术"
lang: "zh-CN"
draft: false
comment: true
---

最近有朋友问我：“现在做前端还搞 SSR 吗？不是都客户端渲染（CSR）就行了吗？”

我笑着回答：“你以为我愿意吗？是 SEO 要求的。”

今天简单分享一下我最近踩 SSR 的一些坑，顺便聊聊 `EventSource` 这个冷门但有趣的 API，还有 Next.js / Nuxt.js 这些现代框架如何帮我们优雅地搞定 SSR。

## SSR 是什么？为啥还要用？

**SSR（Server Side Rendering）**，顾名思义，就是在服务器把 HTML 页面“渲染好”，再返回给浏览器，而不是像 CSR 那样等 JS 加载完再让浏览器自己渲染。

如下所示：

| 模式 | CSR（客户端渲染） | SSR（服务端渲染） |
| --- | --- | --- |
| 页面内容 | 初始空壳，JS 拉数据后渲染 | 初始就有完整内容 |
| SEO 友好 | 不友好，搜索引擎可能抓不到内容 | 友好，服务端已生成 |
| 首屏速度 | 相对慢 | 相对快，但服务器压力更大 |

## EventSource 是啥？它和 SSR 有啥关系？

`EventSource` 是 HTML5 提供的一种 **服务端推送（Server-Sent Events, SSE）** 技术。光看字面意思就是“事件来源”。

你可以简单理解成：服务端主动往前端发消息，而不是前端不停轮询。

```js
const es = new EventSource("/api/events");

es.onmessage = (event) => {
	console.log("推送消息:", event.data);
};
```

它的作用和 WebSocket 类似，但更轻量、基于 HTTP 协议。

在 SSR 场景中，SSE 可以作为 SSR 后的补充手段，用于实时推送数据，实现日志回显、构建进度或 AI 对话等流式输出。

在我之前做的 MeowTalk，一个 LLM 对话平台中，就是用了该 API 做 AI 对话的实时推送。以下是代码片段：

```ts
try {
	const eventSource = new EventSource(
		`http://localhost:5927/askai?message=${encodeURIComponent(value)}&sessionId=${sessionId}`,
		{ withCredentials: true },
	);

	eventSource.onmessage = (event) => {
		onAiMessage?.(event.data);
	};

	eventSource.onerror = () => {
		eventSource.close();
		setLoading(false);
		message.error("连接异常，已断开");
	};

	eventSource.addEventListener("end", () => {
		eventSource.close();
		setLoading(false);
	});
} catch (error) {
	console.error("请求出错:", error);
}
```

后端大致是这样：

```ts
const fetchMeowTalk = async (
	message: string,
	res?: Response,
): Promise<string> => {
	return new Promise((resolve, reject) => {
		try {
			const response = axios.post(
				"https://spark-api-open.xf-yun.com/v1/chat/completions/",
				{
					max_tokens: 4096,
					top_k: 4,
					temperature: 0.5,
					messages: [
						{
							role: "system",
							content: "你是一个无所不知的人",
						},
						{
							role: "user",
							content: message,
						},
					],
					model: "4.0Ultra",
					stream: true,
				},
				{
					headers: {
						"Content-Type": "application/json",
						authorization: `Bearer ${process.env.SERVER_CLIENT_ID}`,
					},
					responseType: "stream",
				},
			);

			res?.setHeader("Content-Type", "text/event-stream");
			res?.setHeader("Cache-Control", "no-cache");
			res?.setHeader("Connection", "keep-alive");

			let fullResponse = "";
			let isFirstChunk = true;

			response
				.then((apiResponse) => {
					apiResponse.data.on("data", (chunk: Buffer) => {
						const dataStr = chunk.toString().trim();

						if (!dataStr.startsWith("data: [DONE]")) {
							try {
								const jsonData = JSON.parse(dataStr.replace(/^data:/, "").trim());
								const textChunk = jsonData?.choices?.[0]?.delta?.content;

								fullResponse += textChunk;

								const parsedText = isFirstChunk ? textChunk : fullResponse;
								isFirstChunk = false;

								res?.write(`data: ${parsedText}\n\n`);
							} catch (err) {
								console.error("JSON 解析失败:", err);
							}
						}
					});

					apiResponse.data.on("end", () => {
						res?.end();
						console.log("完整响应内容：", fullResponse);
						resolve(fullResponse);
					});

					apiResponse.data.on("error", (err: any) => {
						console.error("流式请求出错:", err);
						res?.status(500).end();
					});
				})
				.catch((error) => {
					console.error("Error fetching MeowTalk API:", error);
					res?.status(500).json({ error: "Failed to fetch AI response" });
					reject("Ai 请求出错");
				});
		} catch (error) {
			console.error("Error in fetchMeowTalk:", error);
			res?.status(500).json({ error: "Failed to fetch AI response" });
			reject("Ai 请求出错");
		}
	});
};
```

## Next.js / Nuxt.js 是如何做 SSR 的？

### Next.js（React 生态）

Next.js 是 Vercel 出品的 React SSR 框架，它主打“**一站式全栈 React 应用**”。

它的 SSR 主要通过两种方式：

1. `getServerSideProps`：每次请求时都会在服务端执行，适合动态内容。

```ts
export async function getServerSideProps() {
	const res = await fetch("https://api.example.com/data");
	const data = await res.json();

	return { props: { data } };
}
```

2. `getStaticProps` + `getStaticPaths`：适合生成静态页面（SSG），打包时一次性生成，提高性能。

你甚至可以把 `EventSource` 接入 API 路由（`/api/events`），实现“服务端推送构建进度”。

### Nuxt.js（Vue 生态）

Nuxt.js 就是 Vue 的 SSR 框架，和 Next.js 的理念非常像。

它的 SSR 路由系统基于文件结构，你只需要放好页面和组件，Nuxt 会帮你自动配置服务端渲染。

它也有几种“数据预取”方式：

* `asyncData`：页面级钩子，在 SSR 时运行，用来提前加载数据。
* `fetch`：在组件中运行，也支持 SSR。

```js
export default {
	async asyncData({ params }) {
		const data = await fetchData(params.id);
		return { data };
	},
};
```

此外，你可以用 Nuxt 插件方式监听服务端事件或使用中间件，实现和 `EventSource` 类似的功能。

## 小结

虽然现在前端框架普遍以客户端渲染（CSR）为主，但服务端渲染（SSR）在某些场景下依然不可替代，甚至越来越重要。在以下场景下它依然有用：

* 需要 SEO 的场景，比如博客、电商
* 首屏加载性能敏感的项目
* 面向爬虫、社交分享卡片的内容

而在 SSR 渲染之后，有些数据仍然是动态变化的、实时性的，比如日志回显、AI 对话响应、构建进度、股票价格、数据大屏更新等。此时，`EventSource`（SSE）就是一种非常合适的解决方案：

* 不需要复杂的 WebSocket 握手和状态管理
* 建立简单，只用一个 HTTP 请求
* 推送稳定可靠，兼容主流浏览器
* 非常适合“从服务端单向推送数据到客户端”的需求
