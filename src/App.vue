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
      <div id="local-player"></div>
    </div>

    <!-- 摄像头控制按钮 (悬浮在右上角摄像头下方) -->
    <div class="camera-controls" v-if="isConnected">
      <div class="camera-toggle" @click="toggleCamera">
        {{ isCameraOn ? '关闭摄像头' : '开启摄像头' }}
      </div>
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
            <span v-else-if="isAiSpeaking">AI正在回复...</span>
            <span v-else>等待中...</span>
          </div>

          <!-- 打断提示 -->
          <div class="interrupt-message" v-if="interruptedMessage">
            {{ interruptedMessage }}
          </div>
          
          <div class="asr-text" v-if="currentAsrText">
            “{{ currentAsrText }}”
          </div>
        </div>

        <!-- 底部控制按钮 -->
        <div class="controls-area">
          <div class="side-btn" @click="showSettings = !showSettings">
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
      
      <!-- 设置界面 -->
      <div class="settings-panel" v-if="showSettings">
        <div class="settings-content">
          <div class="settings-header">
            <h3>关键词打断设置</h3>
            <button class="close-btn" @click="showSettings = false">×</button>
          </div>
          
          <div class="keyword-settings">
            <h4>当前关键词列表</h4>
            <div class="keyword-list">
              <div
                v-for="(keyword, index) in interruptKeywords"
                :key="index"
                class="keyword-item"
              >
                <span>{{ keyword }}</span>
                <button class="remove-keyword" @click="removeKeyword(index)">×</button>
              </div>
            </div>
            
            <div class="add-keyword-section">
              <input
                v-model="newKeyword"
                placeholder="输入新关键词"
                @keyup.enter="addKeyword"
              />
              <button @click="addKeyword">添加</button>
            </div>
            
            <div class="instructions">
              <p><strong>说明：</strong></p>
              <p>• <strong>方案二（客户端关键词检测）</strong>：在ASR识别过程中实时检测关键词</p>
              <p>• 推荐使用2-4个汉字的短语，如"大阳山助手"、"停止"等</p>
              <p>• 当AI正在回复时，如果您说出包含关键词的语音，AI会立即停止</p>
              <p>• <em>注意：此方案依赖ASR识别准确率，可能有延迟</em></p>
            </div>
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
const isCameraOn = ref(false);
const showSettings = ref(false);
const currentAsrText = ref('');
const messages = ref<Message[]>([]);
const messageListRef = ref<HTMLElement | null>(null);
const interruptKeywords = ref(['大阳山助手', '你好助手', '助手']);
const newKeyword = ref('');
const isAiSpeaking = ref(false); // AI是否正在说话
const interruptedMessage = ref(''); // 被打断的消息提示
const isAiThinking = ref(false); // 是否正在等待 AI 开始回复，用于锁定用户输入

let client: RealtimeClient | null = null;
let currentAiMessageId = '';

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

const lockUserInput = () => {
  isAiThinking.value = true;
  currentAsrText.value = '';
  client?.setAudioEnable(false).catch(e => console.error("静音失败:", e));
};

const unlockUserInput = () => {
  if (!isAiThinking.value) return;
  isAiThinking.value = false;
  client?.setAudioEnable(true).catch(e => console.error("恢复麦克风失败:", e));
};

const getEventRole = (eventData: any) => {
  return eventData?.role || eventData?.data?.role || eventData?.message?.role || eventData?.data?.message?.role || '';
};

