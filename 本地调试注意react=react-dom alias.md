我还是不明白 为什么要加上 ‘react: path.join(__dirname, 'node_modules/react'),

'react-dom': path.join(__dirname, 'node_modules/react-dom'),’


Builder

参考了 1 个上下文

webpack.config.js 49-50

这是为了解决“页面里出现了两个 React 实例”导致的 hooks 崩溃问题，保证 React/ReactDOM 在整条依赖链上都是同一份拷贝。

简单结论

- 不加这两个 alias 时，你的示例工程、@qiankunjs/react、本地 ESM 构建等，可能各自通过不同的解析起点去“各自”找到 react/react-dom，结果同一页面打进了两份 React（哪怕版本相同），React 的 hooks 运行时上下文就会错乱，出现 useState 读取 null 的错误。
- 加上这两个 alias 后，强制让所有代码（主应用、@qiankunjs/react、本地 qiankun ESM 等）都从 examples/main/node_modules 下取到“同一份” react 和 react-dom，React 的单例假设成立，hooks 才能正常工作。
为什么会出现多个 React

- Monorepo + 本地联调：我们把 @qiankunjs/react、@qiankunjs/ui-shared、qiankun 都 alias 到了 packages 下的 dist/esm。Webpack 解析它们内部的 import 'react' 时，会从这些文件所处目录出发去解析依赖（packages/.../dist/esm），在 pnpm 的软链结构下，很容易找到和主应用不同路径的 React 拷贝（符号链接目标不同也会被当作不同模块）。
- React 的运行机制要求“单例”：React DOM 在渲染时会把 hooks 的 dispatcher 安装到它所用的 React 实例上（模块对象）。如果你的组件代码是从另一份 React 实例里 import useState，那么它看到的 dispatcher 是空的，于是就出现 Cannot read properties of null (reading 'useState') 这样的报错。
- 这不仅限于 React 本体，react-dom 也必须和同一份 React 配对使用，否则也会发生上下文不一致的问题。
alias 的作用

- 在 examples/main/webpack.config.js 里加这两条：
  - react → path.join(__dirname, 'node_modules/react')
  - react-dom → path.join(__dirname, 'node_modules/react-dom')
- 含义是：无论谁（主应用、本地 ESM、三方包）在 import 'react' 或 'react-dom'，都一律指向主应用这两个明确的路径。这样整个页面里只有一个 React/ReactDOM 实例，hooks 的 dispatcher 和渲染器配对一致，错误自然消失。
为什么一开始不显性加 alias 也“看起来可用”

- 如果只跑 Vanilla 路径（主入口直接 import 本地 qiankun ESM），且依赖解析碰巧都落在同一路径，可能不会暴露问题；
- 一旦引入 @qiankunjs/react 这条链路，解析起点不同 + pnpm 的目录结构，有较高概率打出重复的 React；
- workspace:* 并不等价于“bundler 一定用你想要的那份构建”，最终是 bundler 的模块解析结果说了算，alias 是最可靠的强制手段。
如何自检是否有“多 React”

- 现象层面：出现 hooks 报错（useState 读 null）、或热更新、Context/memo 行为异常。
- 构建层面：查看打包产物是否出现两段 React 运行时代码；或在构建日志/分析里看到 react 被打进两次。
- 运行层面：在不同位置打印 React 对象引用对比（对象是否严格相等），但这需要在库里也插入日志，操作略麻烦，所以推荐用 alias 从源头杜绝。
最佳实践小结

- 在 monorepo 做本地联调（特别是把本地包 alias 到 dist/esm）时，务必把 react 和 react-dom 也 alias 到“主应用自己的 node_modules”；
- 对于会间接调用 qiankun 的 UI 绑定包（@qiankunjs/react 等），同样建议在示例工程里 alias 到本地 dist/esm，且配套做 React 单例 alias；
- 改动源码后先构建本地包、再重启 dev server，避免吃到旧产物。
一句话记忆

- 你一旦“本地直连”了库的 ESM 源/构建，就请“强制单例化 React/ReactDOM”，否则很容易因为多 React 导致 hooks 报错。
