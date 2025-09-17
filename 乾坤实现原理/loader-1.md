一、loader 负责解决什么问题

- 从一个 HTML Entry（URL 或 Response）开始，流式解析 HTML，识别并改写其中的资源（script/link），在沙箱内按浏览器原生语义执行，同时尽量优化加载顺序与性能。
- 找出“入口脚本”（entry script），并在其执行成功后返回微应用暴露的全局对象（或用 sandbox 的 latestSetProp 兜底）。
- 与 qiankun sandbox、shared 包里的资源转译器、core 的 loadApp 流程配合，做到“可复用依赖”“有序执行”“沙箱隔离”。

  核心入口源码位置： `loadEntry`
