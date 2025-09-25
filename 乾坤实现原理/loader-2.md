loader 的核心职责：拉取子应用的 HTML entry，并将其中的 DOM（尤其是脚本和样式）以“流式”方式、安全且按顺序注入到你的容器节点中，最终得到子应用暴露的运行时导出
<img width="1033" height="602" alt="image" src="https://github.com/user-attachments/assets/9ec05591-727b-47c8-b62c-4a44fbd3e224" />

## 拿到内容后
- 对文本节点，进行流式写入，然后解析html，通过parseHtml 可以拿到 script / style/ link这些标签
- 对script / style/ link标签 进行nodetransform 转换，并且把这些标签  都会放入qiankun-head 自定义标签中，对document 进行proxy代理，访问得document.head  document.body都会走代理
- 其中对script标签，重写链接（src、href）到正确的绝对路径或代理路径；
- 对脚本注入沙箱执行环境（例如通过 sandbox 包装执行）；
- 对样式或脚本进行内联/外链的转化；
- 对 defer 脚本打点，保证它们在“HTML 主体完成后”和“自身依赖就绪后”再执行。

- 外链脚本（REMOTE_ASSETS_IN_SANDBOX）：移除原始 src，保留到 dataset.src；用 fetch 拉取脚本源码；把源码用 sandbox.makeEvaluateFactory(code, src) 包一层，得到“在沙箱 globalThis 执行”的代码工厂；然后用 Blob 生成一个临时的 blob URL 设为 script.src，这样浏览器会按照正常 script 机制去执行。执行前会触发一个自定义事件（q:bse）用于回调里“撤销 blob URL、再把 src 改回原始地址，并打上 dataset.consumed 标记”，以保持行为与原生更一致。这段逻辑就在该文件的“REMOTE_ASSETS_IN_SANDBOX”分支。
- 内联脚本（INLINE_CODE_IN_SANDBOX）：把原始内联代码替换成 sandbox.makeEvaluateFactory(code) 的返回值，等于强制让脚本以沙箱方式执行，同时标记 dataset.consumed，避免重复消费。
