<template>
  <div class="app-container">
    <!-- 背景图 (使用了一张古风人物的占位图) -->
    <div class="background-wrapper">
      <img src="https://jyt.tqingdao.com/attachment/images/2026/03/20/image_1773973044_J9I8I1R0.jpg" class="bg-image" alt="background" />
      <div class="bg-overlay"></div>
    </div>

    <!-- 顶部标题 -->
    <div class="top-header">
      AI数字人交互界面
    </div>

    <!-- 摄像头预览区域 (右上角) -->
    <div class="camera-preview" v-show="isConnected && isCameraOn">
      <video ref="localVideoRef" autoplay playsinline muted></video>
    </div>

    <!-- 摄像头控制按钮 (悬浮在右上角摄像头下方) -->
    <div class="camera-toggle" v-if="isConnected" @click="toggleCamera">
      {{ isCameraOn ? '关闭摄像头' : '开启摄像头' }}
    </div>

    <!-- 主交互区 -->
    <div class="main-content">
      
      <!-- 聊天气泡区域 -->
      <div class="chat-container">
        <div class="chat-header">
          AI数字人交互界面
        </div>
        <div class="message-list" ref="messageListRef">
          <div 
            v-for="(msg, index) in messages" 
            :key="index"
            :class="['message-wrapper', msg.role === 'user' ? 'is-user' : 'is-ai']"
          >
            <div class="message-bubble">
              {{ msg.content }}
            </div>
          </div>
        </div>
        
        <!-- 语音波形与状态区 -->
        <div class="voice-status-area">
          <div class="wave-container" v-if="isConnected">
            <div class="wave-bar" v-for="i in 30" :key="i" :style="{ animationDelay: `${Math.random() * 0.5}s` }"></div>
          </div>
          
          <div class="status-text">
            <span v-if="!isConnected">未连接，点击下方按钮开始</span>
            <span v-else-if="isListening">正在倾听...</span>
            <span v-else>等待中...</span>
          </div>
          
          <div class="asr-text" v-if="currentAsrText">
            “{{ currentAsrText }}”
          </div>
        </div>

        <!-- 底部控制按钮 -->
        <div class="controls-area">
          <div class="side-btn">
            <div class="icon">⚙️</div>
            <span>设置</span>
          </div>
          
          <button 
            class="mic-btn" 
            :class="{ active: isConnected }"
            @click="toggleConnect"
          >
            <span class="mic-icon">🎤</span>
          </button>
          
          <div class="side-btn">
            <div class="icon">❓</div>
            <span>帮助</span>
          </div>
        </div>
      </div>
      
    </div>

    <!-- 底部导航 -->
    <div class="bottom-nav">
      <div class="nav-item">
        <div class="nav-icon">🏠</div>
        <span>首页</span>
      </div>
      <div class="nav-item active">
        <div class="nav-icon">💬</div>
        <span>聊天</span>
      </div>
      <div class="nav-item">
        <div class="nav-icon">⏱️</div>
        <span>历史</span>
      </div>
      <div class="nav-item">
        <div class="nav-icon">👤</div>
        <span>个人中心</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted, nextTick } from 'vue';
import { RealtimeClient, RealtimeUtils, EventNames } from '@coze/realtime-api';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
}

const isConnected = ref(false);
const isListening = ref(false);
const isCameraOn = ref(true);
const currentAsrText = ref('');
const messages = ref<Message[]>([]);
const messageListRef = ref<HTMLElement | null>(null);
const localVideoRef = ref<HTMLVideoElement | null>(null);

let client: RealtimeClient | null = null;
let currentAiMessageId = '';
let localMediaStream: MediaStream | null = null;

const config = {
  accessToken: 'pat_3l2YLpNLbN5dbASKYcntLOvxHJ39owN7J6OcrHSbWszHwucxkzmitavX1M9PemHG', 
  botId: '7618851548236005391',     
  connectorId: '1024',       
};

