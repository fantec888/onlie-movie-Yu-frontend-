# 项目开发指南 & 结构说明

## 1. 项目结构核心文件

| 路径 | 说明 |
| :--- | :--- |
| `src/views/HomeView.vue` | **大厅首页**。包含创建房间的入口，简约风格。 |
| `src/views/CreateRoomView.vue` | **创建房间页**。包含房间配置表单（标题、最大人数、密码、片源地址）。<br/>**对接点**：向后端 POST 提交创建请求。 |
| `src/views/RoomView.vue` | **放映室主页 (核心)**。包含：<br/>1. 播放器 (Video/Iframe)<br/>2. 控制栏 (同步状态/强制同步)<br/>3. 聊天与成员列表侧边栏<br/>**对接点**：WebSocket 连接、状态同步、消息收发。 |
| `src/views/JoinView.vue` | **加入房间页**:显示房间列表并加入和通过搜索加入<br/>**对接点**：向后端 请求房间列表数据，和POST 提交加入请求。 |
| `src/router/index.ts` | **路由配置**。定义了页面的跳转规则。 |

---

## 2. 前后端对接指南

本项目前端采用 Vue 3 composition API，推荐使用 WebSocket 实现实时互动。

### A. 创建房间 (REST API)

*   **位置**: `src/views/CreateRoomView.vue` -> `handleCreate`
*   **请求**: POST `/api/rooms`
*   **数据**:
    ```json
    {
      "title": "周末电影夜",
      "notice": "请勿剧透",
      "maxUsers": 10,
      "password": "123",
      "videoUrl": "https://..."
    }
    ```
*   **响应**: 返回 `{ "id": "room_1001", "token": "..." }`。前端拿到 ID 后跳转至 `RoomView`。

### B. 房间连接 (WebSocket)

*   **位置**: `src/views/RoomView.vue`
*   **初始化**: 在 `onMounted` 中建立连接。
    ```javascript
    const ws = new WebSocket(`wss://your-api.com/ws/room/${roomId}?token=${userToken}`);
    ```
*   **心跳保活**: 建议每 30s 发送一次 ping，避免连接断开。

### C. 播放状态同步 (核心难点)

前端仅负责**上报操作**和**响应指令**，不要直接操作播放器进度（除非是响应后端的指令）。

1.  **用户操作 (如点击播放)**:
    *   前端拦截事件（或监听 `play` 事件），阻止默认行为（如果可能）或立即发送指令。
    *   发送: `{ "type": "ACTION_PLAY", "timestamp": 12.5 }`
2.  **后端广播**:
    *   后端接收到指令，广播给**房间内所有人** (包括发送者)。
    *   广播: `{ "type": "SYNC_PLAY", "timestamp": 12.5 }`
3.  **前端响应**:
    *   收到 `SYNC_PLAY` -> 执行 `videoRef.value.currentTime = 12.5; videoRef.value.play()`。

### D. 聊天与成员列表

*   **聊天**: 发送 `{ "type": "CHAT", "content": "..." }`，接收广播并 push 到 `messages` 数组。
*   **成员变动**: 监听 `{ "type": "MEMBER_UPDATE", "data": [ ... ] }` 消息，全量更新 `members` 响应式对象。

---

## 3. 下一步开发建议

1.  **引入 Pinia**: 目前房间状态耦合在 `RoomView` 组件内。建议引入 Pinia 创建 `useRoomStore`，管理 WebSocket 连接实例和房间状态，这样在“返回大厅”时连接不会丢失。
2.  **完善播放器**: 原生 `<video>` 功能有限。建议封装 `Video.js` 或 `ArtPlayer`，能更好地支持 HLS (`.m3u8`) 流媒体和弹幕功能。
3.  **鉴权机制**: 在路由钩子 (`router.beforeEach`) 中增加鉴权，非房间成员禁止直接通过 URL 访问房间。
