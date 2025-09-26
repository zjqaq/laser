<template>
  <div class="page-container min-h-screen bg-gradient-to-b from-gray-50 to-gray-100 flex flex-col items-center p-2 md:p-4 lg:p-6">
    <!-- 标题 -->
    <h1 class="page-title mb-4 mt-1 tracking-tight">激光队列编排系统</h1>

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

    <!-- 自定义画布区域 - 减小底部边距 -->
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

    <!-- 操作按钮组 - 减小顶部边距 -->
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
  </div>
</template>

<script setup>
// 脚本部分保持不变，与原代码一致
import { ref, onMounted, onUnmounted, nextTick, watch } from 'vue';

// 预设阵型的逻辑坐标（基于16×16画布，原点在左上角）
const presetShapes = {
  rect: [
    { x: 5, y: 7 },
    { x: 7, y: 7 },
    { x: 9, y: 7 },
    { x: 11, y: 7 },
    { x: 5, y: 9 },
    { x: 7, y: 9 },
    { x: 9, y: 9 },
    { x: 11, y: 9 },
  ],
  line: [
    { x: 8, y: 8 },
    { x: 9, y: 8 },
    { x: 10, y: 8 },
    { x: 11, y: 8 },
    { x: 4, y: 8 },
    { x: 5, y: 8 },
    { x: 6, y: 8 },
    { x: 7, y: 8 },
  ],
  cross: [
    { x: 8, y: 4 },
    { x: 8, y: 6 },
    { x: 8, y: 10 },
    { x: 8, y: 12 },
    { x: 4, y: 8 },
    { x: 6, y: 8 },
    { x: 10, y: 8 },
    { x: 12, y: 8 },
  ],
  circle: [
    { x: 7, y: 6 },
    { x: 9, y: 6 },
    { x: 6, y: 7 },
    { x: 6, y: 9 },
    { x: 7, y: 10 },
    { x: 9, y: 10 },
    { x: 10, y: 7 },
    { x: 10, y: 9 },
  ],
};

// 响应式数据
const activeShape = ref(null); // 当前选中的预设阵型
const customPoints = ref([]); // 自定义绘制的点
const canvasRef = ref(null); // Canvas DOM 引用
const ctx = ref(null); // Canvas 2D 上下文
const canvasSize = ref({ width: 16, height: 16 }); // 画布逻辑尺寸（16×16）
const canvasHeight = ref(0); // 画布实际显示高度（将在初始化时设置为与宽度相同）
const canvasLoaded = ref(false); // 画布是否已加载的标志

// 页面挂载时初始化画布
onMounted(() => {
  // 使用nextTick确保DOM已完全渲染
  nextTick(() => {
    // 增加初始检查和重试机制
    checkAndInitCanvas();
    window.addEventListener('resize', handleResize);
  });
});

// 监听canvasRef变化，确保DOM元素就绪后初始化
watch(canvasRef, (newVal) => {
  if (newVal && !canvasLoaded.value) {
    checkAndInitCanvas();
  }
});

// 页面卸载时清理事件
onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

// 检查并初始化画布，增加多重保障
const checkAndInitCanvas = () => {
  if (canvasRef.value) {
    initCanvas();
  } else {
    // 如果canvas还未准备好，最多重试15次，每次间隔100ms
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

// 初始化画布 - 优化初始加载时网格显示
const initCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;

  // 让画布保持“方形”并自适应容器宽度
  const container = canvas.parentElement;
  if (!container) return;

  // 强制触发重排以获取准确尺寸
  container.getBoundingClientRect();
  // 计算容器最大可用宽度（考虑padding）
  let containerWidth = Math.min(
    container.clientWidth,
    window.innerWidth - 40 // 减去左右边距，避免溢出
  );

  // 确保容器宽度有效，即使为0也设置默认宽度保证初始网格显示
  if (containerWidth <= 0) {
    containerWidth = 300; // 设置默认宽度，确保初始网格能显示
  }

  // 设置画布为正方形
  canvas.width = containerWidth;
  canvas.height = containerWidth;
  canvasHeight.value = containerWidth;
  ctx.value = canvas.getContext('2d');

  // 确保绘制函数被调用
  drawCanvas();

  // 标记画布已加载
  canvasLoaded.value = true;

  // 如果使用了默认宽度，启动重试机制调整到正确尺寸
  if (containerWidth === 300) {
    // 增加重试次数限制，避免无限循环
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
        drawCanvas(); // 调整尺寸后重新绘制
      } else {
        setTimeout(retryInit, 100);
      }
    };

    retryInit();
  }
};