const scrollToBottom = () => {
  nextTick(() => {
    if (messageListRef.value) {
      messageListRef.value.scrollTop = messageListRef.value.scrollHeight;
    }
  });
};

const initClient = async () => {
  try {
    const permission = await RealtimeUtils.checkDevicePermission(true);
    // 这里我们直接用浏览器原生API请求摄像头，因为 sdk 内部配置好像变了
    if (!permission.audio) throw new Error('需要麦克风访问权限');
    
    try {
      localMediaStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
    } catch(err) {
      console.warn('获取摄像头失败:', err);
    }

    client = new RealtimeClient({
      accessToken: config.accessToken,
      botId: config.botId,
      connectorId: config.connectorId,
      allowPersonalAccessTokenInBrowser: true,
      debug: true,
      
      // --- AI 降噪与语音增强 ---
      suppressStationaryNoise: true,     // 开启静态噪声抑制（过滤空调、风扇等持续噪音）
      suppressNonStationaryNoise: true,  // 开启非静态噪声抑制（过滤键盘、咳嗽等突发噪音）
      dereverberation: true,             // 开启回声消除（减少空旷房间或设备外放的回声）

      // --- 语音触发与打断配置 ---
      turnDetection: {
        type: "server_vad",
        interrupt_config: {
          mode: "keyword_contains",      // 包含关键词即打断
          keywords: ["大阳山助手"]        // 自定义唤醒/打断关键词
        },
        server_vad_config: {
          voice_prob_threshold: 0.8,     // 提高语音判定阈值，减少嘈杂环境误触发
          min_voice_duration_ms: 500,    // 延长最短语音时长
          max_silence_duration_ms: 1000  // 延长最大静音超时
        }
      }
    } as any);

    client.on(EventNames.ALL, (eventName, data: any) => {
      // 恢复全部日志输出，方便排查所有接收到的事件
      console.log(`事件: ${eventName}`, data);

      if (eventName === EventNames.CONNECTED) {
        isConnected.value = true;
        nextTick(() => {
          if (localVideoRef.value && localMediaStream) {
            localVideoRef.value.srcObject = localMediaStream;
          }
        });

        // 暂时注释掉发送打断配置的代码，看看是不是它导致了连接中断或后续不返回消息
        /*
        // 尝试发送事件配置关键词打断
        // 根据文档，WebSocket 场景用 chat.update，RTC 音视频场景用 session.update
        // 由于我们这里使用了摄像头，可能被识别为音视频场景
        try {
          client?.sendMessage({
            "id": `event_${Date.now()}`,
            "event_type": "session.update",
            "data": {
              "chat_config": {
                "allow_voice_interrupt": true,
                "interrupt_config": {
                  "mode": "keyword_contains",
                  "keywords": ["大阳山助手", "大阳山"]
                }
              }
            }
          });
          console.log("已发送 session.update 配置关键词打断");
        } catch(e) {
          console.error("配置 session.update 打断词失败:", e);
        }

        // 同时保留 chat.update 以防万一
        try {
          client?.sendMessage({
            "id": `event_${Date.now() + 1}`,
            "event_type": "chat.update",
            "data": {
              "turn_detection": {
                "type": "server_vad",
                "interrupt_config": {
                  "mode": "keyword_contains",
                  "keywords": ["大阳山助手", "大阳山"]
                },
                "server_vad_config": {
                  "voice_prob_threshold": 0.8,
                  "min_voice_duration_ms": 500,
                  "max_silence_duration_ms": 1000
                }
              },
              "asr_config": {
                "hot_words": ["大阳山助手", "大阳山"]
              }
            }
          });
          console.log("已发送 chat.update 配置关键词打断");
        } catch(e) {
          console.error("配置 chat.update 打断词失败:", e);
        }
        */
      }
      
      if (eventName === EventNames.DISCONNECTED) {
        isConnected.value = false;
        isListening.value = false;
        if (localMediaStream) {
          localMediaStream.getTracks().forEach(track => track.stop());
          localMediaStream = null;
        }
      }

      // 用户开始说话
      if (eventName === EventNames.AUDIO_USER_SPEECH_STARTED) {
        isListening.value = true;
        currentAsrText.value = '';
      }

      // 用户停止说话
      if (eventName === EventNames.AUDIO_USER_SPEECH_STOPPED) {
        isListening.value = false;
        // 把最后识别的话加到消息列表
        if (currentAsrText.value.trim()) {
          messages.value.push({
            id: Date.now().toString(),
            role: 'user',
            content: currentAsrText.value
          });
          currentAsrText.value = '';
          scrollToBottom();
        }
      }

      // 用户语音识别中间结果
      if (eventName === EventNames.CONVERSATION_AUDIO_TRANSCRIPT_DELTA) {
        console.log("收到用户识别文本:", data);
        const asrContent = data?.content || data?.data?.content || '';
        if (asrContent) {
          currentAsrText.value += asrContent;
        }
      }

      // AI 回复中间结果
      if (eventName === EventNames.CONVERSATION_MESSAGE_DELTA) {
        console.log("收到AI文本:", data); // 增加日志以便调试
        const content = data?.content || data?.data?.content || ''; // 兼容不同的数据结构
        if (!content) return;
        
        const lastMsg = messages.value[messages.value.length - 1];
        // 如果当前没有正在记录的AI消息ID，或者最后一条消息不是当前的AI消息
        if (lastMsg && lastMsg.role === 'assistant' && lastMsg.id === currentAiMessageId) {
          lastMsg.content += content;
        } else {
          currentAiMessageId = data?.message_id || data?.data?.message_id || Date.now().toString();
          messages.value.push({
            id: currentAiMessageId,
            role: 'assistant',
            content: content
          });
        }
        scrollToBottom();
      }
      
      // AI 回复完成
      if (eventName === EventNames.CONVERSATION_MESSAGE_COMPLETED) {
        console.log("AI文本回复完成:", data);
        currentAiMessageId = '';
      }
    });

    return client;
  } catch (error) {
    console.error('初始化失败:', error);
    alert(`初始化失败: ${(error as Error).message}`);
    throw error;
  }
};