const initClient = async () => {
  try {
    const permission = await RealtimeUtils.checkDevicePermission(true);
    // 这里我们直接用浏览器原生API请求摄像头，因为 sdk 内部配置好像变了
    if (!permission.audio) throw new Error('需要麦克风访问权限');
    
    try {
      // 不自动获取摄像头,改为用户手动开启
      // localMediaStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
      console.log('摄像头已准备就绪,等待用户开启');
    } catch(err) {
      console.warn('获取摄像头失败:', err);
    }

    client = new RealtimeClient({
      accessToken: config.accessToken,
      botId: config.botId,
      connectorId: config.connectorId,
      allowPersonalAccessTokenInBrowser: true,
      debug: true,
      videoConfig: {
        renderDom: 'local-player',
        videoOnDefault: false,
      },
      
      // --- AI 降噪与语音增强 ---
      suppressStationaryNoise: true,     // 开启静态噪声抑制（过滤空调、风扇等持续噪音）
      suppressNonStationaryNoise: true,  // 开启非静态噪声抑制（过滤键盘、咳嗽等突发噪音）
    });

    client.on(EventNames.ALL, (eventName, data: any) => {
      // 恢复全部日志输出，方便排查所有接收到的事件
      console.log(`事件: ${eventName}`, data);

      if (eventName === EventNames.CONNECTED) {
        isConnected.value = true;

        // 发送事件配置关键词打断
        // 根据文档，WebSocket 场景用 chat.update，RTC 音视频场景用 session.update
        // 由于我们这里使用了摄像头，可能被识别为音视频场景，所以先尝试 session.update
        setTimeout(async () => {
          try {
            // 首先尝试 session.update (适用于 RTC 音视频场景)
            const sessionUpdateResult = await client?.sendMessage({
              "id": `event_${Date.now()}`,
              "event_type": "session.update",
              "data": {
                "chat_config": {
                  "allow_voice_interrupt": true,
                  "interrupt_config": {
                    "mode": "keyword_contains",
                    "keywords": ["大阳山助手", "你好助手", "助手"]
                  }
                }
              }
            });
            console.log("已发送 session.update 配置关键词打断", sessionUpdateResult);
          } catch(e) {
            console.error("配置 session.update 打断词失败:", e);
            
            // 如果 session.update 失败，则尝试 chat.update (适用于 WebSocket 语音通话场景)
            try {
              const chatUpdateResult = await client?.sendMessage({
                "id": `event_${Date.now() + 1}`,
                "event_type": "chat.update",
                "data": {
                  "turn_detection": {
                    "type": "server_vad",
                    "interrupt_config": {
                      "mode": "keyword_contains",
                      "keywords": ["大阳山助手", "你好助手", "助手"]
                    },
                    "server_vad_config": {
                      "voice_prob_threshold": 0.8,
                      "min_voice_duration_ms": 500,
                      "max_silence_duration_ms": 3000 // 配置停顿1000ms后自动发送
                    }
                  },
                  "asr_config": {
                    "hot_words": ["大阳山助手", "你好助手", "助手"]
                  }
                }
              });
              console.log("已发送 chat.update 配置关键词打断", chatUpdateResult);
            } catch(e2) {
              console.error("配置 chat.update 打断词失败:", e2);
            }
          }
          
          // 额外发送一次配置，确保生效
          setTimeout(async () => {
            try {
              // 使用 sendMessage 发送 chat.update 事件来配置关键词打断
              const additionalConfig = await client?.sendMessage({
                "id": `event_${Date.now() + 2}`,
                "event_type": "chat.update",
                "data": {
                  "turn_detection": {
                    "type": "server_vad",
                    "interrupt_config": {
                      "mode": "keyword_contains",
                      "keywords": ["大阳山助手", "你好助手", "助手"]
                    },
                    "server_vad_config": {
                      "voice_prob_threshold": 0.8,
                      "min_voice_duration_ms": 500,
                      "max_silence_duration_ms": 3000
                    }
                  },
                  "asr_config": {
                    "hot_words": ["大阳山助手", "你好助手", "助手"]
                  }
                }
              });
              console.log("已通过 chat.update 额外配置关键词打断", additionalConfig);
            } catch(e) {
              console.error("通过 chat.update 额外配置关键词打断失败:", e);
            }
          }, 2000);
        }, 1000); // 延迟1秒发送配置，确保连接完全建立
      }
      
      if (eventName === EventNames.DISCONNECTED) {
        isConnected.value = false;
        isListening.value = false;
        isAiThinking.value = false;
        isCameraOn.value = false;
        currentAsrText.value = '';
      }

      // 用户开始说话
      if (eventName === EventNames.AUDIO_USER_SPEECH_STARTED) {
        if (isAiThinking.value) {
          console.log('正在等待AI回复，忽略本次语音输入');
          return;
        }
        isListening.value = true;
        currentAsrText.value = '';
      }

      // 用户停止说话
      if (eventName === EventNames.AUDIO_USER_SPEECH_STOPPED) {
        if (isAiThinking.value) {
          return;
        }
        isListening.value = false;
        // 把最后识别的话加到消息列表
        if (currentAsrText.value.trim()) {
          messages.value.push({
            id: `user_${Date.now()}`,
            role: 'user',
            content: currentAsrText.value
          });
          currentAsrText.value = '';
          scrollToBottom();
          lockUserInput();
        }
      }

      // 用户语音识别中间结果
      if (eventName === EventNames.CONVERSATION_AUDIO_TRANSCRIPT_DELTA) {
        if (isAiThinking.value) {
           return;
        }
        console.log("收到用户识别文本:", data);
        const asrContent = data?.content || data?.data?.content || '';
        if (asrContent) {
          // 由于 asrContent 通常是当前句子的完整识别文本，直接赋值避免重复
          currentAsrText.value = asrContent;

          // 【方案二：客户端关键词检测】
          // 在AI说话时检测关键词，如果匹配则打断
          if (isAiSpeaking.value && checkForKeywords()) {
            console.log('检测到关键词，准备打断AI回复');
            handleKeywordInterrupt();
          }
        }
      }

      // AI 开始说话
      if (eventName === EventNames.AUDIO_AGENT_SPEECH_STARTED) {
        console.log('AI开始说话');
        isAiSpeaking.value = true;
      }

      // AI 停止说话
      if (eventName === EventNames.AUDIO_AGENT_SPEECH_STOPPED) {
        console.log('AI停止说话');
        isAiSpeaking.value = false;
        unlockUserInput();
      }

      // AI 回复中间结果
      if (eventName === EventNames.CONVERSATION_MESSAGE_DELTA) {
        console.log("收到AI文本:", data); // 增加日志以便调试
        const content = data?.content || data?.data?.content || ''; // 兼容不同的数据结构
        if (!content) return;

        const message_id = data?.message_id || data?.data?.message_id || currentAiMessageId || Date.now().toString();

        // 查找是否存在对应的AI消息
        let aiMessageIndex = messages.value.findIndex(msg => msg.id === message_id && msg.role === 'assistant');

        if (aiMessageIndex !== -1) {
          // 更新现有的AI消息
          messages.value[aiMessageIndex].content += content;
        } else {
          // 创建新的AI消息
          messages.value.push({
            id: message_id,
            role: 'assistant',
            content: content
          });
          currentAiMessageId = message_id;
        }
        scrollToBottom();
      }

      // AI 回复完成
      if (eventName === EventNames.CONVERSATION_MESSAGE_COMPLETED) {
        console.log("AI文本回复完成:", data);
        const role = getEventRole(data);
        const completedMessageId = data?.message_id || data?.data?.message_id || '';
        const isAssistantCompleted = role === 'assistant' || role === 'bot' || (currentAiMessageId && completedMessageId && currentAiMessageId === completedMessageId);
        if (!isAssistantCompleted) {
          return;
        }
        currentAiMessageId = '';
        unlockUserInput();
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

const toggleCamera = async () => {
  if (!client || !isConnected.value) {
    alert('请先连接后再操作摄像头');
    return;
  }
  try {
    const nextCameraState = !isCameraOn.value;
    await client.setVideoEnable(nextCameraState);
    isCameraOn.value = nextCameraState;
  } catch (error) {
    console.error('切换摄像头失败:', error);
    alert('摄像头切换失败，请检查设备权限');
  }
};

onUnmounted(() => {
  if (client) {
    client.disconnect();
    client = null;
  }
});

// 添加关键词
const addKeyword = () => {
  if (newKeyword.value.trim() && !interruptKeywords.value.includes(newKeyword.value.trim())) {
    interruptKeywords.value.push(newKeyword.value.trim());
    newKeyword.value = '';
    
    // 如果已经连接，更新关键词配置
    if (client && isConnected.value) {
      updateInterruptKeywords();
    }
  }
};

// 删除关键词
const removeKeyword = (index: number) => {
  if (interruptKeywords.value.length <= 1) {
    alert('至少需要保留一个关键词');
    return;
  }
  
  interruptKeywords.value.splice(index, 1);
  
  // 如果已经连接，更新关键词配置
  if (client && isConnected.value) {
    updateInterruptKeywords();
  }
};

// 更新中断关键词配置
const updateInterruptKeywords = async () => {
  if (!client) return;

  try {
    // 尝试通过 session.update 更新配置
    await client.sendMessage({
      "id": `event_${Date.now()}`,
      "event_type": "session.update",
      "data": {
        "chat_config": {
          "allow_voice_interrupt": true,
          "interrupt_config": {
            "mode": "keyword_contains",
            "keywords": interruptKeywords.value
          }
        }
      }
    });
    console.log("已更新关键词配置", interruptKeywords.value);
  } catch(e) {
    console.error("更新关键词配置失败 (session.update):", e);

    // 如果 session.update 失败，尝试 chat.update
    try {
      await client.sendMessage({
        "id": `event_${Date.now() + 1}`,
        "event_type": "chat.update",
        "data": {
          "turn_detection": {
            "type": "server_vad",
            "interrupt_config": {
              "mode": "keyword_contains",
              "keywords": interruptKeywords.value
            },
            "server_vad_config": {
              "voice_prob_threshold": 0.8,
              "min_voice_duration_ms": 500,
              "max_silence_duration_ms": 3000
            }
          },
          "asr_config": {
            "hot_words": interruptKeywords.value
          }
        }
      });
      console.log("已通过 chat.update 更新关键词配置", interruptKeywords.value);
    } catch(e2) {
      console.error("更新关键词配置失败 (chat.update):", e2);
    }
  }
};

// 【方案二：客户端关键词检测】
// 检测文本中是否包含关键词
const checkForKeywords = (): boolean => {
  const currentText = currentAsrText.value.toLowerCase();
  return interruptKeywords.value.some(keyword =>
    currentText.includes(keyword.toLowerCase())
  );
};

// 【方案二：客户端关键词检测】
// 处理关键词打断
const handleKeywordInterrupt = async () => {
  if (!client) return;

  try {
    console.log('调用client.interrupt()打断AI回复');
    await client.interrupt();

    // 显示打断提示
    interruptedMessage.value = '已通过关键词打断AI回复';
    setTimeout(() => {
      interruptedMessage.value = '';
    }, 3000);

    // 清空当前识别的文本，避免再次发送或显示
    currentAsrText.value = '';
    unlockUserInput();
    scrollToBottom();
  } catch (error) {
    console.error('打断失败:', error);
  }
};
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

#local-player {
  width: 100%;
  height: 100%;
}

:deep(#local-player video) {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1); /* 镜像翻转 */
}

/* 摄像头控制按钮 */
.camera-controls {
  position: absolute;
  top: 230px; /* 在摄像头预览区下方 */
  right: 20px;
  z-index: 20;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.camera-toggle {
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
  transform: scale(1.05);
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

.interrupt-message {
  margin-top: 8px;
  padding: 8px 12px;
  background: rgba(255, 193, 7, 0.2);
  border: 1px solid rgba(255, 193, 7, 0.5);
  border-radius: 8px;
  font-size: 13px;
  color: #FFC107;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
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

/* 设置面板 */
.settings-panel {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.8);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
}

.settings-content {
  background: #222;
  border-radius: 16px;
  padding: 24px;
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  overflow-y: auto;
  position: relative;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.settings-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.settings-header h3 {
  margin: 0;
  color: #fff;
  font-size: 18px;
}

.close-btn {
  background: none;
  border: none;
  color: #fff;
  font-size: 24px;
  cursor: pointer;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background-color 0.3s;
}

.close-btn:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

.keyword-settings h4 {
  color: #fff;
  margin: 0 0 12px 0;
  font-size: 16px;
}

.keyword-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 20px;
}

.keyword-item {
  display: flex;
  align-items: center;
  background: rgba(0, 122, 255, 0.2);
  color: #fff;
  padding: 6px 12px;
  border-radius: 16px;
  font-size: 14px;
}

.remove-keyword {
  background: none;
  border: none;
  color: #fff;
  margin-left: 8px;
  cursor: pointer;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}

.remove-keyword:hover {
  background-color: rgba(255, 0, 0, 0.3);
}

.add-keyword-section {
  display: flex;
  gap: 8px;
  margin-bottom: 20px;
}

.add-keyword-section input {
  flex: 1;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(0, 0, 0, 0.3);
  color: #fff;
  font-size: 14px;
}

.add-keyword-section input:focus {
  outline: none;
  border-color: #007AFF;
}

.add-keyword-section button {
  padding: 10px 16px;
  background: #007AFF;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
}

.add-keyword-section button:hover {
  background: #0056cc;
}

.instructions {
  color: #aaa;
  font-size: 13px;
  line-height: 1.5;
}

.instructions p {
  margin: 6px 0;
}
</style>
