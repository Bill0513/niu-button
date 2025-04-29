<template>
    <div class="jelly-container" ref="containerRef">
        <button class="jelly-button" ref="buttonRef" @click="handleClick" :style="buttonStyle">
            <svg class="jelly-svg" width="100%" height="100%" viewBox="0 0 200 70" preserveAspectRatio="none">
                <defs>
                    <filter id="gooey-filter">
                        <feGaussianBlur in="SourceGraphic" stdDeviation="8" result="blur" />
                        <feColorMatrix in="blur" mode="matrix" values="1 0 0 0 0  0 1 0 0 0  0 0 1 0 0  0 0 0 19 -9"
                            result="gooey" />
                        <feComposite in="SourceGraphic" in2="gooey" operator="atop" />
                    </filter>
                    <linearGradient id="jellygradient" x1="0%" y1="0%" x2="0%" y2="100%">
                        <stop offset="0%" :stop-color="colorLight" />
                        <stop offset="50%" :stop-color="buttonColor" />
                        <stop offset="100%" :stop-color="colorDark" />
                    </linearGradient>
                </defs>
                <path ref="pathRef" d="M0,0 L200,0 L200,70 L0,70 Z" fill="url(#jellygradient)"
                    filter="url(#gooey-filter)" />
            </svg>
            <div class="jelly-content">
                <div class="jelly-reflection"></div>
                <span class="jelly-btn-text">{{ buttonText }}</span>
            </div>
        </button>
    </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, onMounted, onBeforeUnmount, computed, watch, toRefs } from 'vue';

interface SpringPoint {
    x: number;
    y: number;
    ox: number;
    oy: number;
    vx: number;
    vy: number;
}

interface JellyConfig {
    stiffness: number;
    damping: number;
    mass: number;
    bounce: number;
}