const toggleConnect = async () => {
  if (isConnected.value) {
    await client?.disconnect();
  } else {
    if (!client) {
      await initClient();
    }
    await client?.connect();
  }
};

const toggleCamera = () => {
  if (localMediaStream) {
    const videoTrack = localMediaStream.getVideoTracks()[0];
    if (videoTrack) {
      isCameraOn.value = !isCameraOn.value;
      videoTrack.enabled = isCameraOn.value;
    }
  }
};

onUnmounted(() => {
  if (client) {
    client.disconnect();
    client = null;
  }
});
</script>

<style scoped>
/* 全局充当移动端视图 */
.app-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  background-color: #000;
  color: #fff;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

/* 背景图处理 */
.background-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
}
.bg-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.8;
}
.bg-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(to right, rgba(0,0,0,0.2) 0%, rgba(0,0,0,0.7) 100%);
}

/* 顶部标题 */
.top-header {
  position: relative;
  z-index: 10;
  text-align: center;
  padding: 16px 0;
  font-size: 18px;
  font-weight: 500;
  background: rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(10px);
}

/* 摄像头预览区 */
.camera-preview {
  position: absolute;
  top: 60px; /* 在标题下方 */
  right: 20px;
  width: 120px;
  height: 160px;
  z-index: 20;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
  border: 2px solid rgba(255, 255, 255, 0.2);
  background-color: #000;
}

.camera-preview video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1); /* 镜像翻转 */
}

