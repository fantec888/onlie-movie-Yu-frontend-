# 🍿 Online Movie House (Frontend)

基于 Vue 3 的在线多人同步观影系统前端项目。

## 📅 当前进度：第三次迭代 (v1.3)
**核心目标**：实现前端聊天界面与视频弹幕互动。

### ✨ 本次迭代新增功能
1.  **Socket 客户端集成**：
    - 引入 `socket.io-client` 库。
    - 实现了进入房间自动连接 WebSocket 服务。
2.  **实时聊天系统**：
    - 完成了左侧聊天列表的 UI 开发。
    - 实现了多用户实时收发消息，状态同步。
3.  **视频弹幕层 (Danmaku)**：
    - 在视频播放器上方覆盖透明弹幕层。
    - 实现了 CSS 动画 (`@keyframes fly`)，让接收到的消息实时飘过屏幕。
4.  **连接状态管理**：
    - 实时显示 Socket 连接状态 (🟢 在线 / 🔴 离线)。

---

## 🛠 技术栈
- **Framework**: Vue 3 + Vite
- **Language**: TypeScript
- **Real-time**: Socket.io-client
- **Player**: ArtPlayer

## 🚀 快速开始

### 1. 安装依赖
```bash
npm install
