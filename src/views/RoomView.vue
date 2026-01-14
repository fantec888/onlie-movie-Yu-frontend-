<template>
  <div class="room-layout">
    <div v-if="showVideoSettings" class="modal-overlay">
      <div class="modal-content">
        <h2>更换片源</h2>
        <p>输入新的视频播放地址</p>
        <div class="input-column">
          <input
            v-model="newVideoUrl"
            placeholder="例如: https://example.com/movie.mp4"
            @keyup.enter="updateVideoSource"
            ref="videoInput"
            autofocus
            class="full-width-input"
          />
          <div class="modal-actions">
            <button class="btn-text" @click="showVideoSettings = false">取消</button>
            <button class="btn-primary" @click="updateVideoSource">确认更换</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="!username" class="modal-overlay">
      <div class="modal-content">
        <h2>欢迎加入放映室</h2>
        <p>请给自己起个名字吧</p>
        <div class="input-row">
          <input
            v-model="tempUsername"
            placeholder="你的昵称"
            @keyup.enter="confirmUsername"
            ref="nameInput"
            autofocus
          />
          <button class="btn-primary" @click="confirmUsername">进入</button>
        </div>
      </div>
    </div>

    <div class="main-area" :class="{ blurred: !username }">
      <header class="room-header">
        <div class="room-info">
          <h1>{{ roomInfo.name || '未命名房间' }}</h1>
          <span class="room-id">ID: {{ roomId }}</span>
          <span class="socket-status" :class="{ connected: isSocketConnected }">
            {{ isSocketConnected ? '🟢 实时互动已连接' : '🔴 连接中...' }}
          </span>
        </div>
        <div class="header-actions">
          <button v-if="isOwner" class="btn-secondary" @click="showVideoSettings = true">设置片源</button>
          <button v-if="isOwner" class="btn-danger" @click="endSession">结束活动并解散</button>
          <button v-else class="btn-danger" @click="goBack">退出房间</button>
        </div>
      </header>

      <div class="player-container">
        <ArtPlayer
          :src="roomInfo.videoUrl"
          class="video-player"
          airplay
          aspect-ratio
          auto-size
          auto-orientation
          auto-playback
          fast-forward
          flip
          fullscreen-web
          lock
          loop
          muted
          mini-progress-bar
          pip
          screenshot
          subtitle-offset
        />

        <div class="danmaku-layer" ref="danmakuLayer"></div>
      </div>

      <div class="control-bar">
        <div class="playback-info">
          <span class="status-dot online"></span>
          <span>{{ connectedUsers }} 人在线 - 同步正常</span>
        </div>
        <button class="btn-sync" @click="forceSync">全员强制同步</button>
      </div>
    </div>

    <aside class="sidebar">
      <div class="tabs">
        <div
          class="tab"
          :class="{ active: currentTab === 'chat' }"
          @click="currentTab = 'chat'"
        >
          讨论区
        </div>
        <div
          class="tab"
          :class="{ active: currentTab === 'members' }"
          @click="currentTab = 'members'"
        >
          成员 ({{ members.length }})
        </div>
      </div>

      <div
        v-show="currentTab === 'chat'"
        class="chat-messages"
        ref="chatBox"
      >
        <div
          v-for="(msg, index) in messages"
          :key="index"
          class="message-item"
          :class="{ 'system-msg': msg.type === 'system', 'self-msg': msg.isSelf }"
        >
          <span v-if="msg.type !== 'system'" class="sender">{{ msg.sender }}:</span>
          <span class="text">{{ msg.content }}</span>
        </div>
      </div>

      <div v-show="currentTab === 'members'" class="member-list">
        <div v-for="m in members" :key="m.id" class="member-item-row">
          <div class="avatar-placeholder">{{ m.avatar }}</div>
          <div class="member-info">
            <span class="name">{{ m.name }}</span>
            <span v-if="m.role === 'owner'" class="badge-owner">房主</span>
          </div>
        </div>
      </div>

      <div class="chat-input-area" v-show="currentTab === 'chat'">
        <input
          v-model="newMessage"
          @keyup.enter="sendMessage"
          type="text"
          placeholder="参与讨论..."
          :disabled="!isSocketConnected"
        />
        <button @click="sendMessage" :disabled="!isSocketConnected">发送</button>
      </div>
    </aside>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import ArtPlayer from '../components/ArtPlayer.vue'
