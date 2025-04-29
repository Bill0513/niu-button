<template>
  <div class="app">
    <header class="app-header">
      <h1>特效按钮演示</h1>
      <p>点击下面的按钮，体验不同的视觉效果！</p>
    </header>
    <main class="content">
      <div class="buttons-container">
        <ShatterButton button-text="点击炸裂屏幕" :shard-count="50" :crack-count="10" :animation-duration="2000"
          @click="handleShatterClick" />
        <RippleButton button-text="点击水波效果" ripple-color="#3498db" :ripple-count="5" :duration="2000"
          @click="handleRippleClick" />
        <IonExplosionButton button-text="离子爆炸" color-scheme="neon" :particle-count="150" :ray-count="20"
          :duration="2500" @click="handleIonClick" />

        <JellyButton button-text="果冻按钮" button-color="#FF6B6B" :intensity="1.2" :duration="1200"
          @click="handleJellyClick" />

        <BlackHoleButton button-text="召唤卡冈图雅" :intensity="1.3" :duration="6000" @click="handleBlackHoleClick" />
      </div>
      <div class="info-panel">
        <h2>点击统计</h2>
        <p>破裂效果点击: {{ shatterClickCount }} 次</p>
        <p>水波效果点击: {{ rippleClickCount }} 次</p>
        <p>离子爆炸点击: {{ ionClickCount }} 次</p>
        <p>果冻按钮点击: {{ jellyClickCount }} 次</p>
        <p>黑洞按钮点击: {{ blackHoleClickCount }} 次</p>
        <p>
          总点击次数:
          {{ shatterClickCount + rippleClickCount + ionClickCount + jellyClickCount + blackHoleClickCount }} 次
        </p>
        <div class="description">
          <h3>关于这些效果</h3>
          <p>这是使用Vue3和TypeScript实现的五种不同的视觉交互效果:</p>
          <ul>
            <li>
              <strong>破裂效果</strong> - 模拟屏幕被击碎的效果，碎片向四周飞散
            </li>
            <li>
              <strong>水波效果</strong> - 模拟水滴落入水面产生的同心圆波纹
            </li>
            <li>
              <strong>离子爆炸</strong> - 模拟炫目的离子光束和粒子爆炸效果
            </li>
            <li>
              <strong>果冻按钮</strong> - 模拟真实的果冻物理效果，按钮点击后会像果冻一样晃动
            </li>
            <li>
              <strong>黑洞效果</strong> - 模拟宇宙黑洞的视觉效果，空间扭曲、光线弯曲和吸积盘旋转
            </li>
          </ul>
          <p>这些效果都使用先进的前端技术实现，包括Canvas动态绘制、SVG路径动画、基于物理的弹簧动画系统以及复杂的粒子系统，可以根据需要进行自定义配置。</p>
        </div>
      </div>
    </main>
  </div>
</template>
<script lang="ts">
import { defineComponent, ref } from "vue";
import ShatterButton from "./components/ShatterButton.vue";
import RippleButton from "./components/RippleButton.vue";
import IonExplosionButton from "./components/IonExplosionButton.vue";
import JellyButton from "./components/JellyButton.vue";
import BlackHoleButton from "./components/BlackHoleButton.vue";
export default defineComponent({
  name: "App",
  components: {
    ShatterButton,
    RippleButton,
    IonExplosionButton,
    JellyButton,
    BlackHoleButton
  },
  setup() {
    const shatterClickCount = ref(0);
    const rippleClickCount = ref(0);
    const ionClickCount = ref(0);
    const jellyClickCount = ref(0);
    const blackHoleClickCount = ref(0);
    const handleShatterClick = () => {
      shatterClickCount.value++;
    };
    const handleRippleClick = () => {
      rippleClickCount.value++;
    };
    const handleIonClick = () => {
      ionClickCount.value++;
    };

    const handleJellyClick = () => {
      jellyClickCount.value++;
    };

    const handleBlackHoleClick = () => {
      blackHoleClickCount.value++;
    };
    return {
      shatterClickCount,
      rippleClickCount,
      ionClickCount,
      jellyClickCount,
      blackHoleClickCount,
      handleShatterClick,
      handleRippleClick,
      handleIonClick,
      handleJellyClick,
      handleBlackHoleClick
    };
  },
});
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background-color: #f5f5f5;
  color: #333;
  line-height: 1.6;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.app-header {
  text-align: center;
  padding: 50px 20px;
  background-color: #2c3e50;
  color: white;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.app-header h1 {
  font-size: 36px;
  margin-bottom: 10px;
}

.content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex: 1;
  padding: 50px 20px;
}

.buttons-container {
  display: flex;
  gap: 50px;
  margin-bottom: 60px;
  flex-wrap: wrap;
  justify-content: center;
}

.info-panel {
  padding: 30px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  max-width: 800px;
  width: 100%;
}

.info-panel h2 {
  text-align: center;
  margin-bottom: 20px;
  color: #2c3e50;
  font-size: 24px;
}

.info-panel p {
  margin: 10px 0;
  font-size: 16px;
}

.description {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid #eee;
}

.description h3 {
  margin-bottom: 15px;
  color: #2c3e50;
}

.description ul {
  margin: 15px 0;
  padding-left: 20px;
}

.description li {
  margin-bottom: 8px;
}

@media (max-width: 600px) {
  .buttons-container {
    flex-direction: column;
    gap: 40px;
  }

  .info-panel {
    padding: 20px;
  }
}
</style>