// 窗口 resize 时重绘画布
const handleResize = () => {
  if (canvasLoaded.value) {
    initCanvas();
  }
};

// 绘制画布（清空 + 画网格 + 画点）
const drawCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  // 1. 清空画布
  ctx.value.clearRect(0, 0, canvas.width, canvas.height);

  // 2. 绘制辅助网格（包含横线和竖线）
  drawGrid();

  // 3. 绘制对应阵型的点
  if (activeShape.value) {
    drawPoints(presetShapes[activeShape.value]); // 绘制预设阵型
  } else {
    drawPoints(customPoints.value); // 绘制自定义阵型
  }
};

// 绘制辅助网格（包含横线）
const drawGrid = () => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  // 绘制浅色背景
  ctx.value.fillStyle = '#f9fafb';
  ctx.value.fillRect(0, 0, canvas.width, canvas.height);

  // 绘制网格线
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

  // 画横线（初始加载就会显示）
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

// 绘制点（通用方法：接收点数组，绘制红色圆点）
const drawPoints = (points) => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value || !points.length) return;

  const cellWidth = canvas.width / canvasSize.value.width;
  const cellHeight = canvas.height / canvasSize.value.height;

  points.forEach(({ x, y }) => {
    const px = x * cellWidth;
    const py = y * cellHeight;
    // 增大圆点半径，使其更明显
    const radius = Math.min(cellWidth, cellHeight) / 2.5;

    // 添加阴影效果
    ctx.value.shadowColor = 'rgba(220, 38, 38, 0.3)';
    ctx.value.shadowBlur = 5;
    ctx.value.shadowOffsetX = 0;
    ctx.value.shadowOffsetY = 0;

    // 绘制点的外圈
    ctx.value.fillStyle = 'white';
    ctx.value.beginPath();
    ctx.value.arc(px, py, radius, 0, Math.PI * 2);
    ctx.value.fill();

    // 绘制点的内圈，使用更鲜艳的颜色
    ctx.value.fillStyle = '#dc2626';
    ctx.value.beginPath();
    ctx.value.arc(px, py, radius * 0.7, 0, Math.PI * 2);
    ctx.value.fill();

    // 重置阴影
    ctx.value.shadowColor = 'transparent';
  });
};

// 切换预设阵型
const selectShape = (shape) => {
  activeShape.value = shape;
  customPoints.value = []; // 切换预设时清空自定义点
  drawCanvas();
};

// 清除阵型
const clearShape = () => {
  activeShape.value = null;
  customPoints.value = [];
  drawCanvas();
};

// 画布点击事件（自定义绘制逻辑：添加/删除点）
const handleCanvasClick = (e) => {
  const canvas = canvasRef.value;
  if (!canvas || !ctx.value) return;

  activeShape.value = null; // 点击画布时，切换到“自定义模式”

  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  const cellWidth = canvas.width / canvasSize.value.width;
  const cellHeight = canvas.height / canvasSize.value.height;

  // 计算“逻辑坐标”（四舍五入到最近的格子）
  const logicX = Math.round(x / cellWidth);
  const logicY = Math.round(y / cellHeight);

  // 检查是否点击了已存在的点（用于“删除”）
  const pointIndex = customPoints.value.findIndex(
    (p) => p.x === logicX && p.y === logicY
  );

  if (pointIndex !== -1) {
    customPoints.value.splice(pointIndex, 1); // 删除点
  } else if (customPoints.value.length < 8) {
    customPoints.value.push({ x: logicX, y: logicY }); // 添加新点（最多8个）
  }

  drawCanvas(); // 重绘画布
};

