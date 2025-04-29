<template>
  <div class="shatter-container">
    <button class="shatter-button" @click="handleClick">
      {{ buttonText }}
    </button>
    <canvas
      v-show="isShattered"
      ref="canvasRef"
      class="shatter-canvas"
    ></canvas>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onUnmounted, PropType } from "vue";

// 定义碎片类型接口
interface Shard {
  x: number;
  y: number;
  size: number;
  velocity: { x: number; y: number };
  rotation: number;
  rotationSpeed: number;
  opacity: number;
  points: Array<{ x: number; y: number }>;
}

export default defineComponent({
  name: "ShatterButton",
  props: {
    buttonText: {
      type: String,
      default: "点击破裂",
    },
    shardCount: {
      type: Number,
      default: 40,
    },
    crackCount: {
      type: Number,
      default: 8,
    },
    animationDuration: {
      type: Number,
      default: 1500, // 毫秒
    },
  },
  emits: ["click"],
  setup(props, { emit }) {
    const canvasRef = ref<HTMLCanvasElement | null>(null);
    const isShattered = ref(false);
    const shards = ref<Shard[]>([]);
    let animationId: number | null = null;
    let ctx: CanvasRenderingContext2D | null = null;

    // 屏幕尺寸调整
    const handleResize = () => {
      if (canvasRef.value) {
        canvasRef.value.width = window.innerWidth;
        canvasRef.value.height = window.innerHeight;
      }
    };

    onMounted(() => {
      window.addEventListener("resize", handleResize);
    });

    onUnmounted(() => {
      if (animationId !== null) {
        cancelAnimationFrame(animationId);
      }
      window.removeEventListener("resize", handleResize);
    });

    // 创建随机碎片形状
    const createShard = (centerX: number, centerY: number): Shard => {
      const size = Math.random() * 30 + 10;
      const speed = Math.random() * 5 + 3;
      const angle = Math.random() * Math.PI * 2;

      // 计算碎片飞散的方向和速度
      const velocityX = Math.cos(angle) * speed;
      const velocityY = Math.sin(angle) * speed;

      // 创建不规则多边形碎片
      const pointCount = Math.floor(Math.random() * 3) + 3; // 3-5个顶点
      const points = [];

      for (let i = 0; i < pointCount; i++) {
        const pointAngle = (i / pointCount) * Math.PI * 2;
        const distance = size * (0.7 + Math.random() * 0.3); // 不规则半径
        points.push({
          x: Math.cos(pointAngle) * distance,
          y: Math.sin(pointAngle) * distance,
        });
      }

      return {
        x: centerX,
        y: centerY,
        size,
        velocity: { x: velocityX, y: velocityY },
        rotation: Math.random() * Math.PI * 2,
        rotationSpeed: (Math.random() - 0.5) * 0.2,
        opacity: 1,
        points,
      };
    };

    // 绘制碎片
    const drawShard = (shard: Shard) => {
      if (!ctx) return;

      ctx.save();
      ctx.translate(shard.x, shard.y);
      ctx.rotate(shard.rotation);
      ctx.globalAlpha = shard.opacity;

      // 绘制玻璃碎片效果
      ctx.beginPath();
      ctx.moveTo(shard.points[0].x, shard.points[0].y);

      for (let i = 1; i < shard.points.length; i++) {
        ctx.lineTo(shard.points[i].x, shard.points[i].y);
      }

      ctx.closePath();

      // 玻璃质感
      const gradient = ctx.createLinearGradient(
        -shard.size,
        -shard.size,
        shard.size,
        shard.size
      );
      gradient.addColorStop(0, "rgba(255, 255, 255, 0.7)");
      gradient.addColorStop(1, "rgba(200, 200, 255, 0.5)");

      ctx.fillStyle = gradient;
      ctx.fill();

      // 边缘高光
      ctx.strokeStyle = "rgba(255, 255, 255, 0.8)";
      ctx.lineWidth = 1;
      ctx.stroke();

      ctx.restore();
    };

    // 动画更新
    const updateShards = () => {
      if (!ctx || !canvasRef.value) return;

      ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);

      let allDone = true;

      for (let i = 0; i < shards.value.length; i++) {
        const shard = shards.value[i];

        // 更新位置
        shard.x += shard.velocity.x;
        shard.y += shard.velocity.y;

        // 添加重力效果
        shard.velocity.y += 0.2;

        // 添加旋转
        shard.rotation += shard.rotationSpeed;

        // 减小不透明度 - 基于动画持续时间
        shard.opacity -= 1 / (props.animationDuration / 16); // 假设16ms一帧

        // 如果还有可见碎片，继续动画
        if (shard.opacity > 0) {
          allDone = false;
          drawShard(shard);
        }
      }

      if (!allDone) {
        animationId = requestAnimationFrame(updateShards);
      } else {
        isShattered.value = false;
        shards.value = [];
      }
    };

    // 点击处理函数
    const handleClick = (event: MouseEvent) => {
      // 发送原始点击事件
      emit("click", event);

      const { clientX, clientY } = event;

      if (!isShattered.value) {
        isShattered.value = true;

        if (canvasRef.value) {
          canvasRef.value.width = window.innerWidth;
          canvasRef.value.height = window.innerHeight;

          ctx = canvasRef.value.getContext("2d");
          if (!ctx) return;

          // 创建碎片
          shards.value = [];

          // 中心碎片
          for (let i = 0; i < props.shardCount; i++) {
            shards.value.push(createShard(clientX, clientY));
          }

          // 创建裂纹线
          for (let i = 0; i < props.crackCount; i++) {
            const angle = (i / props.crackCount) * Math.PI * 2;
            const length = Math.random() * 200 + 100;

            // 沿着裂纹线添加更多碎片
            for (let j = 0; j < 5; j++) {
              const distance = ((j + 1) * length) / 5;
              const x = clientX + Math.cos(angle) * distance;
              const y = clientY + Math.sin(angle) * distance;
              const jitter = 20;
              shards.value.push(
                createShard(
                  x + (Math.random() - 0.5) * jitter,
                  y + (Math.random() - 0.5) * jitter
                )
              );
            }
          }

          // 开始动画
          if (animationId !== null) {
            cancelAnimationFrame(animationId);
          }
          animationId = requestAnimationFrame(updateShards);
        }
      }
    };

    return {
      canvasRef,
      isShattered,
      handleClick,
    };
  },
});
</script>

<style scoped>
.shatter-container {
  position: relative;
  display: inline-block;
}

.shatter-button {
  padding: 12px 24px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.shatter-button:hover {
  background-color: #2980b9;
}

.shatter-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 9999;
  pointer-events: none;
}
</style>
