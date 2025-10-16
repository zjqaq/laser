<template>
  <div class="page-container min-h-screen bg-gradient-to-b from-gray-50 to-gray-100 flex flex-col items-center p-2 md:p-4 lg:p-6">
    <!-- 标题 -->
    <h1 class="page-title mb-4 mt-1 tracking-tight">激光队列编排系统</h1>

    <!-- 新增的设置和校准按钮 -->
    <div class="settings-calibration-buttons mb-4 w-full max-w-4xl flex gap-2">
      <button @click="showSettingsDialog = true" class="settings-btn">
        设置
      </button>
      <button @click="showCalibrationDialog = true" class="calibration-btn">
        校准
      </button>
    </div>

    <!-- 预设阵型按钮组 -->
    <div class="preset-btn-group mb-4 w-full max-w-4xl">
      <button
        @click="selectShape('rect')"
        :class="['preset-btn', activeShape === 'rect' ? 'active' : '']"
      >
        矩形阵
      </button>
      <button
        @click="selectShape('line')"
        :class="['preset-btn', activeShape === 'line' ? 'active' : '']"
      >
        一字阵
      </button>
      <button
        @click="selectShape('cross')"
        :class="['preset-btn', activeShape === 'cross' ? 'active' : '']"
      >
        十字阵
      </button>
      <button
        @click="selectShape('circle')"
        :class="['preset-btn', activeShape === 'circle' ? 'active' : '']"
      >
        圆形阵
      </button>
    </div>

    <!-- 自定义画布区域 -->
    <div class="canvas-wrapper w-full max-w-4xl mb-1 flex-1">
      <p class="canvas-title">自定义阵型（点击画布添加/删除点，最多8个）</p>
      <div class="canvas-container">
        <canvas
          ref="canvasRef"
          @click="handleCanvasClick"
          class="cursor-pointer transition-all duration-200 hover:shadow-inner"
          :height="canvasHeight"
        ></canvas>
      </div>
    </div>

    <!-- 变换操作按钮组 -->
    <div class="transform-btn-group mt-2 mb-2 w-full max-w-4xl">
      <button
        @click="showTranslateDialog = true"
        class="transform-btn translate-btn"
        :disabled="!hasPoints"
      >
        平移
      </button>
      <button
        @click="showRotateDialog = true"
        class="transform-btn rotate-btn"
        :disabled="!hasPoints"
      >
        旋转
      </button>
    </div>

    <!-- 操作按钮组 -->
    <div class="action-btn-group mt-0 mb-3 w-full max-w-4xl">
      <button
        @click="clearShape"
        class="action-btn clear-btn"
      >
        清除阵型
      </button>
      <button
        @click="confirmCustom"
        class="action-btn confirm-btn"
      >
        确认阵型
      </button>
    </div>

    <!-- 平移弹窗 -->
    <div v-if="showTranslateDialog" class="dialog-overlay">
      <div class="dialog">
        <h3 class="dialog-title">平移设置</h3>
        <div class="dialog-content">
          <div class="translate-controls">
            <div class="direction-group">
              <label class="direction-label">方向：</label>
              <div class="direction-buttons">
                <button @click="translateDirection = 'up'" :class="translateDirection === 'up' ? 'active' : ''">上</button>
                <button @click="translateDirection = 'down'" :class="translateDirection === 'down' ? 'active' : ''">下</button>
                <button @click="translateDirection = 'left'" :class="translateDirection === 'left' ? 'active' : ''">左</button>
                <button @click="translateDirection = 'right'" :class="translateDirection === 'right' ? 'active' : ''">右</button>
              </div>
            </div>
            <div class="steps-control">
              <label>平移格数：</label>
              <input
                type="number"
                v-model.number="translateSteps"
                min="1"
                max="5"
                class="steps-input"
              >
            </div>
          </div>
        </div>
        <div class="dialog-buttons">
          <button @click="showTranslateDialog = false" class="dialog-btn cancel-btn">取消</button>
          <button @click="applyTranslate" class="dialog-btn confirm-btn">应用</button>
        </div>
      </div>
    </div>

    <!-- 旋转弹窗 -->
    <div v-if="showRotateDialog" class="dialog-overlay">
      <div class="dialog">
        <h3 class="dialog-title">旋转设置</h3>
        <div class="dialog-content">
          <div class="rotate-controls">
            <div class="rotation-direction">
              <label class="direction-label">方向：</label>
              <div class="direction-buttons">
                <button @click="rotateDirection = 'clockwise'" :class="rotateDirection === 'clockwise' ? 'active' : ''">顺时针</button>
                <button @click="rotateDirection = 'counterclockwise'" :class="rotateDirection === 'counterclockwise' ? 'active' : ''">逆时针</button>
              </div>
            </div>
            <div class="angle-control">
              <label>旋转角度：</label>
              <select v-model="rotateAngle" class="angle-select">
                <option value="90">90°</option>
                <option value="180">180°</option>
                <option value="270">270°</option>
              </select>
            </div>
          </div>
        </div>
        <div class="dialog-buttons">
          <button @click="showRotateDialog = false" class="dialog-btn cancel-btn">取消</button>
          <button @click="applyRotate" class="dialog-btn confirm-btn">应用</button>
        </div>
      </div>
    </div>

    <!-- 新增的设置弹窗 -->
    <div v-if="showSettingsDialog" class="dialog-overlay">
      <div class="dialog settings-dialog">
        <h3 class="dialog-title">系统设置</h3>
        <div class="dialog-content">
          <div class="settings-controls">
            <div class="setting-item">
              <label>高度：</label>
              <div class="input-with-unit">
                <input
                  type="number"
                  v-model.number="height"
                  min="1"
                  class="setting-input"
                  step="0.1"
                >
                <span class="unit">厘米</span>
              </div>
            </div>
            <div class="setting-item">
              <label>距离：</label>
              <div class="input-with-unit">
                <input
                  type="number"
                  v-model.number="distance"
                  min="1"
                  class="setting-input"
                  step="0.1"
                >
                <span class="unit">厘米</span>
              </div>
            </div>
            <div class="setting-item">
              <label>网格宽度：</label>
              <div class="input-with-unit">
                <input
                  type="number"
                  v-model.number="gridWidth"
                  min="1"
                  class="setting-input"
                  step="0.1"
                >
                <span class="unit">厘米</span>
              </div>
            </div>
          </div>
        </div>
        <div class="dialog-buttons">
          <button @click="showSettingsDialog = false" class="dialog-btn cancel-btn">取消</button>
          <button @click="saveSettings" class="dialog-btn confirm-btn">保存</button>
        </div>
      </div>
    </div>

    <!-- 校准弹窗 -->
    <div v-if="showCalibrationDialog" class="dialog-overlay">
      <div class="dialog calibration-dialog">
        <h3 class="dialog-title">校准设置</h3>
        <div class="dialog-content">
          <div class="calibration-controls">
            <div class="calibration-item">
              <label>校准值 1：</label>
              <div class="slider-with-value">
                <input
                  type="range"
                  v-model.number="calibrationValue1"
                  min="0"
                  max="180"
                  :step="currentCalibrationPrecision"
                  class="calibration-slider"
                >
                <span class="slider-value">{{ calibrationValue1 }}</span>
              </div>
            </div>
            <div class="calibration-item">
              <label>校准值 2：</label>
              <div class="slider-with-value">
                <input
                  type="range"
                  v-model.number="calibrationValue2"
                  min="0"
                  max="180"
                  :step="currentCalibrationPrecision"
                  class="calibration-slider"
                >
                <span class="slider-value">{{ calibrationValue2 }}</span>
              </div>
            </div>
            <div class="precision-control">
              <label>滑动精度：</label>
              <div class="precision-options">
                <select
                  v-model="calibrationPrecision"
                  class="precision-select"
                  @change="usePresetPrecision"
                >
                  <option value="0.1">0.1</option>
                  <option value="1">1</option>
                  <option value="5">5</option>
                  <option value="10">10</option>
                  <option value="custom">自定义</option>
                </select>

                <div v-if="calibrationPrecision === 'custom'" class="custom-precision">
                  <input
                    type="number"
                    v-model.number="customCalibrationPrecision"
                    min="0.01"
                    max="50"
                    step="0.01"
                    class="custom-precision-input"
                  >
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="dialog-buttons">
          <button @click="showCalibrationDialog = false" class="dialog-btn cancel-btn">关闭</button>
          <button @click="confirmCalibration" class="dialog-btn confirm-btn">确认</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, watch, computed } from 'vue';