import roomApi from '@/api/roomApi'
// [新增] 引入 Socket.IO 客户端
import { io, Socket } from 'socket.io-client'

const route = useRoute()
const router = useRouter()
const roomId = route.params.id as string

// 从 localStorage 获取用户信息
const userId = ref(localStorage.getItem('userId') || '')
const userNickname = ref(localStorage.getItem('userNickname') || '')

// [新增] Socket 相关状态
const socket = ref<Socket | null>(null)
const isSocketConnected = ref(false)
const danmakuLayer = ref<HTMLElement | null>(null) // 弹幕容器引用

// 房间信息状态
const roomInfo = reactive({
  name: '',
  announcement: '',
  creatorId: '',
  currentCount: 0,
  capacity: 10,
  videoUrl: '',
  status: 'waiting'
})

// 参与者列表
const participants = ref<Array<{id: string; nickname: string; role: string; status: string}>>([])

const isOwner = computed(() => roomInfo.creatorId === userId.value)

// 用户名逻辑
const username = ref(userNickname.value)
const tempUsername = ref('')
const nameInput = ref<HTMLInputElement | null>(null)

// 片源设置逻辑
const showVideoSettings = ref(false)
const newVideoUrl = ref('')
const videoInput = ref<HTMLInputElement | null>(null)
const isUpdatingVideo = ref(false)

const confirmUsername = () => {
  if (!tempUsername.value.trim()) return
  username.value = tempUsername.value
  userNickname.value = tempUsername.value
  localStorage.setItem('userNickname', username.value)
  // [新增] 如果Socket未连接，确认用户名后尝试连接
  if (!isSocketConnected.value) {
    initSocket()
  }
}

const updateVideoSource = async () => {
  if (!newVideoUrl.value.trim()) {
    alert('请输入有效的视频地址')
    return
  }

  isUpdatingVideo.value = true

  try {
    const response = await roomApi.update(roomId, {
      operatorId: userId.value,
    })

    if (response.data.success) {
      roomInfo.videoUrl = newVideoUrl.value
      messages.push({
        type: 'system',
        content: '房主更新了片源'
      })
      showVideoSettings.value = false
      newVideoUrl.value = ''
    }
  } catch (error: any) {
    alert('更新片源失败：' + (error.response?.data?.message || error.message))
  } finally {
    isUpdatingVideo.value = false
  }
}

// 状态模拟
const currentTab = ref<'chat' | 'members'>('chat')
const connectedUsers = ref(1)
const isConnected = ref(false)
const isLoading = ref(true)
const errorMessage = ref('')
const messages = reactive<Array<{type: string; sender?: string; content: string; isSelf?: boolean}>>([])
const newMessage = ref('')

// 成员列表（映射到 participants）
const members = computed(() => {
  return participants.value.map(p => ({
    id: p.id,
    name: p.nickname,
    role: p.role === 'creator' ? 'owner' : 'guest',
    avatar: p.nickname.charAt(0)
  }))
})

const goBack = () => {
  leaveRoomAPI()
}

const chatBox = ref<HTMLElement | null>(null)

