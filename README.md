# 该仓库目标
用于调试react 源码
## 好处
1 解决没有sourcemap，无法定位到源文件的问题
2 解决 丢失了注释的 问题
3 解决 每次修改文件后需要手动build 的问题
## 坏处
配置复杂
# 配置方式
## 1 创建项目
```
yarn create react-app react-source-debug
```
## 2 clone react 源码
```
cd react-source-debug
cd src
git clone git@github.com:facebook/react.git --depth 1 --branch v18.2.0
cd ..
git add .
gcmsg 'feat(): download react code'
```
## 3 eject webpack config（为了修改配置）
```
yarn eject
```
## 4 修改webpack 配置
1. 文件 `react-source-debug/confi
g/webpack.config.js`
中 alias 添加如下路径映射：
```
      alias: {
        react: path.resolve(__dirname, '../src/react/packages/react'),
        'react-dom': path.resolve(__dirname, '../src/react/packages/react-dom'),
        'react-reconciler': path.resolve(
          __dirname,
          '../src/react/packages/react-reconciler'
        ),
        'react-dom-binding': path.resolve(
          __dirname,
          '../src/react/packages/react-dom-binding'
        ),
        scheduler: path.resolve(__dirname, '../src/react/packages/scheduler'), // 调度
        shared: path.resolve(__dirname, '../src/react/packages/shared'), // 一些通用工具包
      },
```
2. 文件 `react-source-debug/confi
g/env.js`
增加环境变量：
```
  const stringified = {
    'process.env': Object.keys(raw).reduce((env, key) => {
      env[key] = JSON.stringify(raw[key]);
      return env;
    }, {}),
    // 增加如下变量（默认rollup会添加，但此处webpack需手动添加）
    __DEV__: true,
    __PROFILE__: true,
    __UMD__: true,
    __EXPERIMENTAL__: true
  };
```

## 5 解决报错
1. ERROR in ./src/index.js 10:13-32
export 'default' (imported as 'ReactDOM') was not found in 'react-dom/client' (possible exports: createRoot, hydrateRoot)
解决： 如下文件`react-source-debug/src/index.js'`导入方式：
```
import React from 'react';
import ReactDOM from 'react-dom/client';
```
改为 import * :
```
import * as React from 'react';
import * as ReactDOM from 'react-dom/client';
```
2. ERROR in ./src/react/packages/react-reconciler/src/Scheduler.js 30:35-64
export 'unstable_yieldValue' (imported as 'Scheduler') was not found in 'scheduler'
解决：从文件`./src/react/packages/react-reconciler/src/Scheduler.js`注释可知，
```
// this doesn't actually exist on the scheduler, but it *does*
// on scheduler/unstable_mock, which we'll need for internal testing

```
确实没有，内部测试时，需自己mock，方式是将下面代码
```
// this doesn't actually exist on the scheduler, but it *does*
// on scheduler/unstable_mock, which we'll need for internal testing
export const unstable_yieldValue = Scheduler.unstable_yieldValue;
export const unstable_setDisableYieldValue =
  Scheduler.unstable_setDisableYieldValue;
```
改为如下：
```
// this doesn't actually exist on the scheduler, but it *does*
// on scheduler/unstable_mock, which we'll need for internal testing
// export const unstable_yieldValue = () => {};
// export const unstable_setDisableYieldValue =() => {};
```
3. ERROR in [eslint] Failed to load config "fbjs" to extend from
该问题报eslint 问题，我们不需要
解决：
在项目根目录创建 .env 文件，内部加入如下：
```
DISABLE_ESLINT_PLUGIN=true
```
## 6 其他错误
修改完以上问题之后，运行 `yarn start`，控制台不再报错
但是启动之后白屏，打开浏览器检查，发现如下错误，仍要解决：
1. React is not defined
    at ./src/react/packages/shared/ReactSharedInternals.js
提示是说变量不存在，确实是不存在（因为实际中该文件是由rollup帮我们处理，此处需手动引入），解决，
将如下代码：
```
import * as React from 'react';

const ReactSharedInternals =
   React.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED;
```
改为：
```
// 手动引入此处变量
import ReactSharedInternals from "../react/src/ReactSharedInternals";
```
2. Error: This module must be shimmed by a specific renderer.
    at ./src/react/packages/react-reconciler/src/ReactFiberHostConfig.js
看注释可知，实际中该文件会被rollup注入代码，用来确认渲染环境，
此处需手动修改，将如下代码:
```
throw new Error('This module must be shimmed by a specific renderer.');
```
改为：
```
// 由rollup 等工具动态告诉渲染环境，在此我们是dom环境
export * from './forks/ReactFiberHostConfig.dom';
```
## 7 其他异常处理
此时运行 `yarn start` 不再报错，
但是打开源码文件比如`src/react/packages/rea
ct-reconciler/src/ReactFiberWorkLoop.old.js`
发现会有很多错误提示，这是react默认语法检查是flow，vscode把它当成了typescript，解决方法是把vscode 的
ts校验关掉。
setting -> workspace -> script validate
关掉 js 和 ts validate 即可。

# 调试
1 首先打开浏览器检查工具，查看火焰图(关注FCP之前的阶段)
2 查看，火焰图参考：https://www.debugbear.com/blog/devtools-performance
3 调试
4 peacock vscode color主题