/* 摄像头控制按钮 */
.camera-toggle {
  position: absolute;
  top: 230px; /* 在摄像头预览区下方 */
  right: 20px;
  z-index: 20;
  background: rgba(0, 0, 0, 0.6);
  color: #fff;
  padding: 6px 12px;
  border-radius: 16px;
  font-size: 12px;
  cursor: pointer;
  border: 1px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(4px);
  text-align: center;
  transition: all 0.3s ease;
}

.camera-toggle:hover {
  background: rgba(0, 0, 0, 0.8);
}

/* 主内容区 */
.main-content {
  position: relative;
  z-index: 10;
  flex: 1;
  display: flex;
  justify-content: flex-end; /* 靠右对齐，留出左边的人物 */
  padding: 20px;
  overflow: hidden;
}

/* 聊天面板 */
.chat-container {
  width: 100%;
  max-width: 380px; /* 限制最大宽度，右侧悬浮 */
  /* background: rgba(20, 20, 20, 0.75); */
  /* backdrop-filter: blur(16px); */
  border-radius: 24px;
  display: flex;
  flex-direction: column;
  height: 100%;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  margin-left: auto; /* 靠右 */
}

@media (max-width: 768px) {
  .chat-container {
    width: 100%;
    max-width: 100%;
    /* background: rgba(20, 20, 20, 0.85); */
  }
  .main-content {
    padding: 15px;
  }
}

.chat-header {
  text-align: center;
  padding: 16px;
  font-size: 16px;
  font-weight: 600;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  scroll-behavior: smooth;
}

/* 隐藏滚动条 */
.message-list::-webkit-scrollbar {
  display: none;
}

.message-wrapper {
  display: flex;
  width: 100%;
}

.message-wrapper.is-user {
  justify-content: flex-end;
}

.message-wrapper.is-ai {
  justify-content: flex-start;
}

.message-bubble {
  max-width: 85%;
  padding: 12px 16px;
  border-radius: 18px;
  font-size: 14px;
  line-height: 1.5;
  word-wrap: break-word;
}

.is-user .message-bubble {
  background-color: #007AFF;
  color: white;
  border-bottom-right-radius: 4px;
}

.is-ai .message-bubble {
  background-color: rgba(255, 255, 255, 0.15);
  color: #E0E0E0;
  border-top-left-radius: 4px;
}

/* 语音状态区 */
.voice-status-area {
  padding: 15px 20px;
  text-align: center;
  min-height: 100px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.wave-container {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 3px;
  height: 30px;
  margin-bottom: 12px;
}

.wave-bar {
  width: 3px;
  height: 10px;
  background-color: #007AFF;
  border-radius: 2px;
  animation: wave 1s infinite ease-in-out;
}

@keyframes wave {
  0%, 100% { height: 10px; }
  50% { height: 24px; }
}

.status-text {
  font-size: 13px;
  color: #999;
  margin-bottom: 6px;
}

.asr-text {
  font-size: 15px;
  color: #fff;
  font-weight: 500;
}

/* 底部控制区 */
.controls-area {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 40px 30px;
}

.side-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  color: #999;
  font-size: 12px;
  cursor: pointer;
}

.side-btn .icon {
  font-size: 22px;
}

.mic-btn {
  width: 64px;
  height: 64px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  border: 2px solid rgba(255, 255, 255, 0.2);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
  outline: none;
}

.mic-btn.active {
  background: #007AFF;
  border-color: #007AFF;
  box-shadow: 0 0 20px rgba(0, 122, 255, 0.4);
}

.mic-icon {
  font-size: 28px;
}

/* 底部导航 */
.bottom-nav {
  position: relative;
  z-index: 10;
  display: flex;
  justify-content: space-around;
  padding: 12px 0 24px;
  background: #0a0a0a;
  border-top: 1px solid rgba(255, 255, 255, 0.05);
}

.nav-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  color: #666;
  font-size: 10px;
  cursor: pointer;
}

.nav-item.active {
  color: #007AFF;
}

.nav-icon {
  font-size: 20px;
}
</style>