import axios from 'axios';  // 引入axios用于网络请求

// 新增的设置相关响应式数据
const showSettingsDialog = ref(false);
const height = ref(100);  // 高度默认值，单位：厘米
const distance = ref(50); // 距离默认值，单位：厘米
const gridWidth = ref(10); // 网格宽度默认值，单位：厘米

// 校准相关状态
const showCalibrationDialog = ref(false);
const calibrationValue1 = ref(90); // 第一个校准值，默认90
const calibrationValue2 = ref(90); // 第二个校准值，默认90
const calibrationPrecision = ref('1'); // 校准精度，默认1
const customCalibrationPrecision = ref(0.5); // 自定义校准精度

// 计算当前使用的校准精度
const currentCalibrationPrecision = computed(() => {
  return calibrationPrecision.value === 'custom'
    ? customCalibrationPrecision.value
    : parseFloat(calibrationPrecision.value);
});

// 预设阵型的逻辑坐标（基于10×10画布，原点在左上角）
const presetShapes = {
  rect: [
    { x: 3, y: 4 },
    { x: 4, y: 4 },
    { x: 5, y: 4 },
    { x: 6, y: 4 },
    { x: 3, y: 5 },
    { x: 4, y: 5 },
    { x: 5, y: 5 },
    { x: 6, y: 5 },
  ],
  line: [
    { x: 2, y: 5 },
    { x: 3, y: 5 },
    { x: 4, y: 5 },
    { x: 5, y: 5 },
    { x: 6, y: 5 },
    { x: 7, y: 5 },
    { x: 8, y: 5 },
    { x: 9, y: 5 },
  ],
  cross: [
    // 竖方向4个点（x=5保持不变，y坐标变化）
    { x: 5, y: 3 },
    { x: 5, y: 4 },
    { x: 5, y: 7 },
    { x: 5, y: 6 },
    // 横方向4个点（y=5保持不变，x坐标变化）
    { x: 3, y: 5 },
    { x: 4, y: 5 },
    { x: 6, y: 5 },
    { x: 7, y: 5 },
  ],
  circle: [
    { x: 4, y: 3 },
    { x: 6, y: 3 },
    { x: 3, y: 4 },
    { x: 3, y: 6 },
    { x: 4, y: 7 },
    { x: 6, y: 7 },
    { x: 7, y: 4 },
    { x: 7, y: 6 },
  ],
};

