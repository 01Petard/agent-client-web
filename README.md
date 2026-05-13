# agent-client-web — 智能咖啡客服前端

基于 Vue 3 + Vite 构建的智能咖啡客服 Web 客户端，支持 AI 对话、订单管理、用户管理等。

## 技术栈

| 模块      | 选型                              |
|---------|----------------------------------|
| 框架      | Vue 3 (Composition API)          |
| 构建工具   | Vite 5                           |
| HTTP    | Axios                            |
| Markdown | marked + DOMPurify               |
| Node    | ^20.19.0 \|\| >=22.12.0          |
| 包管理    | pnpm / npm                       |

## 项目结构

```
src/
├── App.vue                        # 主应用（标签页切换 + 登录状态管理）
├── main.js                        # 入口
├── components/
│   ├── AuthModal.vue              # 登录/注册模态框
│   ├── ChatInterface.vue          # 聊天页面（流式SSE + 历史加载）
│   └── OrderManagement.vue        # 订单后台页面
└── services/
    └── api.js                     # API 封装（接口请求 + 流式 SSE 消费）
```

## 功能

### 聊天页面

- **流式 SSE 对话**：输入消息后，AI 回复逐字推送，实时显示
- **Markdown 富文本渲染**：支持加粗、列表、代码块等
- **上下文压缩**：长对话自动压缩，节省 Token
- **历史记录恢复**：登录后自动加载历史对话
- **Alt+Enter 换行**：支持多行输入
- **清除对话**：一键删除所有历史消息

### 订单后台

- **订单列表**：实时查看所有订单及状态
- **操作按钮**：完成 / 取消订单

### 用户管理

- **登录/注册**：输入用户名+手机号即可登录或注册
- **持久化登录**：刷新页面自动恢复登录状态（localStorage）
- **退出登录**：清除本地状态，返回登录页

## 快速开始

### 环境要求

- Node.js >= 22.12.0
- npm >= 11.x (推荐使用 pnpm)

### 安装

```bash
cd agent-client-web

# 使用 npm
npm install

# 或使用 pnpm
pnpm install
```

### 开发

```bash
npm run dev
```

默认启动在 `http://localhost:5173`，通过 Vite proxy 将 `/api` 请求代理到后端 `http://localhost:8080`。

### 构建

```bash
npm run build
```

产物输出到 `dist/` 目录。

### Vite 配置说明

```javascript
// vite.config.js
server: {
  proxy: {
    '/api': {
      target: 'http://127.0.0.1:8080',  // 后端地址
      changeOrigin: true,
    }
  }
}
```

## 与后端联调

1. 启动 `agent-client-server`（后端，端口 8080）
2. 按需执行 `db/schema.sql` 初始化数据库
3. 启动 Redis（用于向量存储 RAG）
4. 启动 `agent-client-web`（前端，端口 5173）
5. 访问 `http://localhost:5173`

### 用户流程

1. 打开页面 → 弹出登录/注册框
2. 新用户：点击「去注册」→ 输入用户名+手机号 → 自动登录
3. 老用户：输入用户名+手机号 → 登录 → 恢复历史对话
4. 在聊天页输入问题，AI 自动回答
5. 点「订单后台」标签查看订单状态
6. 可随时「清除对话」或「退出登录」
