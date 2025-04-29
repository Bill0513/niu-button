<template>
  <div class="ion-container" ref="containerRef">
    <canvas v-show="isExploding" ref="canvasRef" class="ion-canvas"></canvas>
    <button class="ion-button" @click="handleClick" ref="buttonRef">
      {{ buttonText }}
    </button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onUnmounted } from "vue";

interface Particle {
  x: number;
  y: number;
  size: number;
  color: string;
  velocity: { x: number; y: number };
  alpha: number;
  decay: number;
  life: number;
  maxLife: number;
  rotation: number;
  rotationSpeed: number;
  glow: number;
}

interface LightRay {
  x: number;
  y: number;
  length: number;
  angle: number;
  width: number;
  color: string;
  alpha: number;
  decay: number;
}

export default defineComponent({
  name: "IonExplosionButton",
  props: {
    buttonText: {
      type: String,
      default: "离子爆炸",
    },
    particleCount: {
      type: Number,
      default: 80,
    },
    rayCount: {
      type: Number,
      default: 12,
    },
    duration: {
      type: Number,
      default: 2000,
    },
    colorScheme: {
      type: String,
      default: "neon", // neon, cosmic, fire, rainbow
    },
  },
  emits: ["click"],
  setup(props, { emit }) {
    const containerRef = ref<HTMLDivElement | null>(null);
    const buttonRef = ref<HTMLButtonElement | null>(null);
    const canvasRef = ref<HTMLCanvasElement | null>(null);
    const isExploding = ref(false);
    const particles = ref<Particle[]>([]);
    const lightRays = ref<LightRay[]>([]);
    let animationId: number | null = null;
    let ctx: CanvasRenderingContext2D | null = null;
    let startTime = 0;
    let lastTime = 0;

    // 保存原始点击位置
    let clickX = 0;
    let clickY = 0;

    // 颜色方案
    const colorPalettes = {
      neon: ["#ff00ff", "#00ffff", "#ff3377", "#77ffdd", "#ffff00"],
      cosmic: ["#5533ff", "#3377ff", "#22ccff", "#0088ff", "#aaeeff"],
      fire: ["#ff3300", "#ff7700", "#ffaa00", "#ffdd00", "#ffff33"],
      rainbow: [
        "#ff0000",
        "#ff9900",
        "#ffff00",
        "#33ff00",
        "#0099ff",
        "#6633ff",
        "#ff33cc",
      ],
    };

    // 获取当前颜色方案
    const getColors = () => {
      return (
        colorPalettes[props.colorScheme as keyof typeof colorPalettes] ||
        colorPalettes.neon
      );
    };

    // 随机生成颜色
    const getRandomColor = () => {
      const colors = getColors();
      return colors[Math.floor(Math.random() * colors.length)];
    };

    // 设置Canvas尺寸
    const setupCanvas = () => {
      if (canvasRef.value) {
        // 创建全屏Canvas
        canvasRef.value.width = window.innerWidth * 2;
        canvasRef.value.height = window.innerHeight * 2;

        // 重置Canvas位置为屏幕中心
        canvasRef.value.style.left = `${-window.innerWidth / 2}px`;
        canvasRef.value.style.top = `${-window.innerHeight / 2}px`;
      }
    };

    onMounted(() => {
      window.addEventListener("resize", setupCanvas);
      setupCanvas();
    });

    onUnmounted(() => {
      if (animationId !== null) {
        cancelAnimationFrame(animationId);
      }
      window.removeEventListener("resize", setupCanvas);
    });

    // 创建粒子
    const createParticles = (x: number, y: number) => {
      const newParticles: Particle[] = [];
      const colors = getColors();
      const particleCount = Math.min(props.particleCount, 100);

      for (let i = 0; i < particleCount; i++) {
        const speed = 2 + Math.random() * 10;
        const angle = Math.random() * Math.PI * 2;
        const size = Math.random() * 6 + 2;
        const life = 0;
        const maxLife = Math.random() * 60 + 20;
        const color = colors[Math.floor(Math.random() * colors.length)];
        const decay = 0.01 + Math.random() * 0.03;
        const rotation = Math.random() * Math.PI * 2;
        const rotationSpeed = (Math.random() - 0.5) * 0.1;
        const glow = 3 + Math.random() * 8;

        newParticles.push({
          x,
          y,
          size,
          color,
          velocity: {
            x: Math.cos(angle) * speed,
            y: Math.sin(angle) * speed,
          },
          alpha: 0.8 + Math.random() * 0.2,
          decay,
          life,
          maxLife,
          rotation,
          rotationSpeed,
          glow,
        });
      }

      return newParticles;
    };

    // 创建光线
    const createLightRays = (x: number, y: number) => {
      const newRays: LightRay[] = [];
      const colors = getColors();
      const rayCount = Math.min(props.rayCount, 15);

      for (let i = 0; i < rayCount; i++) {
        const angle = (i / rayCount) * Math.PI * 2;
        const length = 100 + Math.random() * 150;
        const width = 1 + Math.random() * 4;
        const color = colors[Math.floor(Math.random() * colors.length)];

        newRays.push({
          x,
          y,
          length,
          angle,
          width,
          color,
          alpha: 0.6 + Math.random() * 0.3,
          decay: 0.02 + Math.random() * 0.03,
        });
      }

      return newRays;
    };

    // 绘制粒子
    const drawParticle = (particle: Particle) => {
      if (!ctx) return;

      ctx.save();
      ctx.translate(particle.x, particle.y);
      ctx.rotate(particle.rotation);

      ctx.shadowBlur = particle.glow / 2;
      ctx.shadowColor = particle.color;
      ctx.globalAlpha = particle.alpha;

      // 简化形状
      ctx.beginPath();
      if (Math.random() > 0.7) {
        // 星形
        const spikes = 4;
        const outerRadius = particle.size;
        const innerRadius = particle.size / 2;

        for (let i = 0; i < spikes * 2; i++) {
          const radius = i % 2 === 0 ? outerRadius : innerRadius;
          const currAngle = (i / (spikes * 2)) * Math.PI * 2;
          const x = Math.cos(currAngle) * radius;
          const y = Math.sin(currAngle) * radius;

          if (i === 0) ctx.moveTo(x, y);
          else ctx.lineTo(x, y);
        }
      } else {
        // 圆形
        ctx.arc(0, 0, particle.size, 0, Math.PI * 2);
      }
      ctx.closePath();

      ctx.fillStyle = particle.color;
      ctx.fill();

      if (Math.random() > 0.5) {
        ctx.strokeStyle = "rgba(255, 255, 255, 0.5)";
        ctx.lineWidth = 0.5;
        ctx.stroke();
      }

      ctx.restore();
    };

    // 绘制光线
    const drawLightRay = (ray: LightRay) => {
      if (!ctx) return;

      ctx.save();

      if (Math.random() > 0.3) {
        const gradient = ctx.createLinearGradient(
          ray.x,
          ray.y,
          ray.x + Math.cos(ray.angle) * ray.length,
          ray.y + Math.sin(ray.angle) * ray.length
        );

        gradient.addColorStop(0, ray.color);
        gradient.addColorStop(1, "rgba(255, 255, 255, 0)");

        ctx.strokeStyle = gradient;
      } else {
        ctx.strokeStyle = ray.color;
      }

      ctx.globalAlpha = ray.alpha;
      ctx.lineWidth = ray.width;
      ctx.lineCap = "round";

      if (Math.random() > 0.7) {
        ctx.shadowBlur = 5;
        ctx.shadowColor = ray.color;
      }

      ctx.beginPath();
      ctx.moveTo(ray.x, ray.y);
      ctx.lineTo(
        ray.x + Math.cos(ray.angle) * ray.length,
        ray.y + Math.sin(ray.angle) * ray.length
      );
      ctx.stroke();

      ctx.restore();
    };

    // 绘制波形光晕
    const drawShockwave = (x: number, y: number, progress: number) => {
      if (!ctx) return;

      const maxRadius = 80;
      const radius = maxRadius * progress;
      const opacity = (1 - progress) * 0.4;

      ctx.save();
      ctx.globalAlpha = opacity;

      ctx.beginPath();
      ctx.arc(x, y, radius, 0, Math.PI * 2);
      ctx.fillStyle = getRandomColor();
      ctx.fill();

      ctx.restore();
    };

    // 更新动画
    const updateEffect = (timestamp: number) => {
      if (!ctx || !canvasRef.value) return;

      const deltaTime = timestamp - lastTime;
      lastTime = timestamp;

      if (deltaTime < 8) {
        animationId = requestAnimationFrame(updateEffect);
        return;
      }

      const elapsed = timestamp - startTime;
      const progress = Math.min(1, elapsed / props.duration);

      // 清空画布
      ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);

      // 计算爆炸效果在画布上的位置
      // 将点击坐标转换为Canvas坐标系中的位置
      const explosionX = clickX + window.innerWidth / 2;
      const explosionY = clickY + window.innerHeight / 2;

      // 更新并绘制光线
      for (let i = lightRays.value.length - 1; i >= 0; i--) {
        const ray = lightRays.value[i];
        ray.alpha = Math.max(0, ray.alpha - ray.decay);

        if (ray.alpha > 0.01) {
          drawLightRay(ray);
        } else {
          lightRays.value.splice(i, 1);
        }
      }

      // 更新并绘制粒子
      for (let i = particles.value.length - 1; i >= 0; i--) {
        const particle = particles.value[i];

        particle.x += particle.velocity.x;
        particle.y += particle.velocity.y;

        particle.velocity.y += 0.03;
        particle.velocity.x *= 0.99;
        particle.velocity.y *= 0.99;

        particle.rotation += particle.rotationSpeed;
        particle.life += 1;
        particle.alpha = Math.max(0, particle.alpha - particle.decay);

        if (particle.alpha > 0.01 && particle.life < particle.maxLife) {
          drawParticle(particle);
        } else {
          particles.value.splice(i, 1);
        }
      }

      // 绘制冲击波
      if (progress < 0.3) {
        const waveProgress = progress / 0.3;
        drawShockwave(explosionX, explosionY, waveProgress);

        if (progress > 0.1 && progress < 0.2) {
          drawShockwave(
            explosionX + (Math.random() - 0.5) * 10,
            explosionY + (Math.random() - 0.5) * 10,
            (progress - 0.1) * 10
          );
        }
      }

      // 判断是否继续动画
      if (
        progress < 1 &&
        (particles.value.length > 0 || lightRays.value.length > 0)
      ) {
        animationId = requestAnimationFrame(updateEffect);
      } else {
        isExploding.value = false;
      }
    };

    // 点击处理
    const handleClick = (event: MouseEvent) => {
      // 发送原始点击事件
      emit("click", event);

      if (canvasRef.value) {
        isExploding.value = true;

        // 记录点击位置
        clickX = event.clientX;
        clickY = event.clientY;

        // 重设Canvas
        setupCanvas();
        ctx = canvasRef.value.getContext("2d");

        if (!ctx) return;

        // 将点击位置转换为Canvas坐标系中的位置
        const canvasClickX = clickX + window.innerWidth / 2;
        const canvasClickY = clickY + window.innerHeight / 2;

        // 生成粒子和光线，以点击位置为中心
        particles.value = createParticles(canvasClickX, canvasClickY);
        lightRays.value = createLightRays(canvasClickX, canvasClickY);

        // 记录起始时间和上一帧时间
        startTime = performance.now();
        lastTime = startTime;

        // 取消之前的动画
        if (animationId !== null) {
          cancelAnimationFrame(animationId);
        }

        // 开始动画
        animationId = requestAnimationFrame(updateEffect);

        // 在指定时间后强制结束动画
        setTimeout(() => {
          if (animationId !== null) {
            cancelAnimationFrame(animationId);
            isExploding.value = false;
          }
        }, props.duration);
      }
    };

    return {
      containerRef,
      buttonRef,
      canvasRef,
      isExploding,
      handleClick,
    };
  },
});
</script>

