<template>
    <div class="blackhole-container" ref="containerRef">
        <canvas class="blackhole-canvas" ref="canvasRef" v-show="isActive"></canvas>
        <button class="blackhole-button" ref="buttonRef" @click="handleClick">
            {{ buttonText }}
        </button>
    </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, onMounted, onBeforeUnmount, watch } from 'vue';

interface StarParticle {
    x: number;
    y: number;
    size: number;
    color: string;
    distance: number; // 距离黑洞中心的距离
    angle: number;    // 相对于黑洞中心的角度
    radialVelocity: number; // 径向速度（向内）
    angularVelocity: number; // 角速度（旋转）
    opacity: number;
    distortion: number; // 扭曲程度
}

interface AccretionDiskSegment {
    innerRadius: number;
    outerRadius: number;
    startAngle: number;
    endAngle: number;
    color: string;
    angularVelocity: number;
    opacity: number;
}

interface GravitationalLensRay {
    startX: number;
    startY: number;
    controlX: number;
    controlY: number;
    endX: number;
    endY: number;
    width: number;
    color: string;
    opacity: number;
}

export default defineComponent({
    name: 'BlackHoleButton',
    props: {
        buttonText: {
            type: String,
            default: '召唤黑洞'
        },
        duration: {
            type: Number,
            default: 6000 // 黑洞效果持续时间，毫秒
        },
        intensity: {
            type: Number,
            default: 1.0 // 效果强度
        }
    },
    emits: ['click'],
    setup(props, { emit }) {
        const containerRef = ref<HTMLDivElement | null>(null);
        const buttonRef = ref<HTMLButtonElement | null>(null);
        const canvasRef = ref<HTMLCanvasElement | null>(null);
        const isActive = ref(false);
        const ctx = ref<CanvasRenderingContext2D | null>(null);
        const animationId = ref<number | null>(null);

        // 黑洞中心位置
        const center = reactive({
            x: 0,
            y: 0
        });

        // 黑洞事件视界半径
        const eventHorizonRadius = ref(0);

        // 黑洞椭圆参数
        const ellipseParams = reactive({
            radiusX: 0,  // X轴半径
            radiusY: 0,  // Y轴半径
            rotation: 0, // 旋转角度
            offsetX: 0,  // 中心点X偏移
            offsetY: 0   // 中心点Y偏移
        });

        // 黑洞动画状态
        const animationState = reactive({
            phase: 0, // 0: 开始形成, 1: 完全形成, 2: 开始消失
            progress: 0, // 0-1 当前阶段的进度
            totalProgress: 0 // 0-1 整体进度
        });

        // 按钮尺寸
        const buttonSize = reactive({
            width: 0,
            height: 0
        });

        // 恒星粒子
        const stars = ref<StarParticle[]>([]);

        // 吸积盘片段
        const accretionDisk = ref<AccretionDiskSegment[]>([]);

        // 引力透镜光线
        const lensRays = ref<GravitationalLensRay[]>([]);

        // 初始化Canvas
        const initCanvas = () => {
            if (!canvasRef.value || !containerRef.value) return;

            const canvas = canvasRef.value;
            const container = containerRef.value;

            // 设置Canvas大小为容器的两倍，以便覆盖更大区域
            canvas.width = window.innerWidth * 2;
            canvas.height = window.innerHeight * 2;

            canvas.style.width = window.innerWidth + 'px';
            canvas.style.height = window.innerHeight + 'px';
            canvas.style.left = '50%';
            canvas.style.top = '50%';
            canvas.style.transform = 'translate(-50%, -50%)';

            // 获取渲染上下文
            ctx.value = canvas.getContext('2d');

            // 设置黑洞的中心位置为按钮的中心
            if (buttonRef.value) {
                const buttonRect = buttonRef.value.getBoundingClientRect();
                const buttonCenterX = buttonRect.left + buttonRect.width / 2;
                const buttonCenterY = buttonRect.top + buttonRect.height / 2;
                
                // 将按钮中心位置转换为canvas坐标
                center.x = buttonCenterX * 2; // 因为canvas宽度是窗口的2倍
                center.y = buttonCenterY * 2; // 因为canvas高度是窗口的2倍
                
                // 记录按钮尺寸，用于后续转换按钮为黑洞
                buttonSize.width = buttonRect.width * 2;
                buttonSize.height = buttonRect.height * 2;
            }
        };

        // 创建恒星粒子
        const createStars = (count: number) => {
            const newStars: StarParticle[] = [];
            const maxDistance = Math.max(window.innerWidth, window.innerHeight) * 1.5;

            for (let i = 0; i < count; i++) {
                // 随机生成恒星的角度和距离
                const angle = Math.random() * Math.PI * 2;
                const distance = Math.random() * maxDistance;

                // 计算恒星的位置
                const x = center.x + Math.cos(angle) * distance;
                const y = center.y + Math.sin(angle) * distance;

                // 随机生成恒星的大小、颜色和速度
                const size = Math.random() * 2 + 1;
                const colors = [
                    '#ffffff', '#ffffdd', '#ffeecc', '#ddddff',
                    '#ccffff', '#ffccee', '#eeeeff'
                ];
                const color = colors[Math.floor(Math.random() * colors.length)];

                // 根据距离计算径向速度和角速度
                const distFactor = 1 - Math.min(1, distance / maxDistance);
                const radialVelocity = (0.1 + distFactor * 2) * props.intensity;
                const angularVelocity = (0.0005 + distFactor * 0.001) * props.intensity;

                newStars.push({
                    x,
                    y,
                    size,
                    color,
                    distance,
                    angle,
                    radialVelocity,
                    angularVelocity,
                    opacity: 1,
                    distortion: 0
                });
            }

            return newStars;
        };

        // 创建吸积盘
        const createAccretionDisk = () => {
            const newDisk: AccretionDiskSegment[] = [];
            const rings = 7; // 吸积盘的环数
            const minRadius = eventHorizonRadius.value * 1.3;
            const maxRadius = eventHorizonRadius.value * 3.5;

            // 冷色调颜色数组 - 每个环一种颜色
            const colors = [
                '#1a237e', // 深蓝
                '#3949ab', // 蓝色
                '#0d47a1', // 深蓝
                '#01579b', // 深青蓝
                '#006064', // 深青色
                '#004d40', // 深绿青
                '#0b5394'  // 靛蓝
            ];

            // 创建环带
            for (let i = 0; i < rings; i++) {
                // 计算环的内半径和外半径，使环更薄
                const ringWidth = (maxRadius - minRadius) / rings * 0.7; // 减小环宽度
                const gap = (maxRadius - minRadius) / rings * 0.3; // 环之间的间隙
                const innerRadius = minRadius + (ringWidth + gap) * i;
                const outerRadius = innerRadius + ringWidth;

                // 确保每个环有不同的颜色
                const color = colors[i % colors.length];
                
                // 角速度，内环旋转更快
                const angularVelocity = (0.02 + (rings - i) * 0.005) * props.intensity;

                // 每个环带只有一个完整的圆形
                newDisk.push({
                    innerRadius,
                    outerRadius,
                    startAngle: 0,
                    endAngle: Math.PI * 2,
                    color,
                    angularVelocity,
                    opacity: 0.5 // 降低不透明度使其更微妙
                });
            }

            return newDisk;
        };

        // 创建引力透镜光线
        const createLensRays = (count: number) => {
            const newRays: GravitationalLensRay[] = [];
            const maxDistance = Math.max(window.innerWidth, window.innerHeight) * 0.7;

            for (let i = 0; i < count; i++) {
                // 光线起始点
                const startAngle = Math.random() * Math.PI * 2;
                const startDistance = eventHorizonRadius.value * 1.5 + Math.random() * eventHorizonRadius.value * 5;
                const startX = center.x + Math.cos(startAngle) * startDistance;
                const startY = center.y + Math.sin(startAngle) * startDistance;

                // 光线终点
                const endDistance = Math.random() * maxDistance;
                const endAngle = startAngle + (Math.random() - 0.5) * 0.5;
                const endX = center.x + Math.cos(endAngle) * endDistance;
                const endY = center.y + Math.sin(endAngle) * endDistance;

                // 控制点 - 用于创建弯曲光线
                const distortFactor = Math.random() * 0.3 + 0.7;
                const controlDistance = startDistance * 0.5;
                const controlAngle = startAngle + (Math.random() - 0.5) * 2 * Math.PI * distortFactor;
                const controlX = center.x + Math.cos(controlAngle) * controlDistance;
                const controlY = center.y + Math.sin(controlAngle) * controlDistance;

                // 线宽
                const width = Math.random() * 1.5 + 0.5;

                // 颜色 - 主要是白色和淡蓝色
                const colors = ['rgba(255,255,255,0.4)', 'rgba(200,230,255,0.5)', 'rgba(230,240,255,0.3)'];
                const color = colors[Math.floor(Math.random() * colors.length)];

                newRays.push({
                    startX,
                    startY,
                    controlX,
                    controlY,
                    endX,
                    endY,
                    width,
                    color,
                    opacity: Math.random() * 0.4 + 0.2
                });
            }

            return newRays;
        };

        // 启动黑洞动画
        const startBlackHoleAnimation = () => {
            if (isActive.value) return;

            isActive.value = true;
            animationState.phase = 0;
            animationState.progress = 0;
            animationState.totalProgress = 0;

            // 初始化黑洞
            if (!canvasRef.value) return;
            initCanvas();

            // 设置黑洞事件视界初始半径
            const baseRadius = Math.min(window.innerWidth, window.innerHeight) * 0.08;
            eventHorizonRadius.value = baseRadius;
            
            // 设置椭圆参数
            ellipseParams.radiusX = buttonSize.width / 2; // 初始X半径为按钮宽度的一半
            ellipseParams.radiusY = buttonSize.height / 2; // 初始Y半径为按钮高度的一半
            ellipseParams.rotation = Math.random() * Math.PI / 4; // 随机旋转角度
            
            // 隐藏按钮
            if (buttonRef.value) {
                buttonRef.value.style.opacity = '0';
                buttonRef.value.style.pointerEvents = 'none';
            }

            // 创建恒星
            stars.value = createStars(300);

            // 创建引力透镜光线
            lensRays.value = createLensRays(40);

            // 创建吸积盘
            accretionDisk.value = createAccretionDisk();

            // 开始动画循环
            let lastTime = performance.now();
            const startTime = lastTime;

            const animate = (currentTime: number) => {
                const deltaTime = currentTime - lastTime;
                lastTime = currentTime;
                const elapsedTime = currentTime - startTime;

                // 更新动画状态
                const phaseDuration = props.duration / 3;  // 三个阶段，每个阶段占三分之一
                animationState.totalProgress = Math.min(1, elapsedTime / props.duration);

                if (elapsedTime < phaseDuration) {
                    // 阶段0：黑洞形成
                    animationState.phase = 0;
                    animationState.progress = elapsedTime / phaseDuration;
                } else if (elapsedTime < phaseDuration * 2) {
                    // 阶段1：黑洞稳定
                    animationState.phase = 1;
                    animationState.progress = (elapsedTime - phaseDuration) / phaseDuration;
                } else {
                    // 阶段2：黑洞消失
                    animationState.phase = 2;
                    animationState.progress = (elapsedTime - phaseDuration * 2) / phaseDuration;
                }

                // 更新和绘制
                updateBlackHole(deltaTime);
                drawBlackHole();

                // 继续动画或结束
                if (elapsedTime < props.duration) {
                    animationId.value = requestAnimationFrame(animate);
                } else {
                    isActive.value = false;

                    // 清空画布
                    if (ctx.value && canvasRef.value) {
                        ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);
                    }

                    if (animationId.value !== null) {
                        cancelAnimationFrame(animationId.value);
                        animationId.value = null;
                    }
                    
                    // 显示按钮
                    if (buttonRef.value) {
                        buttonRef.value.style.opacity = '1';
                        buttonRef.value.style.pointerEvents = 'auto';
                    }
                }
            };

            animationId.value = requestAnimationFrame(animate);
        };

        // 更新黑洞状态
        const updateBlackHole = (deltaTime: number) => {
            // 更新动画进度
            if (animationState.phase === 0) {
                // 形成阶段
                animationState.progress += deltaTime / 1000 * 0.5; // 黑洞形成速度
                if (animationState.progress >= 1) {
                    animationState.progress = 1;
                    animationState.phase = 1;
                }
            } else if (animationState.phase === 1) {
                // 稳定阶段
                const stableDuration = props.duration * 0.6 / 1000; // 稳定阶段持续时间（秒）
                animationState.progress += deltaTime / 1000 / stableDuration;
                if (animationState.progress >= 1) {
                    animationState.progress = 0;
                    animationState.phase = 2;
                }
            } else {
                // 消失阶段
                animationState.progress += deltaTime / 1000 * 0.7; // 黑洞消失速度
                if (animationState.progress >= 1) {
                    // 动画结束
                    isActive.value = false;
                    if (animationId.value !== null) {
                        cancelAnimationFrame(animationId.value);
                        animationId.value = null;
                    }
                    return;
                }
            }

            // 计算总体进度
            if (animationState.phase === 0) {
                animationState.totalProgress = animationState.progress * 0.3;
            } else if (animationState.phase === 1) {
                animationState.totalProgress = 0.3 + animationState.progress * 0.4;
            } else {
                animationState.totalProgress = 0.7 + animationState.progress * 0.3;
            }

            // 根据进度更新黑洞参数
            const intensity = props.intensity;
            
            // 更新事件视界半径
            if (animationState.phase === 0) {
                // 形成阶段，从0增长到最大值
                eventHorizonRadius.value = buttonSize.width * 0.5 * animationState.progress * intensity;
            } else if (animationState.phase === 1) {
                // 稳定阶段，有脉动效果
                const pulseFactor = 1 + Math.sin(animationState.progress * Math.PI * 6) * 0.1;
                eventHorizonRadius.value = buttonSize.width * 0.5 * pulseFactor * intensity;
            } else {
                // 消失阶段，逐渐缩小
                eventHorizonRadius.value = buttonSize.width * 0.5 * (1 - animationState.progress) * intensity;
            }

            // 更新椭圆参数
            const baseRadiusX = eventHorizonRadius.value;
            const baseRadiusY = eventHorizonRadius.value;
            
            // 随着时间变化的椭圆扭曲
            const timeDistortion = Math.sin(Date.now() / 1000) * 0.2;
            
            // 根据阶段设置不同的椭圆参数
            if (animationState.phase === 0) {
                // 形成阶段，从圆形逐渐变成椭圆
                const eccentricity = animationState.progress * 0.3 * intensity; // 椭圆离心率
                ellipseParams.radiusX = baseRadiusX * (1 + eccentricity + timeDistortion);
                ellipseParams.radiusY = baseRadiusY * (1 - eccentricity + timeDistortion * 0.5);
                ellipseParams.rotation = animationState.progress * Math.PI / 6; // 旋转角度
                
                // 随机偏移中心点，但保持在按钮附近
                const maxOffset = buttonSize.width * 0.1 * animationState.progress;
                ellipseParams.offsetX = (Math.random() * 2 - 1) * maxOffset;
                ellipseParams.offsetY = (Math.random() * 2 - 1) * maxOffset;
            } else if (animationState.phase === 1) {
                // 稳定阶段，椭圆形态有波动
                const time = Date.now() / 1000;
                const waveX = Math.sin(time * 1.5) * 0.2 * intensity;
                const waveY = Math.cos(time * 2.0) * 0.15 * intensity;
                
                ellipseParams.radiusX = baseRadiusX * (1.3 + waveX + timeDistortion);
                ellipseParams.radiusY = baseRadiusY * (0.8 + waveY + timeDistortion * 0.5);
                ellipseParams.rotation += deltaTime / 1000 * 0.2; // 缓慢旋转
                
                // 中心点轻微波动
                const orbitRadius = buttonSize.width * 0.15;
                ellipseParams.offsetX = Math.cos(time * 0.7) * orbitRadius;
                ellipseParams.offsetY = Math.sin(time * 0.5) * orbitRadius;
            } else {
                // 消失阶段，椭圆逐渐变回圆形
                const eccentricity = (1 - animationState.progress) * 0.3 * intensity;
                ellipseParams.radiusX = baseRadiusX * (1 + eccentricity);
                ellipseParams.radiusY = baseRadiusY * (1 - eccentricity);
                ellipseParams.rotation -= deltaTime / 1000 * 0.5; // 反向旋转
                
                // 中心点逐渐回归原位
                ellipseParams.offsetX *= (1 - deltaTime / 1000 * 2);
                ellipseParams.offsetY *= (1 - deltaTime / 1000 * 2);
            }

            // 更新恒星粒子
            stars.value.forEach(star => {
                // 更新恒星位置和扭曲程度
                const distanceToCenter = Math.sqrt(
                    Math.pow(star.x - center.x - ellipseParams.offsetX, 2) + 
                    Math.pow(star.y - center.y - ellipseParams.offsetY, 2)
                );
                
                // 根据距离计算引力影响
                const gravitationalPull = Math.min(1, eventHorizonRadius.value * 5 / Math.max(1, distanceToCenter));
                star.distortion = gravitationalPull * intensity;
                
                // 更新角度和径向速度
                star.angle += star.angularVelocity * deltaTime / 1000;
                star.radialVelocity = gravitationalPull * 50 * intensity;
                
                // 如果恒星被吸入黑洞，重新生成在远处
                if (distanceToCenter < eventHorizonRadius.value || star.opacity <= 0) {
                    // 重置恒星位置到远处
                    const angle = Math.random() * Math.PI * 2;
                    const distance = Math.max(
                        window.innerWidth, 
                        window.innerHeight
                    ) * (1 + Math.random());
                    
                    star.x = center.x + Math.cos(angle) * distance;
                    star.y = center.y + Math.sin(angle) * distance;
                    star.opacity = 0.1 + Math.random() * 0.9;
                    star.distance = distance;
                    star.angle = angle;
                    star.distortion = 0;
                } else {
                    // 更新位置
                    const moveAngle = Math.atan2(
                        star.y - center.y - ellipseParams.offsetY, 
                        star.x - center.x - ellipseParams.offsetX
                    );
                    
                    star.x -= Math.cos(moveAngle) * star.radialVelocity * deltaTime / 1000;
                    star.y -= Math.sin(moveAngle) * star.radialVelocity * deltaTime / 1000;
                    
                    // 更新不透明度
                    if (distanceToCenter < eventHorizonRadius.value * 3) {
                        star.opacity -= deltaTime / 1000 * gravitationalPull;
                    }
                }
            });

            // 更新吸积盘
            accretionDisk.value.forEach(segment => {
                // 更新角度
                segment.startAngle += segment.angularVelocity * deltaTime / 1000;
                segment.endAngle += segment.angularVelocity * deltaTime / 1000;
                
                // 根据黑洞阶段更新不透明度
                if (animationState.phase === 0) {
                    segment.opacity = Math.min(1, segment.opacity + deltaTime / 1000 * 0.5);
                } else if (animationState.phase === 2) {
                    segment.opacity = Math.max(0, segment.opacity - deltaTime / 1000 * 0.7);
                }
            });

            // 更新引力透镜光线
            lensRays.value.forEach(ray => {
                // 根据黑洞阶段更新不透明度
                if (animationState.phase === 0) {
                    ray.opacity = Math.min(0.7, ray.opacity + deltaTime / 1000 * 0.3);
                } else if (animationState.phase === 2) {
                    ray.opacity = Math.max(0, ray.opacity - deltaTime / 1000 * 0.5);
                } else {
                    // 稳定阶段，光线不透明度波动
                    ray.opacity = 0.3 + Math.sin(Date.now() / 500) * 0.2;
                }
                
                // 更新控制点位置，使光线弯曲效果更动态
                const centerX = center.x + ellipseParams.offsetX;
                const centerY = center.y + ellipseParams.offsetY;
                const startToCenterX = centerX - ray.startX;
                const startToCenterY = centerY - ray.startY;
                const distanceToCenter = Math.sqrt(startToCenterX * startToCenterX + startToCenterY * startToCenterY);
                
                // 计算控制点位置，使光线围绕黑洞弯曲
                const bendFactor = Math.min(1, eventHorizonRadius.value * 3 / distanceToCenter) * intensity;
                const perpX = -startToCenterY / distanceToCenter;
                const perpY = startToCenterX / distanceToCenter;
                
                ray.controlX = (ray.startX + ray.endX) / 2 + perpX * distanceToCenter * bendFactor * 0.5;
                ray.controlY = (ray.startY + ray.endY) / 2 + perpY * distanceToCenter * bendFactor * 0.5;
            });
        };

        // 绘制黑洞及其效果
        const drawBlackHole = () => {
            if (!ctx.value || !canvasRef.value) return;

            const canvas = canvasRef.value;
            const context = ctx.value;

            // 清除画布
            context.clearRect(0, 0, canvas.width, canvas.height);

            // 确定整体不透明度
            let globalOpacity = 1;
            if (animationState.phase === 0) {
                globalOpacity = animationState.progress;
            } else if (animationState.phase === 2) {
                globalOpacity = 1 - animationState.progress;
            }

            // 设置画布的全局不透明度
            context.globalAlpha = globalOpacity;

            // 绘制背景的轻微扭曲效果
            drawSpaceDistortion(context);

            // 绘制引力透镜光线
            drawGravitationalLensing(context);

            // 绘制恒星粒子
            drawStars(context);

            // 绘制吸积盘
            // drawAccretionDisk(context);

            // 绘制黑洞事件视界
            drawEventHorizon(context);

            // 绘制整体辉光效果
            drawOverallGlow(context);

            // 重置全局不透明度
            context.globalAlpha = 1;
        };

        // 绘制空间扭曲效果
        const drawSpaceDistortion = (context: CanvasRenderingContext2D) => {
            try {
                // 确保半径始终为正值
                const innerRadius = Math.max(0.1, eventHorizonRadius.value * 1.5);
                const distortionRadius = Math.max(innerRadius + 0.1, eventHorizonRadius.value * 8);
                const centerX = center.x + ellipseParams.offsetX;
                const centerY = center.y + ellipseParams.offsetY;

                // 创建径向渐变
                const gradient = context.createRadialGradient(
                    centerX, centerY, innerRadius,
                    centerX, centerY, distortionRadius
                );

                gradient.addColorStop(0, 'rgba(0, 0, 0, 0.8)');
                gradient.addColorStop(0.3, 'rgba(20, 10, 40, 0.6)');
                gradient.addColorStop(0.7, 'rgba(10, 10, 30, 0.3)');
                gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

                context.fillStyle = gradient;
                
                // 绘制椭圆形扭曲区域
                context.save();
                context.translate(centerX, centerY);
                context.rotate(ellipseParams.rotation);
                
                // 确保缩放比例为正值
                const scaleX = Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, eventHorizonRadius.value) * 8;
                const scaleY = Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, eventHorizonRadius.value) * 8;
                context.scale(scaleX, scaleY);
                
                context.beginPath();
                context.arc(0, 0, distortionRadius, 0, Math.PI * 2);
                context.restore();
                context.fill();
            } catch (error) {
                console.error('Error in drawSpaceDistortion:', error);
                // 使用备用绘制方法
                context.fillStyle = 'rgba(10, 10, 30, 0.3)';
                context.beginPath();
                context.ellipse(
                    center.x + ellipseParams.offsetX,
                    center.y + ellipseParams.offsetY,
                    Math.max(0.1, Math.abs(ellipseParams.radiusX * 8)),
                    Math.max(0.1, Math.abs(ellipseParams.radiusY * 8)),
                    ellipseParams.rotation,
                    0,
                    Math.PI * 2
                );
                context.fill();
            }
        };

        // 绘制引力透镜效果
        const drawGravitationalLensing = (context: CanvasRenderingContext2D) => {
            lensRays.value.forEach(ray => {
                context.strokeStyle = ray.color;
                context.globalAlpha = ray.opacity * (1 - animationState.progress * 0.3);
                context.lineWidth = ray.width;

                context.beginPath();
                context.moveTo(ray.startX, ray.startY);
                context.quadraticCurveTo(ray.controlX, ray.controlY, ray.endX, ray.endY);
                context.stroke();
            });

            // 重置透明度
            context.globalAlpha = 1;
        };

        // 绘制恒星粒子
        const drawStars = (context: CanvasRenderingContext2D) => {
            stars.value.forEach(star => {
                if (star.opacity <= 0) return;

                context.fillStyle = star.color;
                context.globalAlpha = star.opacity;

                if (star.distortion > 0) {
                    // 绘制扭曲的恒星
                    context.save();
                    context.translate(star.x, star.y);

                    // 计算恒星相对黑洞的角度
                    const angleToCenter = Math.atan2(star.y - (center.y + ellipseParams.offsetY), star.x - (center.x + ellipseParams.offsetX));
                    context.rotate(angleToCenter);

                    // 椭圆形状变形
                    context.beginPath();
                    context.ellipse(0, 0, star.size * (1 + star.distortion), star.size, 0, 0, Math.PI * 2);
                    context.fill();

                    context.restore();
                } else {
                    // 绘制正常的恒星
                    context.beginPath();
                    context.arc(star.x, star.y, star.size, 0, Math.PI * 2);
                    context.fill();
                }
            });

            // 重置透明度
            context.globalAlpha = 1;
        };

        // 绘制吸积盘
        const drawAccretionDisk = (context: CanvasRenderingContext2D) => {
            accretionDisk.value.forEach(segment => {
                try {
                    context.globalAlpha = segment.opacity;

                    // 确保内半径和外半径为正值，且外半径大于内半径
                    const safeInnerRadius = Math.max(0.1, segment.innerRadius);
                    const safeOuterRadius = Math.max(safeInnerRadius + 0.1, segment.outerRadius);

                    // 创建辉光渐变
                    const glowGradient = context.createRadialGradient(
                        center.x + ellipseParams.offsetX, center.y + ellipseParams.offsetY, safeInnerRadius,
                        center.x + ellipseParams.offsetX, center.y + ellipseParams.offsetY, safeOuterRadius
                    );

                    // 解析颜色以提取RGB值
                    const hexToRgb = (hex: string) => {
                        const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
                        return result ? {
                            r: parseInt(result[1], 16),
                            g: parseInt(result[2], 16),
                            b: parseInt(result[3], 16)
                        } : { r: 0, g: 0, b: 0 };
                    };

                    const rgb = hexToRgb(segment.color);

                    // 添加内部更亮的光辉，但保持冷色调
                    glowGradient.addColorStop(0, `rgba(${rgb.r + 30}, ${rgb.g + 30}, ${rgb.b + 50}, 0.7)`);
                    glowGradient.addColorStop(0.5, segment.color);
                    glowGradient.addColorStop(1, `rgba(${rgb.r}, ${rgb.g}, ${rgb.b}, 0.1)`);

                    context.fillStyle = glowGradient;

                    // 绘制吸积盘环
                    context.save();
                    context.translate(center.x + ellipseParams.offsetX, center.y + ellipseParams.offsetY);
                    context.rotate(segment.startAngle); // 使环旋转
                    
                    // 确保缩放比例为正值
                    const scaleX = Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, eventHorizonRadius.value);
                    const scaleY = Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, eventHorizonRadius.value);
                    context.scale(scaleX, scaleY);
                    
                    context.beginPath();
                    context.arc(0, 0, safeOuterRadius, 0, Math.PI * 2);
                    context.arc(0, 0, safeInnerRadius, 0, Math.PI * 2, true);
                    context.closePath();
                    context.restore();
                    context.fill();
                } catch (error) {
                    console.error('Error in drawAccretionDisk:', error);
                    // 使用备用绘制方法
                    context.fillStyle = segment.color;
                    context.globalAlpha = segment.opacity * 0.5;
                    
                    // 确保所有半径都是正值
                    const outerRadiusX = Math.max(0.1, Math.abs(segment.outerRadius * (Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, eventHorizonRadius.value))));
                    const outerRadiusY = Math.max(0.1, Math.abs(segment.outerRadius * (Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, eventHorizonRadius.value))));
                    const innerRadiusX = Math.max(0.1, Math.abs(segment.innerRadius * (Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, eventHorizonRadius.value))));
                    const innerRadiusY = Math.max(0.1, Math.abs(segment.innerRadius * (Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, eventHorizonRadius.value))));
                    
                    // 绘制外圆
                    context.beginPath();
                    context.ellipse(
                        center.x + ellipseParams.offsetX,
                        center.y + ellipseParams.offsetY,
                        outerRadiusX,
                        outerRadiusY,
                        ellipseParams.rotation + segment.startAngle,
                        0,
                        Math.PI * 2
                    );
                    
                    // 绘制内圆（反方向）
                    context.ellipse(
                        center.x + ellipseParams.offsetX,
                        center.y + ellipseParams.offsetY,
                        innerRadiusX,
                        innerRadiusY,
                        ellipseParams.rotation + segment.startAngle,
                        0,
                        Math.PI * 2,
                        true
                    );
                    context.fill();
                }
            });

            // 重置透明度
            context.globalAlpha = 1;
        };

        // 绘制黑洞事件视界
        const drawEventHorizon = (context: CanvasRenderingContext2D) => {
            try {
                const radius = Math.max(0.1, eventHorizonRadius.value);
                const centerX = center.x + ellipseParams.offsetX;
                const centerY = center.y + ellipseParams.offsetY;

                // 创建径向渐变
                const gradient = context.createRadialGradient(
                    centerX, centerY, 0.1, // 确保内半径为正值
                    centerX, centerY, radius
                );

                gradient.addColorStop(0, 'rgba(0, 0, 0, 1)');
                gradient.addColorStop(0.7, 'rgba(5, 0, 10, 1)');
                gradient.addColorStop(0.9, 'rgba(20, 0, 40, 0.9)');
                gradient.addColorStop(1, 'rgba(0, 0, 0, 0.8)');

                context.fillStyle = gradient;
                
                // 绘制椭圆形黑洞
                context.save();
                context.translate(centerX, centerY);
                context.rotate(ellipseParams.rotation);
                
                // 确保缩放比例为正值
                const scaleX = Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, radius);
                const scaleY = Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, radius);
                context.scale(scaleX, scaleY);
                
                context.beginPath();
                context.arc(0, 0, radius, 0, Math.PI * 2);
                context.restore();
                context.fill();
            } catch (error) {
                console.error('Error in drawEventHorizon:', error);
                // 使用备用绘制方法，确保所有参数都是正值
                context.fillStyle = 'rgba(0, 0, 0, 1)';
                
                // 确保所有半径都是正值
                const radiusX = Math.max(0.1, Math.abs(ellipseParams.radiusX));
                const radiusY = Math.max(0.1, Math.abs(ellipseParams.radiusY));
                
                context.beginPath();
                context.ellipse(
                    center.x + ellipseParams.offsetX,
                    center.y + ellipseParams.offsetY,
                    radiusX,
                    radiusY,
                    ellipseParams.rotation,
                    0,
                    Math.PI * 2
                );
                context.fill();
            }
        };

        // 绘制整体辉光效果
        const drawOverallGlow = (context: CanvasRenderingContext2D) => {
            try {
                const innerRadius = Math.max(0.1, eventHorizonRadius.value);
                const glowRadius = Math.max(innerRadius + 0.1, eventHorizonRadius.value * 3);
                const centerX = center.x + ellipseParams.offsetX;
                const centerY = center.y + ellipseParams.offsetY;

                // 创建径向渐变
                const gradient = context.createRadialGradient(
                    centerX, centerY, innerRadius,
                    centerX, centerY, glowRadius
                );

                gradient.addColorStop(0, 'rgba(80, 20, 120, 0.3)');
                gradient.addColorStop(0.3, 'rgba(60, 15, 90, 0.2)');
                gradient.addColorStop(0.7, 'rgba(40, 10, 60, 0.1)');
                gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

                context.fillStyle = gradient;
                
                // 绘制椭圆形辉光
                context.save();
                context.translate(centerX, centerY);
                context.rotate(ellipseParams.rotation);
                
                // 确保缩放比例为正值
                const scaleX = Math.max(0.1, ellipseParams.radiusX) / Math.max(0.1, eventHorizonRadius.value) * 3;
                const scaleY = Math.max(0.1, ellipseParams.radiusY) / Math.max(0.1, eventHorizonRadius.value) * 3;
                context.scale(scaleX, scaleY);
                
                context.beginPath();
                context.arc(0, 0, glowRadius, 0, Math.PI * 2);
                context.restore();
                context.fill();
            } catch (error) {
                console.error('Error in drawOverallGlow:', error);
                // 使用备用绘制方法，确保所有参数都是正值
                context.fillStyle = 'rgba(40, 10, 60, 0.1)';
                
                // 确保所有半径都是正值
                const radiusX = Math.max(0.1, Math.abs(ellipseParams.radiusX * 3));
                const radiusY = Math.max(0.1, Math.abs(ellipseParams.radiusY * 3));
                
                context.beginPath();
                context.ellipse(
                    center.x + ellipseParams.offsetX,
                    center.y + ellipseParams.offsetY,
                    radiusX,
                    radiusY,
                    ellipseParams.rotation,
                    0,
                    Math.PI * 2
                );
                context.fill();
            }
        };

        // 点击处理函数
        const handleClick = (event: MouseEvent) => {
            if (isActive.value) return;

            // 启动黑洞动画
            startBlackHoleAnimation();

            // 发出点击事件
            emit('click', event);
        };

        // 根据窗口大小调整Canvas
        const resizeCanvas = () => {
            if (isActive.value) {
                initCanvas();
            }
        };

        // 生命周期钩子
        onMounted(() => {
            window.addEventListener('resize', resizeCanvas);
        });

        onBeforeUnmount(() => {
            window.removeEventListener('resize', resizeCanvas);

            if (animationId.value !== null) {
                cancelAnimationFrame(animationId.value);
                animationId.value = null;
            }
        });

        // 返回模板需要的变量和方法
        return {
            containerRef,
            buttonRef,
            canvasRef,
            isActive,
            handleClick
        };
    }
});
</script>

<style scoped>
.blackhole-container {
    position: relative;
    display: inline-block;
    margin-left: -30px;
}

.blackhole-button {
    position: relative;
    cursor: pointer;
    padding: 10px 10px;
    font-size: 16px;
    font-weight: 600;
    color: #ffffff;
    background: linear-gradient(135deg, #30103f, #1e0a29);
    border: 1px solid #6b33a0;
    border-radius: 12px;
    box-shadow: 0 0 15px rgba(111, 33, 173, 0.5);
    transition: all 0.3s ease;
    z-index: 10; /* 确保按钮在canvas上方 */
    min-width: 150px;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);
    outline: none;
    overflow: hidden;
}

.blackhole-button:before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(135deg, #7c33c7, #3c1966);
    opacity: 0;
    transition: opacity 0.3s ease;
    z-index: -1;
}

.blackhole-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 0 20px rgba(132, 56, 204, 0.6);
}

.blackhole-button:hover:before {
    opacity: 1;
}

.blackhole-button:active {
    transform: translateY(1px);
    box-shadow: 0 0 8px rgba(111, 33, 173, 0.4);
}

.blackhole-canvas {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1; /* 将z-index设置为1，确保在按钮后面 */
}
</style>
