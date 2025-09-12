# qiankun-source-study


一、先建立整体心智模型（5 分钟）

- 关键角色
  - 主应用：负责路由/菜单、选择并承载子应用容器（我们 examples/main 就是）。
  - 加载器：解析子应用入口 HTML/JS、按需下载并执行脚本（packages/loader）。
  - 沙箱：用 Proxy/with 等机制隔离全局变量、事件、定时器等副作用（packages/sandbox）。
  - 调度器：编排加载、解析生命周期、挂载/卸载流程（packages/qiankun）。
- 生命周期总览
  - 子应用导出/注入 bootstrap、mount、unmount（react15/16 的 Demo 就是最直观的例子）。
  - 主应用通过 loadMicroApp 启动一次完整的“加载→解析→沙箱→挂载→卸载”链路。
二、用“日志+示例”走通一遍链路（15–20 分钟）

- 打开浏览器 DevTools，勾选 Preserve log。
- 在主应用页面 http://localhost:7099/ ：
  - 切换左侧菜单 “react16”“react15”，观察控制台里我们注入的开发日志，关键前缀形如 [qiankun][dev]。
  - 关注这些关键节点的顺序：
    1. 1.
       loadMicroApp 初始化与等待前序实例卸载
    2. 2.
       loader 加载入口：loadEntry start → 入口 HTML/脚本流处理 → 入口脚本完成
    3. 3.
       loadApp：创建沙箱 → 解析 lifecycles → 用户 mount 前后 → 用户 unmount 前后 → 容器清理
    4. 4.
       mountRootParcel 开始/创建，以及最后的清理
- 在两个子应用页面分别打开 DevTools：
  - react16: http://localhost:7100/
  - react15: http://localhost:7102/
  - 观察它们自身的运行日志和资源加载情况（network/source-map），体会“主应用跨域加载子应用资源”的方式。
三、对照源码逐层深入（60–90 分钟）
按下面顺序阅读，每一层都结合我们注入的日志定位对应代码位置。

1. 1.
   从“使用入口”看起
- 主应用如何触发加载
  - `index.js`
  - 关注：点击菜单时调用 loadMicroApp，传入 name/entry/container，以及 sandbox 配置。
- 子应用如何暴露生命周期
  - `index.js`
  - `index.js`
  - 看它们如何导出/注册 bootstrap/mount/unmount（react15 用 webpack4，react16 用 CRA + rescripts 注入插件）。
2. 1.
   编排与生命周期：qiankun 主体
- `loadMicroApp.ts`
  - 看“记忆化”与“前序实例卸载”的等待逻辑、mountRootParcel 的创建时机，以及我们注入的日志顺序。
- `loadApp.ts`
  - 核心：沙箱创建、入口执行后解析 lifecycles、用户 mount/unmount 前后钩子、容器清理。
  - 建议在这些点打断点：创建沙箱处、解析 lifecycles 处、调用用户 mount/unmount 前后。
3. 1.
   入口加载与脚本执行：loader
- `index.ts`
  - 关注 loadEntry：HTML 流式解析、收集并按顺序加载脚本、如何与沙箱协作“在正确的全局环境里执行”。
  - 我们的日志会标出：loadEntry start → 入口解析/脚本检测 → 脚本执行完成 → HTML 流结束。
4. 1.
   沙箱与副作用隔离：sandbox
- `src`
  - 理解 Proxy window/with 作用域、document 代理、事件监听/定时器收集与回收。
  - 看 mount/unmount 与沙箱 enable/disable 的关系，以及 Qiankun 在 loadApp.ts 中何时启用/关闭沙箱。
5. 1.
   构建期注入：webpack-plugin
- `src`
  - 明白插件如何在构建时帮助注入或调整运行所需的标记（示例里 webpack4/CRA 均通过它工作）。
四、建议边看边做的小实验（20–30 分钟）

- 实验 1：关闭沙箱对比行为
  - 在主应用 `index.js` 的 loadMicroApp 第二个参数把 { sandbox: true } 改为 false，对比日志与全局污染。
- 实验 2：重复挂载/卸载
  - 快速切换 react16 ⇆ react15，观察“等待前序实例卸载”的日志，以及内存/事件是否被回收。
- 实验 3：入口异常与容错
  - 暂时改错一个子应用入口地址，观察 loader 层的错误日志和 qiankun 的错误传播。
五、你可以重点关注的“考点”

- 生命周期收敛：如何从“入口脚本任意导出”变成统一的 bootstrap/mount/unmount 调用。
- 顺序保证：HTML → 脚本的顺序执行、async/defer 的处理、打断/错误时如何恢复。
- 隔离与回收：沙箱隔离维度（window、事件、计时器、样式等）以及 unmount 清理策略。
- 并发与串行：同名应用重复挂载的时序处理、前序实例卸载等待。
- 日志定位：用我们加的 [qiankun][dev] 日志快速对照“行为→源码→原因”。
六、遇到问题怎么调

- 浏览器端：打开主应用与子应用的 DevTools，按前缀过滤日志；在源码对应位置打断点。
- Node 端：看启动终端输出（我们已经把 main/react16/react15 都跑起来了），如果需要我可以再帮你在关键函数里加更细的日志或断点。