// 确认自定义阵型
const confirmCustom = () => {
  const points = activeShape.value ? presetShapes[activeShape.value] : customPoints.value;

  if (points.length === 0) {
    alert('请绘制至少一个点后再确认～');
    return;
  }

  console.log('确认的阵型坐标：', points);
  alert('已确认阵型，坐标信息已打印到控制台～');
};
</script>

<style scoped>
/* 防止触摸画布时的默认滑动行为 */
canvas {
  touch-action: none;
  width: 100% !important;
  height: 100% !important;
  display: block; /* 解决画布额外空白问题 */
}

/* 全局样式重置 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  overflow-x: hidden; /* 防止水平滚动条 */
  width: 100%;
  height: 100%;
}

/* 自定义滚动条（保留并优化） */
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

/* 全局容器样式：最大化填充页面但不溢出 */
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

/* 预设阵型按钮：响应式布局 */
.preset-btn-group {
  display: flex;
  flex-wrap: wrap; /* 允许在小屏幕换行 */
  gap: 0.5rem;
  width: 100%;
  margin: 0 auto;
  padding: 0 0.25rem;
}
.preset-btn {
  flex: 1;
  min-width: calc(50% - 0.5rem); /* 移动端每行显示2个按钮 */
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

/* 适配平板及以上设备 - 预设按钮每行4个 */
@media (min-width: 768px) {
  .preset-btn {
    min-width: auto;
    flex: 1;
  }
}

/* 未选中状态 */
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
/* 选中状态：渐变+强阴影 */
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

/* 画布区域：保持正方形并自适应 */
.canvas-wrapper {
  width: 100%;
  max-width: 100%;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.4rem; /* 将0.6rem减小为0.4rem */
  margin-bottom: 0 !important; /* 强制底部外边距为0 */
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
  padding-top: 100%; /* 保持正方形比例 */
  position: relative;
  background: #ffffff;
  border-radius: 0.8rem;
  box-shadow: 0 8px 16px -2px rgba(0, 0, 0, 0.08), 0 4px 8px -1px rgba(0, 0, 0, 0.04);
  border: 1px solid #f1f5f9;
  overflow: hidden;
  transition: all 0.3s ease;
  max-height: 60vh; /* 移动端降低最大高度 */
}

/* 平板及以上设备提高画布最大高度 */
@media (min-width: 768px) {
  .canvas-container {
    max-height: 70vh;
  }
}

/* 绝对定位画布，使其填充容器 */
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
/* 画布hover反馈 */
canvas:hover {
  cursor: crosshair;
  background-color: #fefeff;
}

/* 操作按钮组：响应式布局 */
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
/* 清除按钮：红色系，明确警示 */
.clear-btn {
  background: linear-gradient(135deg, #f87171, #ef4444);
  color: #ffffff;
}
.clear-btn:hover {
  background: linear-gradient(135deg, #fca5a5, #dc2626);
  box-shadow: 0 8px 16px -1px rgba(239, 68, 68, 0.25);
  transform: translateY(-2px) scale(1.03);
}
/* 确认按钮：绿色系，明确正向反馈 */
.confirm-btn {
  background: linear-gradient(135deg, #4ade80, #22c55e);
  color: #ffffff;
}
.confirm-btn:hover {
  background: linear-gradient(135deg, #86efac, #16a34a);
  box-shadow: 0 8px 16px -1px rgba(34, 197, 94, 0.25);
  transform: translateY(-2px) scale(1.03);
}

/* 标题样式优化：更突出+适配屏幕 */
.page-title {
  font-size: clamp(1.5rem, 6vw, 2.5rem);
  font-weight: 700;
  color: #2563eb;
  text-align: center;
  margin: 0;
  letter-spacing: -0.02em;
  text-shadow: 0 2px 4px rgba(37, 99, 235, 0.1);
}

/* 针对超小屏幕优化（如手机竖屏） */
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
</style>