// 原有响应式数据保持不变
const activeShape = ref(null); // 当前选中的预设阵型
const customPoints = ref([]); // 自定义绘制的点
const canvasRef = ref(null); // Canvas DOM 引用
const ctx = ref(null); // Canvas 2D 上下文
const canvasSize = ref({ width: 10, height: 10 }); // 画布逻辑尺寸（10×10）
const canvasHeight = ref(0); // 画布实际显示高度
const canvasLoaded = ref(false); // 画布是否已加载的标志

// 平移旋转相关状态
const showTranslateDialog = ref(false);
const showRotateDialog = ref(false);
const translateDirection = ref('right');
const translateSteps = ref(1);
const rotateDirection = ref('clockwise');
const rotateAngle = ref('90');

// 计算属性：是否有可操作的点
const hasPoints = computed(() => {
  return activeShape.value ? presetShapes[activeShape.value].length > 0 : customPoints.value.length > 0;
});

// 使用预设精度
const usePresetPrecision = () => {
  // 当选择预设精度时，设置自定义输入框的默认值
  if (calibrationPrecision.value !== 'custom') {
    customCalibrationPrecision.value = parseFloat(calibrationPrecision.value);
  }
};

// 新增的设置相关方法
const saveSettings = () => {
  // 保存设置的逻辑，可以根据需要发送到后端或本地存储
  console.log('保存设置:', {
    height: height.value,
    distance: distance.value,
    gridWidth: gridWidth.value
  });

  // 这里可以添加保存成功的提示
  alert('设置已保存');
  showSettingsDialog.value = false;

  // 如果网格宽度改变，可能需要重新初始化画布
  if (canvasLoaded.value) {
    initCanvas();
  }
};