// ==========================================
// 初始化与连接
// ==========================================
const loadRoomDetail = async () => {
  isLoading.value = true
  errorMessage.value = ''

  try {
    const response = await roomApi.getDetail(roomId)

    if (response.data.success) {
      const room = response.data.data
      roomInfo.name = room.name
      roomInfo.announcement = room.announcement
      roomInfo.creatorId = room.creatorId
      roomInfo.currentCount = room.currentCount
      roomInfo.capacity = room.capacity
      roomInfo.status = room.status
      participants.value = room.participants || []

      // 模拟初始视频源
      roomInfo.videoUrl = room.videoState?.source || 'https://vp-demo.u2sb.com/video/caminandes_03_llamigos_720p.mp4'

      // 添加系统消息
      messages.push({
        type: 'system',
        content: `欢迎加入房间 #${roomId}`
      })
      messages.push({
        type: 'system',
        content: `房间成员：${room.currentCount} 人`
      })

      // [新增] 房间详情加载完毕后，初始化Socket
      if (userId.value) {
        initSocket()
      }
    }
  } catch (error: any) {
    errorMessage.value = error.response?.data?.message || '加载房间详情失败'
    console.error('加载房间失败:', error)
  } finally {
    isLoading.value = false
  }
}

const leaveRoomAPI = async () => {
  try {
    await roomApi.leave(roomId, userId.value)
    localStorage.removeItem('currentRoomId')
    router.push('/join')
  } catch (error: any) {
    console.error('退出房间失败:', error)
    router.push('/join')
  }
}

// [新增] Socket 初始化函数
const initSocket = () => {
  // 防止重复连接
  if (socket.value && socket.value.connected) return

  // 连接到后端 Socket 服务
  socket.value = io('http://localhost:3000/chat', {
    transports: ['websocket'],
    reconnection: true
  })

  // 监听连接成功
  socket.value.on('connect', () => {
    isSocketConnected.value = true
    console.log('Socket 连接成功:', socket.value?.id)

    // 发送加入房间请求
    socket.value?.emit('room:join', {
      roomId,
      participantId: userId.value
    }, (ack: any) => {
      if (ack.ok) {
        console.log('成功加入聊天频道:', ack.data.channel)
      } else {
        console.error('加入聊天频道失败:', ack.error)
      }
    })
  })

  // 监听连接断开
  socket.value.on('disconnect', () => {
    isSocketConnected.value = false
    console.log('Socket 断开连接')
  })

  // 监听新消息 (同时处理聊天列表和弹幕)
  socket.value.on('message:new', (msg: any) => {
    // 1. 添加到聊天列表
    messages.push({
      type: 'chat',
      sender: msg.from.nickname,
      content: msg.content,
      isSelf: msg.from.participantId === userId.value // 判断是否是自己发的
    })
    scrollToBottom()

    // 2. 发射弹幕
    shootDanmaku(msg.content)
  })
}

// [新增] 弹幕发射逻辑
const shootDanmaku = (text: string) => {
  if (!danmakuLayer.value) return

  const el = document.createElement('div')
  el.innerText = text
  el.className = 'danmaku-item'

  // 随机高度 (0% - 80%)，避免遮挡底部控制条
  el.style.top = Math.floor(Math.random() * 80) + '%'
  // 随机颜色 (可选，这里用白色)
  el.style.color = '#fff'

  danmakuLayer.value.appendChild(el)

  // 动画结束后移除元素 (时间需与 CSS animation 匹配)
  setTimeout(() => {
    el.remove()
  }, 6000)
}

onMounted(() => {
  if (!userId.value || !userNickname.value) {
    alert('请先加入房间')
    router.push('/join')
    return
  }

  loadRoomDetail()
  isConnected.value = true
  connectedUsers.value = participants.value.length
})

onUnmounted(() => {
  leaveRoomAPI()
  // [新增] 组件销毁时断开 Socket
  if (socket.value) {
    socket.value.disconnect()
  }
})

// ==========================================
// 播放控制
// ==========================================
const handlePlay = () => {
  console.log('Local Play Triggered')
}

const handlePause = () => {
  console.log('Local Pause Triggered')
}

const handleSeek = () => {
  console.log('Local Seek Triggered')
}

const forceSync = () => {
  messages.push({
    type: 'system',
    content: '管理员已执行全员强制同步'
  })
}

