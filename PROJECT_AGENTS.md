# QQ Music API 项目指南

本文档为在 qq-music-api 代码库中工作提供构建命令和代码风格指南。

## 构建 / Lint / 测试 命令

**开发与生产环境：**

- `npm run dev` - 启动开发服务器，包括 nodemon 和文档服务器（端口 3200，文档在 9611）
- `npm start` - 启动生产服务器（node app.js，端口 3200）

**代码质量：**

- `npm run eslint` - 在 module/、routers/、util/ 目录上运行 ESLint 并自动修复
- `npm run prettier` - 使用 Prettier 格式化代码（2空格缩进，单引号）
- `npm run lint-staged` - 运行 lint-staged（通过 Husky 在 pre-commit 时自动运行）

**文档：**

- `npm run docs` - 在端口 9611 启动 docsify 文档服务器

**Docker：**

- `npm run build:local-images` - 构建本地 Docker 镜像
- `npm run build:remote-images` - 构建远程 Docker 镜像
- `npm run build:images` - 构建本地和远程 Docker 镜像
- `npm run run:images` - 运行 Docker 容器

**注意：** 本项目目前没有单元测试（在 README 中提到）。

## 代码风格指南

### 格式化与 Linting

- **缩进：** 2 个空格（代码按 .editorconfig 使用 tab，但 prettier/eslint 使用 2 个空格）
- **引号：** 单引号（ESLint 和 Prettier 强制执行）
- **分号：** 始终需要（ESLint 和 Prettier 强制执行）
- **行宽：** 最多 120 个字符（Prettier）
- **行尾：** Unix (LF)
- **尾随空格：** 删除所有尾随空格
- **文件末尾换行：** 文件末尾需要换行（.md 文件除外）
- **编码：** UTF-8

### 导入与导出

- 整个项目使用 CommonJS `require()` 语法
- 有帮助时使用解构导入：`const { getLyric } = require('../../module');`
- 使用相对路径和 `../../` 表示法来引用本地模块
- 模块导出：多个导出使用 `module.exports = { ... }`
- 函数导出：单个函数使用 `module.exports = (params) => { ... }`

### 命名约定

- **函数：** camelCase（例如：`getLyric`、`setCookie`、`batchGetSongInfo`）
- **变量：** camelCase（例如：`songmid`、`isFormat`、`baseURL`）
- **常量：** camelCase（例如：`commonParams`、`yURL`、`cURL`）
- **类：** 不适用（此代码库中未使用类）
- **文件名：** camelCase（例如：`getLyric.js`、`y_common.js`、`request.js`）
- **目录名：** 小写（例如：`module`、`routers`、`util`、`middlewares`）

### 代码结构模式

**API 模块函数（在 module/apis/ 中）：**

```javascript
module.exports = ({ method = 'get', params = {}, option = {}, isFormat = false }) => {
	const data = Object.assign(params, {
		format: 'json',
		outCharset: 'utf-8',
		pcachetime: moment().valueOf(),
	});
	return request('/api/endpoint', method, { params: data })
		.then(res => {
			// 转换响应
			return { status: 200, body: { response: transformedData } };
		})
		.catch(error => {
			console.log('error', error);
			return { body: { error } };
		});
};
```

**路由上下文处理器（在 routers/context/ 中）：**

```javascript
module.exports = async (ctx, next) => {
	const { param1, param2 } = ctx.query;
	if (param1) {
		const { status, body } = await apiFunction({ param1, param2 });
		Object.assign(ctx, { status, body });
	} else {
		ctx.status = 400;
		ctx.body = { response: 'missing parameter' };
	}
};
```

**路由定义（在 routers/router.js 中）：**

- 使用 koa-router
- GET 端点：`router.get('/path/:param?', context.handlerFunction);`
- POST 端点：`router.post('/path', context.handlerFunction);`
- 可选参数在路由模式中用 `?` 标记

### 错误处理

- 使用 `.catch()` 处理 promise 上的错误
- 在响应体中返回错误对象：`{ body: { error } }`
- 将错误记录到控制台：`console.log('error', error);`
- 对于路由处理器，设置 HTTP 状态码：`ctx.status = 400;`

### 项目架构

**目录结构：**

- `app.js` - 主应用程序入口点，Koa 设置
- `module/` - 向 QQ Music 发起 HTTP 请求的 API 模块函数
  - `apis/` - 按域组织的单个 API 实现
  - `config.js` - 公共配置和参数
  - `index.js` - 所有 API 模块的中央导出
- `routers/` - Koa 路由定义和处理器
  - `context/` - 单个路由处理器函数
  - `router.js` - 中央路由注册
- `util/` - 实用工具函数（请求、解析、助手）
- `middlewares/` - Koa 中间件（CORS 等）
- `config/` - 配置文件（user-info.js）

**全局变量：**

- `global.userInfo` - 从 config/user-info.js 加载的用户认证信息

### 关键依赖

- **koa** - Web 框架（v2+）
- **koa-router** - 路由
- **axios** - 用于向 QQ Music API 发起请求的 HTTP 客户端
- **moment** - 日期/时间工具
- **lodash.get** - 安全属性访问

### 提交约定

- 使用 conventional-changelog 进行提交消息
- Husky pre-commit 钩子运行 lint-staged
- Commitlint 强制执行约定式提交

### 端口配置

- 开发服务器：3200
- 文档服务器：9611
