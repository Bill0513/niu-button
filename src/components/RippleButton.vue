<template>
  <div class="ripple-container" ref="containerRef">
    <button class="ripple-button" @click="handleClick" ref="buttonRef">
      {{ buttonText }}
    </button>
    <div class="canvas-wrapper" v-show="isRippling">
      <canvas ref="canvasRef" class="ripple-canvas"></canvas>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onUnmounted, PropType } from "vue";

interface Ripple {
  x: number;
  y: number;
  radius: number;
  maxRadius: number;
  opacity: number;
  color: string;
  speed: number;
}

export default defineComponent({
  name: "RippleButton",
  props: {
    buttonText: {
      type: String,
      default: "点击涟漪",
    },
    rippleColor: {
      type: String,
      default: "#3498db", // 与破碎按钮相同的蓝色
    },
    rippleCount: {
      type: Number,
      default: 5, // 波纹数量
    },
    duration: {
      type: Number,
      default: 2000, // 持续时间(毫秒)
    },
  },
  emits: ["click"],
  setup(props, { emit }) {
    const containerRef = ref<HTMLDivElement | null>(null);
    const buttonRef = ref<HTMLButtonElement | null>(null);
    const canvasRef = ref<HTMLCanvasElement | null>(null);
    const isRippling = ref(false);
    const ripples = ref<Ripple[]>([]);
    let animationId: number | null = null;
    let ctx: CanvasRenderingContext2D | null = null;

    // 设置Canvas尺寸，使其足够大以容纳扩散的波纹
    const setupCanvas = () => {
      if (canvasRef.value && buttonRef.value) {
        const buttonRect = buttonRef.value.getBoundingClientRect();
        // 使Canvas尺寸扩大到足够容纳最大波纹
        const size = Math.max(window.innerWidth, window.innerHeight) * 2;
        canvasRef.value.width = size;
        canvasRef.value.height = size;
      }
    };

    onMounted(() => {
      window.addEventListener("resize", setupCanvas);
      // 初始化Canvas尺寸
      setupCanvas();
    });

    onUnmounted(() => {
      if (animationId !== null) {
        cancelAnimationFrame(animationId);
      }
      window.removeEventListener("resize", setupCanvas);
    });

    // 创建水波纹
    const createRipple = (x: number, y: number, delay = 0): Ripple => {
      // 为波纹生成轻微的色彩变化
      const colorVariation = Math.floor(Math.random() * 20) - 10;
      const baseColor = props.rippleColor;

      // 提取RGB值并添加变化
      const r = parseInt(baseColor.slice(1, 3), 16) + colorVariation;
      const g = parseInt(baseColor.slice(3, 5), 16) + colorVariation;
      const b = parseInt(baseColor.slice(5, 7), 16) + colorVariation;

      // 确保RGB值在有效范围内
      const boundValue = (value: number) => Math.min(255, Math.max(0, value));
      const colorHex = `#${boundValue(r)
        .toString(16)
        .padStart(2, "0")}${boundValue(g)
        .toString(16)
        .padStart(2, "0")}${boundValue(b).toString(16).padStart(2, "0")}`;

      // 允许更大的最大半径，使波纹可以扩散更远
      const maxRadius = 300 + Math.random() * 200;

      return {
        x,
        y,
        radius: delay === 0 ? 5 : Math.random() * 5 + 5, // 初始半径
        maxRadius,
        opacity: 0.7,
        color: colorHex,
        speed: 1.5 + Math.random() * 1, // 扩散速度
      };
    };

    // 绘制波纹
    const drawRipple = (ripple: Ripple) => {
      if (!ctx) return;

      // 波纹轮廓
      ctx.beginPath();
      ctx.arc(ripple.x, ripple.y, ripple.radius, 0, Math.PI * 2);
      ctx.strokeStyle = `rgba(${parseInt(
        ripple.color.slice(1, 3),
        16
      )}, ${parseInt(ripple.color.slice(3, 5), 16)}, ${parseInt(
        ripple.color.slice(5, 7),
        16
      )}, ${ripple.opacity * 0.8})`;
      ctx.lineWidth = 2;
      ctx.stroke();

      // 半透明填充
      ctx.globalAlpha = ripple.opacity * 0.2;
      ctx.fillStyle = ripple.color;
      ctx.fill();
      ctx.globalAlpha = 1;
    };

    // 更新波纹动画
    const updateRipples = () => {
      if (!ctx || !canvasRef.value) return;

      // 清空画布
      ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);

      let allDone = true;

      for (let i = 0; i < ripples.value.length; i++) {
        const ripple = ripples.value[i];

        // 更新半径
        ripple.radius += ripple.speed;

        // 更新透明度 - 随着半径增大而减小
        ripple.opacity = Math.max(0, 1 - ripple.radius / ripple.maxRadius);

        // 如果还有可见波纹，继续动画
        if (ripple.opacity > 0.01) {
          allDone = false;
          drawRipple(ripple);
        }
      }

      if (!allDone) {
        animationId = requestAnimationFrame(updateRipples);
      } else {
        isRippling.value = false;
        ripples.value = [];
      }
    };

    // 点击处理函数
    const handleClick = (event: MouseEvent) => {
      // 发送原始点击事件
      emit("click", event);

      if (buttonRef.value && canvasRef.value) {
        // 获取按钮位置
        const buttonRect = buttonRef.value.getBoundingClientRect();

        // 获取点击位置相对于按钮中心的坐标
        const buttonCenterX = buttonRect.left + buttonRect.width / 2;
        const buttonCenterY = buttonRect.top + buttonRect.height / 2;

        // 点击位置相对于按钮的偏移
        const offsetX = event.clientX - buttonCenterX;
        const offsetY = event.clientY - buttonCenterY;

        // 计算Canvas中心点
        const canvasCenterX = canvasRef.value.width / 2;
        const canvasCenterY = canvasRef.value.height / 2;

        // 波纹在Canvas上的位置（Canvas中心点 + 偏移量）
        const rippleX = canvasCenterX + offsetX;
        const rippleY = canvasCenterY + offsetY;

        isRippling.value = true;
        ctx = canvasRef.value.getContext("2d");

        if (!ctx) return;

        // 清空先前的波纹
        ripples.value = [];

        // 创建主波纹
        ripples.value.push(createRipple(rippleX, rippleY));

        // 创建附加波纹，每个波纹有少量延迟
        for (let i = 1; i < props.rippleCount; i++) {
          setTimeout(() => {
            if (isRippling.value) {
              ripples.value.push(
                createRipple(
                  rippleX + (Math.random() - 0.5) * 5,
                  rippleY + (Math.random() - 0.5) * 5,
                  i
                )
              );
            }
          }, i * 100);
        }

        // 开始动画
        if (animationId !== null) {
          cancelAnimationFrame(animationId);
        }
        animationId = requestAnimationFrame(updateRipples);
      }
    };

    return {
      containerRef,
      buttonRef,
      canvasRef,
      isRippling,
      handleClick,
    };
  },
});
</script>

<style scoped>
.ripple-container {
  position: relative;
  display: inline-block;
}

.ripple-button {
  padding: 12px 24px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  position: relative;
  z-index: 3;
}

.ripple-button:hover {
  background-color: #2980b9;
}

.canvas-wrapper {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  overflow: visible;
  z-index: 1;
  pointer-events: none;
}

.ripple-canvas {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
}
</style>