// 校准相关方法
const confirmCalibration = () => {
  // 处理校准值的逻辑
  console.log('校准值已确认:', {
    value1: calibrationValue1.value,
    value2: calibrationValue2.value,
    precision: currentCalibrationPrecision.value
  });

  // 这里可以添加实际的校准逻辑，比如发送到后端
  alert(`校准已完成：值1=${calibrationValue1.value}, 值2=${calibrationValue2.value}, 精度=${currentCalibrationPrecision.value}`);
  showCalibrationDialog.value = false;
};

// 原有方法保持不变
const getCurrentPoints = () => {
  return activeShape.value ? [...presetShapes[activeShape.value]] : [...customPoints.value];
};

const setCurrentPoints = (points) => {
  if (activeShape.value) {
    customPoints.value = points;
    activeShape.value = null;
  } else {
    customPoints.value = points;
  }
};

onMounted(() => {
  nextTick(() => {
    checkAndInitCanvas();
    window.addEventListener('resize', handleResize);
  });
});

watch(canvasRef, (newVal) => {
  if (newVal && !canvasLoaded.value) {
    checkAndInitCanvas();
  }
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

const checkAndInitCanvas = () => {
  if (canvasRef.value) {
    initCanvas();
  } else {
    let retries = 0;
    const interval = setInterval(() => {
      if (canvasRef.value || retries >= 15) {
        clearInterval(interval);
        if (canvasRef.value) {
          initCanvas();
        }
      }
      retries++;
    }, 100);
  }
};

const initCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;

  const container = canvas.parentElement;
  if (!container) return;

  container.getBoundingClientRect();
  let containerWidth = Math.min(
    container.clientWidth,
    window.innerWidth - 40
  );

  if (containerWidth <= 0) {
    containerWidth = 300;
  }

  canvas.width = containerWidth;
  canvas.height = containerWidth;
  canvasHeight.value = containerWidth;
  ctx.value = canvas.getContext('2d');

  drawCanvas();
  canvasLoaded.value = true;

  if (containerWidth === 300) {
    const maxRetries = 20;
    let retries = 0;

    const retryInit = () => {
      if (retries >= maxRetries) return;
      retries++;
      container.getBoundingClientRect();
      const newWidth = Math.min(
        container.clientWidth,
        window.innerWidth - 40
      );
      if (newWidth > 0 && newWidth !== containerWidth) {
        canvas.width = newWidth;
        canvas.height = newWidth;
        canvasHeight.value = newWidth;
        ctx.value = canvas.getContext('2d');
        drawCanvas();
      } else {
        setTimeout(retryInit, 100);
      }
    };

    retryInit();
  }
};

const handleResize = () => {
  if (canvasLoaded.value) {
    initCanvas();
  }
};

const drawCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  ctx.value.clearRect(0, 0, canvas.width, canvas.height);
  drawGrid();

  if (activeShape.value) {
    drawPoints(presetShapes[activeShape.value]);
  } else {
    drawPoints(customPoints.value);
  }
};