export default defineComponent({
    name: 'JellyButton',
    props: {
        buttonText: {
            type: String,
            default: '果冻按钮'
        },
        buttonColor: {
            type: String,
            default: '#4e9eff'
        },
        intensity: {
            type: Number,
            default: 1.0 // 晃动强度系数，可以调整
        },
        duration: {
            type: Number,
            default: 1000 // 晃动持续时间
        }
    },
    emits: ['click'],
    setup(props, { emit }) {
        // Use toRefs to make props reactive and accessible in the template
        const { buttonText, buttonColor, intensity, duration } = toRefs(props);

        const containerRef = ref<HTMLDivElement | null>(null);
        const buttonRef = ref<HTMLButtonElement | null>(null);
        const pathRef = ref<SVGPathElement | null>(null);
        const isActive = ref(false);
        const isAnimating = ref(false);
        const points = ref<SpringPoint[]>([]);
        const animationId = ref<number | null>(null);
        const lastTime = ref(0);
        const handleResize = () => {
            initPoints();
            updatePath();
        };

        // 设置果冻物理参数
        const jellyConfig = reactive<JellyConfig>({
            stiffness: 0.08 * intensity.value, // 增加弹簧刚度
            damping: 0.12,                     // 减少阻尼系数使晃动更明显
            mass: 1.0,                         // 质量
            bounce: 1.3 * intensity.value      // 弹跳系数
        });

        // 计算按钮颜色样式
        const colorLight = computed(() => adjustColor(buttonColor.value, 30));
        const colorDark = computed(() => adjustColor(buttonColor.value, -30));

        const buttonStyle = computed(() => {
            return {
                '--button-color': buttonColor.value,
                '--button-color-dark': colorDark.value,
                '--button-color-light': colorLight.value,
                '--button-color-transparent': buttonColor.value + '33', // 添加透明度
            };
        });

        // 调整颜色亮度
        const adjustColor = (hex: string, amount: number): string => {
            let r = parseInt(hex.slice(1, 3), 16);
            let g = parseInt(hex.slice(3, 5), 16);
            let b = parseInt(hex.slice(5, 7), 16);

            r = Math.max(0, Math.min(255, r + amount));
            g = Math.max(0, Math.min(255, g + amount));
            b = Math.max(0, Math.min(255, b + amount));

            return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
        };

        // 初始化弹簧点
        const initPoints = () => {
            if (!buttonRef.value) return;

            const width = buttonRef.value.offsetWidth;
            const height = buttonRef.value.offsetHeight;

            // 创建控制果冻形状的点
            // 为了平滑的果冻效果，我们需要足够多的点
            const numPoints = 16; // 增加点的数量使果冻效果更平滑
            points.value = [];

            for (let i = 0; i <= numPoints; i++) {
                const xPos = (i / numPoints) * width;

                // 添加顶部点
                points.value.push({
                    x: xPos,
                    y: 0,
                    ox: xPos,
                    oy: 0,
                    vx: 0,
                    vy: 0
                });

                // 添加底部点
                points.value.push({
                    x: xPos,
                    y: height,
                    ox: xPos,
                    oy: height,
                    vx: 0,
                    vy: 0
                });
            }

            // 添加左右两侧的点
            for (let i = 1; i < numPoints; i++) {
                const yPos = (i / numPoints) * height;

                // 添加左侧点
                points.value.push({
                    x: 0,
                    y: yPos,
                    ox: 0,
                    oy: yPos,
                    vx: 0,
                    vy: 0
                });

                // 添加右侧点
                points.value.push({
                    x: width,
                    y: yPos,
                    ox: width,
                    oy: yPos,
                    vx: 0,
                    vy: 0
                });
            }
        };

        // 更新SVG路径 - 使用贝塞尔曲线使形状更平滑
        const updatePath = () => {
            if (!pathRef.value || points.value.length < 4) return;

            const width = buttonRef.value?.offsetWidth || 200;
            const height = buttonRef.value?.offsetHeight || 70;

            // 过滤出顶部点、右侧点、底部点和左侧点
            const topPoints = points.value.filter(p => p.oy === 0).sort((a, b) => a.ox - b.ox);
            const rightPoints = points.value.filter(p => p.ox === width).sort((a, b) => a.oy - b.oy);
            const bottomPoints = points.value.filter(p => p.oy === height).sort((a, b) => b.ox - a.ox);
            const leftPoints = points.value.filter(p => p.ox === 0).sort((a, b) => b.oy - a.oy);

            // 构建SVG路径 - 使用三次贝塞尔曲线使形状更平滑
            let pathData = `M${topPoints[0].x},${topPoints[0].y}`;

            // 添加顶部曲线 - 使用三次贝塞尔曲线
            for (let i = 1; i < topPoints.length; i++) {
                const cp1x = topPoints[i - 1].x + (topPoints[i].x - topPoints[i - 1].x) / 3;
                const cp1y = topPoints[i - 1].y;
                const cp2x = topPoints[i].x - (topPoints[i].x - topPoints[i - 1].x) / 3;
                const cp2y = topPoints[i].y;
                pathData += ` C${cp1x},${cp1y} ${cp2x},${cp2y} ${topPoints[i].x},${topPoints[i].y}`;
            }

            // 添加右侧曲线
            for (let i = 0; i < rightPoints.length; i++) {
                const cp1x = rightPoints[i].x;
                const cp1y = rightPoints[i].y + (i === 0 ? 0 : (rightPoints[i].y - rightPoints[i - 1].y) / 3);
                const cp2x = rightPoints[i === rightPoints.length - 1 ? i : i + 1].x;
                const cp2y = rightPoints[i].y + (i === rightPoints.length - 1 ? 0 : (rightPoints[i + 1].y - rightPoints[i].y) / 3);
                pathData += ` C${cp1x},${cp1y} ${cp2x},${cp2y} ${rightPoints[i === rightPoints.length - 1 ? i : i + 1].x},${rightPoints[i === rightPoints.length - 1 ? i : i + 1].y}`;
            }

            // 添加底部曲线
            for (let i = 0; i < bottomPoints.length - 1; i++) {
                const cp1x = bottomPoints[i].x - (bottomPoints[i].x - bottomPoints[i + 1].x) / 3;
                const cp1y = bottomPoints[i].y;
                const cp2x = bottomPoints[i + 1].x + (bottomPoints[i].x - bottomPoints[i + 1].x) / 3;
                const cp2y = bottomPoints[i + 1].y;
                pathData += ` C${cp1x},${cp1y} ${cp2x},${cp2y} ${bottomPoints[i + 1].x},${bottomPoints[i + 1].y}`;
            }

            // 添加左侧曲线
            for (let i = 0; i < leftPoints.length - 1; i++) {
                const cp1x = leftPoints[i].x;
                const cp1y = leftPoints[i].y - (leftPoints[i].y - leftPoints[i + 1].y) / 3;
                const cp2x = leftPoints[i + 1].x;
                const cp2y = leftPoints[i + 1].y + (leftPoints[i].y - leftPoints[i + 1].y) / 3;
                pathData += ` C${cp1x},${cp1y} ${cp2x},${cp2y} ${leftPoints[i + 1].x},${leftPoints[i + 1].y}`;
            }

            // 闭合路径
            pathData += ` Z`;

            // 更新SVG路径
            pathRef.value.setAttribute('d', pathData);
        };

        // 开始果冻动画
        const startJellyAnimation = () => {
            if (isAnimating.value) return;
            isAnimating.value = true;
            lastTime.value = performance.now();

            // 应用初始冲击力
            applyImpulse();

            // 开始动画循环
            const animate = (time: number) => {
                const deltaTime = Math.min(time - lastTime.value, 30) / 16; // 限制最大帧时间差，并转换为16ms为基准
                lastTime.value = time;

                updatePoints(deltaTime);
                updatePath();

                // 检查是否所有点都基本恢复原位
                const allSettled = points.value.every(p =>
                    Math.abs(p.x - p.ox) < 0.3 &&
                    Math.abs(p.y - p.oy) < 0.3 &&
                    Math.abs(p.vx) < 0.03 &&
                    Math.abs(p.vy) < 0.03
                );

                if (allSettled) {
                    isAnimating.value = false;
                    resetPoints();
                    if (animationId.value !== null) {
                        cancelAnimationFrame(animationId.value);
                        animationId.value = null;
                    }
                } else {
                    animationId.value = requestAnimationFrame(animate);
                }
            };

            if (animationId.value !== null) {
                cancelAnimationFrame(animationId.value);
            }

            animationId.value = requestAnimationFrame(animate);

            // 确保动画最终会停止
            setTimeout(() => {
                if (animationId.value !== null) {
                    cancelAnimationFrame(animationId.value);
                    isAnimating.value = false;
                    resetPoints();
                }
            }, duration.value);
        };

        // 应用冲击力 - 改进冲击力模式使其更自然
        const applyImpulse = () => {
            if (!buttonRef.value) return;

            const width = buttonRef.value.offsetWidth;
            const height = buttonRef.value.offsetHeight;
            const centerX = width / 2;
            const centerY = height / 2;

            // 给每个点施加冲击力
            points.value.forEach(point => {
                // 计算点到中心的向量
                const dx = point.ox - centerX;
                const dy = point.oy - centerY;
                const distance = Math.sqrt(dx * dx + dy * dy);
                const maxDistance = Math.sqrt(centerX * centerX + centerY * centerY);
                // 移除未使用的变量
                // const normalizedDistance = distance / maxDistance;

                // 初始化力
                let forceX = 0;
                let forceY = 0;

                // 顶部和底部的挤压效果
                if (point.oy === 0) {
                    // 顶部点 - 中心点向上挤压最强
                    const distFromCenter = Math.abs(point.ox - centerX) / centerX;
                    forceY = -25 * intensity.value * (1 - Math.pow(distFromCenter, 2));
                    // 添加一些水平力让它更自然
                    forceX = (point.ox > centerX ? 1 : -1) * 5 * intensity.value * (1 - distFromCenter);
                } else if (point.oy === height) {
                    // 底部点 - 中心点向下挤压最强
                    const distFromCenter = Math.abs(point.ox - centerX) / centerX;
                    forceY = 20 * intensity.value * (1 - Math.pow(distFromCenter, 2));
                    // 添加一些水平力
                    forceX = (point.ox > centerX ? 1 : -1) * 3 * intensity.value * (1 - distFromCenter);
                }

                // 左右两侧的挤压效果
                if (point.ox === 0) {
                    // 左侧点
                    const distFromCenter = Math.abs(point.oy - centerY) / centerY;
                    forceX = -15 * intensity.value * (1 - Math.pow(distFromCenter, 2));
                } else if (point.ox === width) {
                    // 右侧点
                    const distFromCenter = Math.abs(point.oy - centerY) / centerY;
                    forceX = 15 * intensity.value * (1 - Math.pow(distFromCenter, 2));
                }

                // 添加随机性使动画更自然
                const randomFactor = 0.85 + Math.random() * 0.3;

                // 施加冲击力
                point.vx += forceX * randomFactor;
                point.vy += forceY * randomFactor;
            });
        };

        // 更新弹簧点位置 - 添加时间因子使动画更平滑
        const updatePoints = (deltaTime: number) => {
            points.value.forEach(point => {
                // 计算恢复力 (弹簧力)
                const dx = point.ox - point.x;
                const dy = point.oy - point.y;

                // 应用弹簧力 - 添加非线性弹性使其更像果冻
                const distortion = Math.sqrt(dx * dx + dy * dy);
                const springFactor = jellyConfig.stiffness * (1 + distortion * 0.03);

                const fx = dx * springFactor;
                const fy = dy * springFactor;

                // 应用阻尼力
                point.vx *= Math.pow(1 - jellyConfig.damping, deltaTime);
                point.vy *= Math.pow(1 - jellyConfig.damping, deltaTime);

                // 更新速度
                point.vx += fx / jellyConfig.mass * deltaTime;
                point.vy += fy / jellyConfig.mass * deltaTime;

                // 更新位置
                point.x += point.vx * deltaTime;
                point.y += point.vy * deltaTime;

                // 添加弹跳效果 - 当点接近原始位置时轻微反弹
                if (Math.abs(dx) < 1 && Math.abs(dy) < 1 && Math.random() > 0.9) {
                    point.vx *= -jellyConfig.bounce * 0.1;
                    point.vy *= -jellyConfig.bounce * 0.1;
                }
            });
        };

        // 重置所有点到初始位置
        const resetPoints = () => {
            if (!buttonRef.value) return;

            points.value.forEach(point => {
                point.x = point.ox;
                point.y = point.oy;
                point.vx = 0;
                point.vy = 0;
            });

            updatePath();
        };

        // 点击处理
        const handleClick = (event: MouseEvent) => {
            isActive.value = true;

            // 开始果冻动画
            startJellyAnimation();

            // 发出点击事件
            emit('click', event);

            // 短暂延迟后重置按钮状态
            setTimeout(() => {
                isActive.value = false;
            }, 150);
        };

        // 监听props变化
        watch(() => intensity.value, (newVal) => {
            jellyConfig.stiffness = 0.08 * newVal;
            jellyConfig.bounce = 1.3 * newVal;
        });

        onMounted(() => {
            // 添加一个小延迟确保DOM已完全渲染
            setTimeout(() => {
                initPoints();
                updatePath();
            }, 50);

            // 监听窗口大小变化
            window.addEventListener('resize', handleResize);
        });

        onBeforeUnmount(() => {
            if (animationId.value !== null) {
                cancelAnimationFrame(animationId.value);
            }
            window.removeEventListener('resize', handleResize);
        });

        return {
            containerRef,
            buttonRef,
            pathRef,
            isActive,
            handleClick,
            buttonStyle,
            colorLight,
            colorDark,
            ...toRefs(props)
        };
    }
});
</script>

