<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 导入心率图片资源
import lowImage from './assets/89low.png';
import midLowImage from './assets/90-99.png';
import midHighImage from './assets/100-129.png';
import highImage from './assets/130up.png';

const heartRate = ref(0);
const imageSrc = ref('');
const isConnected = ref(false);
let ws = null;
let reconnectTimeout = null;

// 根据心率值选择对应的图片
const updateImage = (rate) => {
  if (rate < 90) {
    imageSrc.value = '/images/89low.png';
  } else if (rate >= 90 && rate <= 99) {
    imageSrc.value = '/images/90-99.png';
  } else if (rate >= 100 && rate <= 129) {
    imageSrc.value = '/images/100-129.png';
  } else if (rate >= 130) {
    imageSrc.value = '/images/130up.png';
  } else {
    // 其他情况不显示任何图片
    imageSrc.value = '';
  }
};

// 连接WebSocket
const connectWebSocket = () => {
  // 连接到后端WebSocket端点
  const wsUrl = `ws://${window.location.hostname}:28040/api/ws`;
  
  try {
    ws = new WebSocket(wsUrl);
    
    ws.onopen = () => {
      console.log('WebSocket连接已建立');
      isConnected.value = true;
    };
    
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        if (data.value !== undefined) {
          heartRate.value = data.value;
          updateImage(data.value);
        }
      } catch (error) {
        console.error('解析消息数据失败:', error);
      }
    };
    
    ws.onclose = () => {
      console.log('WebSocket连接已断开');
      isConnected.value = false;
      // 清除之前的重连定时器
      if (reconnectTimeout) {
        clearTimeout(reconnectTimeout);
      }
      // 尝试重新连接
      console.log('正在尝试重新连接...');
      reconnectTimeout = setTimeout(connectWebSocket, 3000);
    };
    
    ws.onerror = (error) => {
      console.error('WebSocket错误:', error);
      isConnected.value = false;
    };
  } catch (error) {
    console.error('创建WebSocket连接失败:', error);
    // 即使创建失败也尝试重连
    if (reconnectTimeout) {
      clearTimeout(reconnectTimeout);
    }
    reconnectTimeout = setTimeout(connectWebSocket, 3000);
  }
};

onMounted(() => {
  connectWebSocket();
});

onUnmounted(() => {
  if (ws) {
    ws.close();
  }
  if (reconnectTimeout) {
    clearTimeout(reconnectTimeout);
  }
});
</script>

<template>
  <div class="container">
    <div class="heart-rate-display">
      <!-- 显示心率图片 -->
      <span></span><img v-if="imageSrc" :src="imageSrc" alt="Heart Rate Indicator" class="heart-image" />
      
      <!-- 显示心率数值 -->
      <div class="heart-rate-value" v-if="heartRate > 0">{{ heartRate }}</div>
      
      <!-- 显示连接状态 -->
      <div class="connection-status" :class="{ connected: isConnected }" v-if="!isConnected">
        状态: {{ isConnected ? '已连接' : '未连接(正在重连...)' }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 20px;
}

.heart-rate-display {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
}

.heart-image {
  width: 300px;
  height: auto;
  margin-bottom: 20px;
}

.heart-rate-value {
  font-size: 8rem;
  font-weight: bold;
  color: white; /* 白色显示心率 */
  margin-bottom: 20px;
  text-align: center;
}

.connection-status {
  font-size: 1.2rem;
  color: #ff6b6b;
  padding: 8px 16px;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.1);
}

.connection-status.connected {
  color: #4ecdc4;
}
</style>