const drawGrid = () => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  ctx.value.fillStyle = '#f9fafb';
  ctx.value.fillRect(0, 0, canvas.width, canvas.height);

  ctx.value.strokeStyle = '#e2e8f0';
  ctx.value.lineWidth = 1;

  const cellWidth = canvas.width / canvasSize.value.width;
  const cellHeight = canvas.height / canvasSize.value.height;

  // 画竖线
  for (let x = 0; x <= canvasSize.value.width; x++) {
    ctx.value.beginPath();
    ctx.value.moveTo(x * cellWidth, 0);
    ctx.value.lineTo(x * cellWidth, canvas.height);
    ctx.value.stroke();
  }

  // 画横线
  for (let y = 0; y <= canvasSize.value.height; y++) {
    ctx.value.beginPath();
    ctx.value.moveTo(0, y * cellHeight);
    ctx.value.lineTo(canvas.width, y * cellHeight);
    ctx.value.stroke();
  }

  // 绘制中心参考线
  ctx.value.strokeStyle = '#94a3b8';
  ctx.value.lineWidth = 2;
  ctx.value.beginPath();
  ctx.value.moveTo(canvas.width / 2, 0);
  ctx.value.lineTo(canvas.width / 2, canvas.height);
  ctx.value.moveTo(0, canvas.height / 2);
  ctx.value.lineTo(canvas.width, canvas.height / 2);
  ctx.value.stroke();
};

const drawPoints = (points) => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value || !points.length) return;

  const cellWidth = canvas.width / canvasSize.value.width;
  const cellHeight = canvas.height / canvasSize.value.height;

  points.forEach(({ x, y }) => {
    const px = x * cellWidth;
    const py = y * cellHeight;
    const radius = Math.min(cellWidth, cellHeight) / 2.5;

    ctx.value.shadowColor = 'rgba(220, 38, 38, 0.3)';
    ctx.value.shadowBlur = 5;
    ctx.value.shadowOffsetX = 0;
    ctx.value.shadowOffsetY = 0;

    ctx.value.fillStyle = 'white';
    ctx.value.beginPath();
    ctx.value.arc(px, py, radius, 0, Math.PI * 2);
    ctx.value.fill();

    ctx.value.fillStyle = '#dc2626';
    ctx.value.beginPath();
    ctx.value.arc(px, py, radius * 0.7, 0, Math.PI * 2);
    ctx.value.fill();

    ctx.value.shadowColor = 'transparent';
  });
};

const selectShape = (shape) => {
  activeShape.value = shape;
  customPoints.value = [];
  drawCanvas();
};

const clearShape = () => {
  activeShape.value = null;
  customPoints.value = [];
  drawCanvas();
};

const handleCanvasClick = (e) => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  activeShape.value = null;

  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  const cellWidth = canvas.width / canvasSize.value.width;
  const cellHeight = canvas.height / canvasSize.value.height;

  const logicX = Math.round(x / cellWidth);
  const logicY = Math.round(y / cellHeight);

  const pointIndex = customPoints.value.findIndex(
    (p) => p.x === logicX && p.y === logicY
  );

  if (pointIndex !== -1) {
    customPoints.value.splice(pointIndex, 1);
  } else if (customPoints.value.length < 8) {
    customPoints.value.push({ x: logicX, y: logicY });
  }

  drawCanvas();
};

const confirmCustom = async () => {
  const points = activeShape.value ? presetShapes[activeShape.value] : customPoints.value;

  if (points.length === 0) {
    alert('请绘制至少一个点后再确认～');
    return;
  }

  try {
    alert('正在发送数据到后端...');

    const response = await axios.post('http://localhost:5000/api/send-pattern', {
      points: points
    });

    if (response.data.success) {
      alert('数据已成功发送到后端！');
      console.log('发送成功，后端响应：', response.data);
    } else {
      alert(`发送失败：${response.data.message}`);
    }
  } catch (error) {
    console.error('发送数据时发生错误：', error);
    alert('发送数据失败，请检查后端服务是否正常运行');
  }
};