<style scoped>
.jelly-container {
    position: relative;
    display: inline-block;
}

.jelly-button {
    position: relative;
    cursor: pointer;
    outline: none;
    border: none;
    padding: 16px 32px;
    font-size: 16px;
    font-weight: 600;
    border-radius: 12px;
    min-width: 150px;
    overflow: visible;
    background: transparent;
    transition: transform 0.1s;
    z-index: 1;
    display: flex;
    justify-content: center;
    align-items: center;
}

.jelly-button:active {
    transform: scale(0.95);
}

.jelly-content {
    position: relative;
    z-index: 2;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
    width: 100%;
    color: white;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
    pointer-events: none;
    left: -19px;
    top: -5px;
}

.jelly-reflection {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 50%;
    background-color: transparent;
    border-radius: 8px 8px 0 0;
    pointer-events: none;
}

.jelly-svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 0;
    filter: drop-shadow(0 8px 20px var(--button-color-transparent));
    border-radius: 12px;
    overflow: visible;
}

/* 添加果冻感的阴影效果 */
.jelly-button::after {
    content: '';
    position: absolute;
    bottom: -5px;
    left: 10%;
    right: 10%;
    height: 10px;
    background: rgba(0, 0, 0, 0.1);
    border-radius: 50%;
    filter: blur(5px);
    z-index: -1;
    transition: all 0.3s ease;
}

.jelly-button:active::after {
    bottom: -2px;
    left: 15%;
    right: 15%;
    height: 5px;
}
</style>
