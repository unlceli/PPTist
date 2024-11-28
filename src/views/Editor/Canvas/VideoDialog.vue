<template>
  <div class="video-dialog">
    <!-- 视频展示 -->
    <video 
      ref="videoPlayer" 
      :src="videoData.url" 
      controls 
      class="video-player"
      @timeupdate="updateCurrentTime"
    ></video>
    
    <!-- 控制按钮 -->
    <div class="controls">
      <button @click="rewind">快退</button>
      <button @click="playPause">{{ isPlaying ? '暂停' : '播放' }}</button>
      <button @click="fastForward">快进</button>
    </div>
    
    <!-- 时间输入框 -->
    <div class="time-controls">
      <label>
        开始时间：
        <input type="text" v-model="startTime" @change="updateStartFromInput" />
      </label>
      <label>
        结束时间：
        <input type="text" v-model="endTime" @change="updateEndFromInput" />
      </label>
    </div>

    <!-- 进度条剪辑 -->
    <div class="progress-bar" ref="progressBar" @click="setClipFromProgress">
      <div 
        class="progress-indicator start" 
        :style="{ left: `${(startTime / videoDuration) * 100}%` }" 
        draggable="true" 
        @drag="dragStart"
      ></div>
      <div 
        class="progress-indicator end" 
        :style="{ left: `${(endTime / videoDuration) * 100}%` }" 
        draggable="true" 
        @drag="dragEnd"
      ></div>
    </div>
     <!-- 确认与取消 -->
     <div class="action-buttons">
      <button @click="confirm">确认</button>
      <button @click="cancel">取消</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';

defineProps({
  videoData: {
    type: Object,
    required: true,
  },
});

const emit = defineEmits(['confirm', 'cancel']);

const videoPlayer = ref<HTMLVideoElement | null>(null);
const progressBar = ref<HTMLDivElement | null>(null);

const isPlaying = ref(false);
const videoDuration = ref(0);
const currentTime = ref(0);

const startTime = ref(0); // 开始时间
const endTime = ref(0); // 结束时间

// 视频加载完成后获取视频时长
watch(videoPlayer, () => {
  if (videoPlayer.value) {
    videoPlayer.value.onloadedmetadata = () => {
      videoDuration.value = videoPlayer.value?.duration || 0;
      endTime.value = videoDuration.value; // 默认结束时间为视频时长
    };
  }
});

// 播放和暂停
const playPause = () => {
  if (videoPlayer.value) {
    if (isPlaying.value) {
      videoPlayer.value.pause();
    } else {
      // 确保从 startTime 开始播放
      if (videoPlayer.value.currentTime < startTime.value || videoPlayer.value.currentTime > endTime.value) {
        videoPlayer.value.currentTime = startTime.value;
      }
      videoPlayer.value.play();
    }
    isPlaying.value = !isPlaying.value;
  }
};

// 快退功能
const rewind = () => {
  if (videoPlayer.value) {
    videoPlayer.value.currentTime = Math.max(videoPlayer.value.currentTime - 5, startTime.value);
  }
};

// 快进功能
const fastForward = () => {
  if (videoPlayer.value) {
    videoPlayer.value.currentTime = Math.min(videoPlayer.value.currentTime + 5, endTime.value);
  }
};

// 更新时间并限制播放时间
const updateCurrentTime = () => {
  if (videoPlayer.value) {
    currentTime.value = videoPlayer.value.currentTime;
    // 限制播放时间在 startTime 和 endTime 范围内
    if (currentTime.value < startTime.value) {
      videoPlayer.value.currentTime = startTime.value;
    }
    if (currentTime.value >= endTime.value) {
      videoPlayer.value.pause();
      isPlaying.value = false;
    }
  }
};

// 更新开始时间
const updateStartFromInput = () => {
  // 修正开始时间不得小于 0
  if (startTime.value < 0) {
    startTime.value = 0;
  }

  // 修正开始时间不得大于等于结束时间
  if (startTime.value >= endTime.value) {
    startTime.value = Math.max(0, endTime.value - 1); // 确保与结束时间至少相差 1 秒
  }

  // 如果当前播放时间小于新开始时间，调整视频播放位置
  if (videoPlayer.value && videoPlayer.value.currentTime < startTime.value) {
    videoPlayer.value.currentTime = startTime.value;
  }
};

// 更新结束时间
const updateEndFromInput = () => {
  // 修正结束时间不得小于等于开始时间
  if (endTime.value <= startTime.value) {
    endTime.value = Math.min(videoDuration.value, startTime.value + 1); // 确保与开始时间至少相差 1 秒
  }

  // 修正结束时间不得超出视频总时长
  if (endTime.value > videoDuration.value) {
    endTime.value = videoDuration.value;
  }

  // 如果当前播放时间超出新结束时间，暂停播放
  if (videoPlayer.value && videoPlayer.value.currentTime >= endTime.value) {
    videoPlayer.value.pause();
    isPlaying.value = false;
  }
};


// 进度条拖动剪辑开始时间
const dragStart = (e: DragEvent) => {
  if (!progressBar.value || !videoDuration.value) return;
  const rect = progressBar.value.getBoundingClientRect();
  const offsetX = e.clientX - rect.left;
  startTime.value = Math.max(0, (offsetX / rect.width) * videoDuration.value);
};

// 进度条拖动剪辑结束时间
const dragEnd = (e: DragEvent) => {
  if (!progressBar.value || !videoDuration.value) return;
  const rect = progressBar.value.getBoundingClientRect();
  const offsetX = e.clientX - rect.left;
  endTime.value = Math.min(videoDuration.value, (offsetX / rect.width) * videoDuration.value);
};

// 点击进度条设置剪辑范围
const setClipFromProgress = (e: MouseEvent) => {
  if (!progressBar.value || !videoDuration.value) return;
  const rect = progressBar.value.getBoundingClientRect();
  const offsetX = e.clientX - rect.left;
  const time = (offsetX / rect.width) * videoDuration.value;

  // 判断靠近开始时间还是结束时间
  if (Math.abs(time - startTime.value) < Math.abs(time - endTime.value)) {
    startTime.value = Math.max(0, time);
  } else {
    endTime.value = Math.min(videoDuration.value, time);
  }
};


const confirm = () => {
  // 确认当前的截取时间段
  console.log(`确认截取时间段: 开始时间=${startTime.value}, 结束时间=${endTime.value}`);
  // 可通过 emit 或其他方法将数据传递给父组件
  emit('confirm', { startTime: startTime.value, endTime: endTime.value });
};

const cancel = () => {
  // 取消时恢复默认状态（也可直接关闭对话框）
  startTime.value = 0;
  endTime.value = videoDuration.value;
  videoPlayer.value && (videoPlayer.value.currentTime = 0);
  emit('cancel');
};





</script>

<style scoped>
.video-dialog {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.video-player {
  width: 100%;
  max-height: 400px;
}

.controls {
  display: flex;
  justify-content: space-around;
}

.time-controls {
  display: flex;
  justify-content: space-between;
}

.progress-bar {
  position: relative;
  height: 10px;
  background-color: #ddd;
  border-radius: 5px;
  overflow: hidden;
  cursor: pointer;
}

.progress-indicator {
  position: absolute;
  top: 0;
  width: 10px;
  height: 100%;
  background-color: #007BFF;
  cursor: grab;
}

.progress-indicator.start {
  background-color: #007BFF;
}

.progress-indicator.end {
  background-color: #FF5733;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}
</style>