// ==========================================
// 互动
// ==========================================
const sendMessage = () => {
  if (!newMessage.value.trim()) return

  // [修改] 原有的本地 push 逻辑已移除，改为通过 Socket 发送
  // 消息显示将由 socket.on('message:new') 统一处理，避免重复
  if (socket.value && isSocketConnected.value) {
    socket.value.emit('message:send', {
      roomId,
      participantId: userId.value,
      content: newMessage.value
    }, (ack: any) => {
      if (ack.ok) {
        // 发送成功，清空输入框
        newMessage.value = ''
      } else {
        alert('消息发送失败: ' + ack.error?.message)
      }
    })
  } else {
    alert('聊天服务未连接，请刷新重试')
  }
}

const scrollToBottom = () => {
  nextTick(() => {
    if (chatBox.value) {
      chatBox.value.scrollTop = chatBox.value.scrollHeight
    }
  })
}

// ==========================================
// 结束活动
// ==========================================
const endSession = async () => {
  if (!confirm('确定要解散房间吗？所有成员将被移出。')) return

  try {
    const response = await roomApi.dissolve(roomId, userId.value)

    if (response.data.success) {
      alert('房间已解散')
      localStorage.removeItem('currentRoomId')
      router.push('/')
    }
  } catch (error: any) {
    const errorCode = error.response?.data?.errorCode

    if (errorCode === 'PERMISSION_DENIED') {
      alert('只有房间创建者才能解散房间')
    } else {
      alert('解散房间失败：' + (error.response?.data?.message || error.message))
    }
  }
}
</script>

<style scoped>
.room-layout {
  display: flex;
  height: 100vh;
  background-color: #000;
  color: #fff;
  overflow: hidden;
}

.main-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #333;
}

.room-header {
  padding: 1rem 1.5rem;
  background-color: #121212;
  border-bottom: 1px solid #333;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* [新增] Socket 状态样式 */
.socket-status {
  font-size: 0.8rem;
  margin-left: 10px;
  color: #e53e3e;
}
.socket-status.connected {
  color: #4cd964;
}

.blurred {
  filter: blur(5px);
  pointer-events: none;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.8);
  z-index: 9999;
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: #1e1e1e;
  padding: 2rem;
  border-radius: 12px;
  text-align: center;
  box-shadow: 0 10px 25px rgba(0,0,0,0.5);
  border: 1px solid #333;
}

.modal-content h2 {
  margin-top: 0;
  color: #fff;
}

.modal-content p {
  color: #aaa;
  margin-bottom: 1.5rem;
}

.input-row {
  display: flex;
  gap: 1rem;
}

.input-row input {
  padding: 0.8rem;
  background: #2a2a2a;
  border: 1px solid #444;
  color: #fff;
  border-radius: 6px;
  outline: none;
}

.input-row button {
  white-space: nowrap;
}

.room-info h1 {
  font-size: 1.2rem;
  margin: 0;
  color: #eee;
}

.room-id {
  font-size: 0.8rem;
  color: #666;
  margin-top: 2px;
  display: block;
}

.player-container {
  flex: 1;
  background-color: #000;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* [新增] 弹幕层样式 */
.danmaku-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none; /* 关键：点击穿透 */
  z-index: 20; /* 确保在视频上方 */
  overflow: hidden;
}

/* [新增] 动态弹幕元素样式 (注意使用 deep 选择器) */
:deep(.danmaku-item) {
  position: absolute;
  right: -100%;
  font-size: 24px;
  font-weight: bold;
  white-space: nowrap;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.8);
  animation: fly 6s linear forwards;
}

@keyframes fly {
  from { left: 100%; transform: translateX(0); }
  to { left: 0; transform: translateX(-100%); }
}

.video-player, .iframe-player {
  width: 100%;
  height: 100%;
  max-height: 80vh;
}

.empty-player {
  color: #444;
}

.control-bar {
  height: 60px;
  background-color: #1a1a1a;
  display: flex;
  align-items: center;
  padding: 0 1.5rem;
  justify-content: space-between;
  border-top: 1px solid #333;
}