const applyTranslate = () => {
  const points = getCurrentPoints();
  if (!points.length) return;

  let newPoints = [];

  switch (translateDirection.value) {
    case 'up':
      newPoints = points.map(point => ({
        x: point.x,
        y: Math.max(0, point.y - translateSteps.value)
      }));
      break;
    case 'down':
      newPoints = points.map(point => ({
        x: point.x,
        y: Math.min(canvasSize.value.height - 1, point.y + translateSteps.value)
      }));
      break;
    case 'left':
      newPoints = points.map(point => ({
        x: Math.max(0, point.x - translateSteps.value),
        y: point.y
      }));
      break;
    case 'right':
      newPoints = points.map(point => ({
        x: Math.min(canvasSize.value.width - 1, point.x + translateSteps.value),
        y: point.y
      }));
      break;
  }

  setCurrentPoints(newPoints);
  showTranslateDialog.value = false;
  drawCanvas();
};

const applyRotate = () => {
  const points = getCurrentPoints();
  if (!points.length) return;

  const centerX = canvasSize.value.width / 2;
  const centerY = canvasSize.value.height / 2;

  let angle = (parseInt(rotateAngle.value) * Math.PI) / 180;
  if (rotateDirection.value === 'counterclockwise') {
    angle = -angle;
  }

  const newPoints = points.map(point => {
    const x = point.x - centerX;
    const y = point.y - centerY;

    const rotatedX = x * Math.cos(angle) - y * Math.sin(angle);
    const rotatedY = x * Math.sin(angle) + y * Math.cos(angle);

    return {
      x: Math.round(rotatedX + centerX),
      y: Math.round(rotatedY + centerY)
    };
  });

  const clampedPoints = newPoints.map(point => ({
    x: Math.max(0, Math.min(canvasSize.value.width - 1, point.x)),
    y: Math.max(0, Math.min(canvasSize.value.height - 1, point.y))
  }));

  setCurrentPoints(clampedPoints);
  showRotateDialog.value = false;
  drawCanvas();
};
</script>

<style scoped>
canvas {
  touch-action: none;
  width: 100% !important;
  height: 100% !important;
  display: block;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  overflow-x: hidden;
  width: 100%;
  height: 100%;
}

::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}
::-webkit-scrollbar-track {
  background: #f1f5f9;
  border-radius: 4px;
}
::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 4px;
  transition: background 0.3s ease;
}
::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