<style scoped>
.ion-container {
  position: relative;
  display: inline-block;
}

.ion-button {
  padding: 12px 24px;
  background: linear-gradient(135deg, #7d00cc, #b300ea);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s ease;
  box-shadow: 0 0 10px rgba(138, 43, 226, 0.5),
    inset 0 0 5px rgba(255, 255, 255, 0.5);
  text-shadow: 0 0 5px white;
  position: relative;
  z-index: 10;
  overflow: hidden;
}

.ion-button::before {
  content: "";
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(
    circle,
    rgba(255, 255, 255, 0.3) 0%,
    rgba(255, 255, 255, 0) 70%
  );
  transform: rotate(30deg);
  opacity: 0.3;
  transition: all 0.3s ease;
  z-index: -1;
}

.ion-button:hover {
  background: linear-gradient(135deg, #9000ea, #c600ff);
  box-shadow: 0 0 20px rgba(186, 85, 255, 0.8),
    inset 0 0 10px rgba(255, 255, 255, 0.7);
  transform: translateY(-2px);
}

.ion-button:hover::before {
  opacity: 0.5;
}

.ion-button:active {
  transform: translateY(1px);
}

.ion-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 200vw;
  height: 200vh;
  pointer-events: none;
  z-index: 1; /* 确保画布在按钮后面 */
}
</style>