.status-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  margin-right: 8px;
}

.status-dot.online {
  background-color: #4cd964;
  box-shadow: 0 0 5px #4cd964;
}

.playback-info {
  font-size: 0.9rem;
  color: #aaa;
  display: flex;
  align-items: center;
}

/* Sidebar Chat */
.sidebar {
  width: 350px;
  background-color: #121212;
  display: flex;
  flex-direction: column;
}

.tabs {
  display: flex;
  border-bottom: 1px solid #333;
}

.tab {
  flex: 1;
  text-align: center;
  padding: 1rem;
  cursor: pointer;
  color: #888;
  font-size: 0.9rem;
}

.tab.active {
  color: #667eea;
  border-bottom: 2px solid #667eea;
  background-color: #1a1a1a;
}

.chat-messages {
  flex: 1;
  padding: 1rem;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

.message-item {
  font-size: 0.9rem;
  line-height: 1.4;
  word-break: break-all;
}

.message-item.system-msg {
  text-align: center;
  color: #555;
  font-size: 0.8rem;
  margin: 0.5rem 0;
}

.message-item .sender {
  color: #667eea;
  font-weight: bold;
  margin-right: 6px;
}

.message-item.self-msg {
  text-align: right;
}

.message-item.self-msg .text {
  background-color: #2a2a2a;
  padding: 4px 8px;
  border-radius: 4px;
}

.chat-input-area {
  padding: 1rem;
  background-color: #1a1a1a;
  border-top: 1px solid #333;
  display: flex;
  gap: 0.5rem;
}

input {
  flex: 1;
  background-color: #2c2c2c;
  border: 1px solid #444;
  padding: 0.6rem;
  color: white;
  border-radius: 4px;
}

input:focus {
  outline: none;
  border-color: #667eea;
}

button {
  cursor: pointer;
  border: none;
  border-radius: 4px;
  padding: 0 1rem;
  font-weight: 500;
  transition: opacity 0.2s;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-danger {
  background-color: #e53e3e;
  color: white;
  padding: 0.5rem 1rem;
}

/* Header Buttons */
.btn-secondary {
  background-color: transparent;
  border: 1px solid #444;
  color: #ccc;
  padding: 0.5rem 1rem;
  margin-right: 1rem;
}

.btn-secondary:hover {
  border-color: #666;
  color: #fff;
}

/* Member List Styles */
.member-list {
  flex: 1;
  padding: 1rem;
  overflow-y: auto;
}

.member-item-row {
  display: flex;
  align-items: center;
  padding: 0.8rem 0.5rem;
  border-bottom: 1px solid #2a2a2a;
  transition: background-color 0.2s;
}

.member-item-row:hover {
  background-color: #1a1a1a;
}

.avatar-placeholder {
  width: 36px;
  height: 36px;
  background-color: #333;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.2rem;
  margin-right: 12px;
}

.member-info {
  display: flex;
  flex-direction: column;
}

.member-info .name {
  font-size: 0.95rem;
  color: #e0e0e0;
}

.badge-owner {
  font-size: 0.7rem;
  background-color: #d69e2e; /* Gold/Yellowish */
  color: #000;
  padding: 1px 4px;
  border-radius: 3px;
  margin-top: 2px;
  align-self: flex-start;
  font-weight: bold;
}

.btn-danger:hover {
  background-color: #c53030;
}

.btn-sync {
  background-color: #4a5568;
  color: white;
  padding: 0.5rem 1rem;
}

.btn-sync:hover {
  background-color: #2d3748;
}

.chat-input-area button {
  background-color: #667eea;
  color: white;
}

/* Modal Forms */
.input-column {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
  margin-top: 1rem;
}

.full-width-input {
  width: 100%;
  box-sizing: border-box;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 0.5rem;
}

.btn-text {
  background: transparent;
  color: #aaa;
  padding: 0.5rem 1rem;
}

.btn-text:hover {
  color: #fff;
}
</style>