.page-container {
  min-height: 50vh;
  width: 100%;
  margin: 0 auto;
  max-width: 1400px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

/* 新增的设置和校准按钮样式 */
.settings-calibration-buttons {
  display: flex;
  gap: 0.8rem;
  width: 100%;
  justify-content: center;
  padding: 0 0.25rem;
}

.settings-btn, .calibration-btn {
  padding: 0.6rem 1.2rem;
  border-radius: 0.6rem;
  font-size: clamp(0.9rem, 3vw, 1.1rem);
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
  white-space: nowrap;
}

.settings-btn {
  background: linear-gradient(135deg, #fbbf24, #f59e0b);
  color: #ffffff;
}

.settings-btn:hover {
  background: linear-gradient(135deg, #fcd34d, #d97706);
  box-shadow: 0 8px 16px -1px rgba(245, 158, 11, 0.25);
  transform: translateY(-2px) scale(1.03);
}

.calibration-btn {
  background: linear-gradient(135deg, #ec4899, #db2777);
  color: #ffffff;
}

.calibration-btn:hover {
  background: linear-gradient(135deg, #f472b6, #be185d);
  box-shadow: 0 8px 16px -1px rgba(236, 72, 153, 0.25);
  transform: translateY(-2px) scale(1.03);
}

.preset-btn-group {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  width: 100%;
  margin: 0 auto;
  padding: 0 0.25rem;
}
.preset-btn {
  flex: 1;
  min-width: calc(50% - 0.5rem);
  padding: 0.6rem 0.5rem;
  border-radius: 0.6rem;
  font-size: clamp(0.9rem, 3vw, 1.1rem);
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -1px rgba(0, 0, 0, 0.03);
  white-space: nowrap;
}

@media (min-width: 768px) {
  .preset-btn {
    min-width: auto;
    flex: 1;
  }
}

.preset-btn:not(.active) {
  background: #ffffff;
  color: #334155;
  border: 1px solid #e2e8f0;
}
.preset-btn:not(.active):hover {
  background: #f8fafc;
  box-shadow: 0 6px 8px -1px rgba(0, 0, 0, 0.08), 0 3px 5px -1px rgba(0, 0, 0, 0.04);
  transform: translateY(-2px) scale(1.02);
}
.preset-btn.active {
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: #ffffff;
  box-shadow: 0 8px 16px -1px rgba(37, 99, 235, 0.25), 0 4px 8px -1px rgba(37, 99, 235, 0.15);
  transform: translateY(-1px);
}
.preset-btn.active:hover {
  background: linear-gradient(135deg, #1d4ed8, #2563eb);
  box-shadow: 0 10px 20px -1px rgba(37, 99, 235, 0.3), 0 5px 10px -1px rgba(37, 99, 235, 0.2);
}

.canvas-wrapper {
  width: 100%;
  max-width: 100%;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
  margin-bottom: 0 !important;
  padding: 0 0.25rem;
}
.canvas-title {
  font-size: clamp(0.9rem, 2vw, 1.1rem);
  color: #475569;
  font-weight: 500;
  text-align: center;
  margin: 0;
}
.canvas-container {
  width: 100%;
  padding-top: 100%;
  position: relative;
  background: #ffffff;
  border-radius: 0.8rem;
  box-shadow: 0 8px 16px -2px rgba(0, 0, 0, 0.08), 0 4px 8px -1px rgba(0, 0, 0, 0.04);
  border: 1px solid #f1f5f9;
  overflow: hidden;
  transition: all 0.3s ease;
  max-height: 60vh;
}

@media (min-width: 768px) {
  .canvas-container {
    max-height: 70vh;
  }
}

.canvas-container canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100% !important;
  height: 100% !important;
}
.canvas-container:hover {
  box-shadow: 0 12px 24px -2px rgba(0, 0, 0, 0.12), 0 6px 12px -1px rgba(0, 0, 0, 0.06);
  padding-bottom: 0 !important;
}
canvas:hover {
  cursor: crosshair;
  background-color: #fefeff;
}

.action-btn-group {
  display: flex;
  gap: 0.8rem;
  width: 100%;
  justify-content: center;
  padding-top: 0 !important;
  margin-top: 0 !important;

}
.action-btn {
  flex: 1;
  min-width: 100px;
  padding: 0.6rem 0.8rem;
  border-radius: 0.6rem;
  font-size: clamp(0.9rem, 3vw, 1.1rem);
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
  white-space: nowrap;
}
.clear-btn {
  background: linear-gradient(135deg, #f87171, #ef4444);
  color: #ffffff;
}
.clear-btn:hover {
  background: linear-gradient(135deg, #fca5a5, #dc2626);
  box-shadow: 0 8px 16px -1px rgba(239, 68, 68, 0.25);
  transform: translateY(-2px) scale(1.03);
}
.confirm-btn {
  background: linear-gradient(135deg, #4ade80, #22c55e);
  color: #ffffff;
}
.confirm-btn:hover {
  background: linear-gradient(135deg, #86efac, #16a34a);
  box-shadow: 0 8px 16px -1px rgba(34, 197, 94, 0.25);
  transform: translateY(-2px) scale(1.03);
}

.page-title {
  font-size: clamp(1.5rem, 6vw, 2.5rem);
  font-weight: 700;
  color: #2563eb;
  text-align: center;
  margin: 0;
  letter-spacing: -0.02em;
  text-shadow: 0 2px 4px rgba(37, 99, 235, 0.1);
}

@media (max-width: 360px) {
  .action-btn {
    min-width: auto;
    padding: 0.5rem 0.6rem;
    font-size: 0.85rem;
  }

  .page-container {
    gap: 0.6rem;
  }
}

.transform-btn-group {
  display: flex;
  gap: 0.8rem;
  width: 100%;
  justify-content: center;
  padding: 0 0.25rem;
}

.transform-btn {
  flex: 1;
  min-width: 100px;
  padding: 0.6rem 0.8rem;
  border-radius: 0.6rem;
  font-size: clamp(0.9rem, 3vw, 1.1rem);
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
  white-space: nowrap;
}

.transform-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.translate-btn {
  background: linear-gradient(135deg, #60a5fa, #3b82f6);
  color: #ffffff;
}

.translate-btn:not(:disabled):hover {
  background: linear-gradient(135deg, #93c5fd, #2563eb);
  box-shadow: 0 8px 16px -1px rgba(59, 130, 246, 0.25);
  transform: translateY(-2px) scale(1.03);
}

.rotate-btn {
  background: linear-gradient(135deg, #a78bfa, #7c3aed);
  color: #ffffff;
}

.rotate-btn:not(:disabled):hover {
  background: linear-gradient(135deg, #c4b5fd, #6d28d9);
  box-shadow: 0 8px 16px -1px rgba(124, 58, 237, 0.25);
  transform: translateY(-2px) scale(1.03);
}

/* 弹窗样式 */
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 1rem;
}

.dialog {
  background-color: white;
  border-radius: 0.8rem;
  width: 100%;
  max-width: 400px;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1), 0 8px 10px -6px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.dialog-title {
  font-size: 1.2rem;
  font-weight: 600;
  color: #1e293b;
  padding: 1rem;
  border-bottom: 1px solid #e2e8f0;
  text-align: center;
}

.dialog-content {
  padding: 1.5rem;
}

.dialog-buttons {
  display: flex;
  gap: 0.5rem;
  padding: 1rem;
  border-top: 1px solid #e2e8f0;
}

.dialog-btn {
  flex: 1;
  padding: 0.6rem;
  border-radius: 0.4rem;
  font-weight: 600;
  transition: all 0.2s ease;
  border: none;
  cursor: pointer;
}

.cancel-btn {
  background-color: #f1f5f9;
  color: #475569;
}

.cancel-btn:hover {
  background-color: #e2e8f0;
}

/* 平移控制样式 */
.translate-controls, .rotate-controls, .settings-controls, .calibration-controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.direction-group, .rotation-direction {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.direction-label {
  font-weight: 500;
  color: #334155;
}

.direction-buttons {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.direction-buttons button {
  flex: 1;
  padding: 0.5rem;
  border-radius: 0.4rem;
  border: 1px solid #e2e8f0;
  background-color: #f8fafc;
  cursor: pointer;
  transition: all 0.2s ease;
}

.direction-buttons button.active {
  background-color: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.steps-control, .angle-control, .setting-item, .precision-control {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.steps-control label, .angle-control label, .setting-item label, .precision-control label {
  font-weight: 500;
  color: #334155;
  min-width: 80px;
}

.steps-input, .angle-select, .setting-input, .precision-select {
  flex: 1;
  padding: 0.5rem;
  border-radius: 0.4rem;
  border: 1px solid #e2e8f0;
}

.angle-select, .precision-select {
  background-color: white;
}

/* 新增的设置弹窗样式 */
.settings-dialog .dialog-content {
  padding: 1.5rem;
}

.setting-item {
  margin-bottom: 1rem;
}

.setting-item:last-child {
  margin-bottom: 0;
}

.input-with-unit {
  display: flex;
  align-items: center;
  flex: 1;
}

.setting-input {
  flex: 1;
  padding-right: 50px; /* 为单位预留空间 */
}

.unit {
  position: relative;
  right: 40px;
  color: #64748b;
  font-size: 0.9rem;
}

/* 校准弹窗样式 */
.calibration-dialog .dialog-content {
  padding: 1.5rem;
}

.calibration-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.calibration-item label {
  font-weight: 500;
  color: #334155;
}

.slider-with-value {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.calibration-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
}

.calibration-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  transition: all 0.2s ease;
}

.calibration-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
  transform: scale(1.1);
}

.slider-value {
  min-width: 40px;
  text-align: center;
  font-weight: 500;
  color: #3b82f6;
}

/* 精度选择相关样式 */
.precision-options {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex: 1;
}

.custom-precision {
  flex: 1;
}

.custom-precision-input {
  width: 100%;
  padding: 0.5rem;
  border-radius: 0.4rem;
  border: 1px solid #e2e8f0;
}
</style>