<template>
    <div class="canvasbox">
        <template>
            <div class="canvasbox">
                <a-space>
                    <a-button type="primary" @click="startDrawing">绘制</a-button>
                    <a-button type="primary" @click="togglePencilMode">编辑</a-button>
                    <a-button type="primary" @click="saveCanvas">保存</a-button>
                    <a-button type="primary" @click="loadCanvas">加载</a-button>
                    <a-button type="primary" @click="clearCanvas">清除</a-button>
                    <a-button type="primary" @click="prevImage">上一张</a-button> <!-- 上一张按钮 -->
                    <a-button type="primary" @click="nextImage">下一张</a-button> <!-- 下一张按钮 -->
                </a-space>
                <canvas ref="canvasRef" class="canvas" width="500" height="300"></canvas>

                <!-- 缩略图展示区域 -->
                <div class="thumbnail-container">
                    <!-- 使用 v-for 遍历显示所有缩略图 -->
                    <img v-for="(imageUrl, index) in imageUrls" :key="index" :src="imageUrl" alt="Image Thumbnail"
                        class="thumbnail" :class="{ selected: index === currentImageIndex }"
                        @click="selectImage(index)" />
                </div>
            </div>
        </template>


    </div>
</template>


<script setup>
    import { ref, onMounted } from 'vue';
    import { fabric } from 'fabric';

    // Vue ref 来引用 canvas 元素
    const canvasRef = ref(null);// canvas 元素的引用
    const fabricObj = ref(null);// fabric 实例
    const drawingEnabled = ref(false);// 是否允许绘制
    let isPencilMode = ref(false);// 是否是画笔模式
    const roofPoints = ref([]);// 屋顶的点集合
    const lines = ref([]);// 线条集合
    const pointArr = ref([]);// 点集合
    const savedCanvasData = ref({});// 保存的画布数据
    const roof = ref();// 屋顶对象

    // 图片资源及索引
    const imageUrls = ref([
        '/img/cover1.png',
        '/img/cover2.png'
    ]);
    const currentImageIndex = ref(0);// 当前图片索引

    // 加载当前图片
    const loadImage = () => {
        fabric.Image.fromURL(imageUrls.value[currentImageIndex.value], (img) => {
            // 获取画布的尺寸
            const canvasWidth = fabricObj.value.getWidth();
            const canvasHeight = fabricObj.value.getHeight();

            // 计算图片的比例并缩放
            const scaleX = canvasWidth / img.width;
            const scaleY = canvasHeight / img.height;
            const scale = Math.min(scaleX, scaleY);  // 选择较小的比例，确保图片按比例缩放，不会被拉伸

            // 设置图片属性
            img.set({
                left: 0,
                top: 0,
                selectable: false, // 禁用选择（不允许拖动）
                evented: false, // 禁用事件（不响应拖动、点击等）
                scaleX: scale,  // 设置缩放比例
                scaleY: scale,  // 设置缩放比例
            });

            fabricObj.value.clear(); // 清除画布上的所有元素
            fabricObj.value.add(img); // 添加新图片
            fabricObj.value.renderAll(); // 重新渲染画布
        });
    };

    // 初始化事件绑定
    const initializeEvents = () => {
        fabricObj.value.on({
            'mouse:down': handleMouseDown,
            'mouse:move': handleMouseMove,
            'mouse:dblclick': handleMouseDblClick,
        });
    };

    // 上一张图片
    const prevImage = () => {
        currentImageIndex.value = (currentImageIndex.value - 1 + imageUrls.value.length) % imageUrls.value.length;
        loadImage();
        // loadCanvas();

    };

    // 下一张图片
    const nextImage = () => {
        currentImageIndex.value = (currentImageIndex.value + 1) % imageUrls.value.length;
        loadImage();
        // loadCanvas();
    };

    // 选择图片（点击缩略图）
    const selectImage = (index) => {
        currentImageIndex.value = index;
        loadImage(); // 切换图片
    };

    // 开始绘制
    const startDrawing = () => {
        drawingEnabled.value = true;
        console.log('绘制开始！');
    };

    // 切换画笔模式
    const togglePencilMode = () => {
        isPencilMode.value = !isPencilMode.value;
        fabricObj.value.isDrawingMode = isPencilMode.value;
        fabricObj.value.freeDrawingBrush.width = 2; // 画笔宽度
        fabricObj.value.freeDrawingBrush.color = 'red'; // 画笔颜色
        console.log(isPencilMode.value ? '画笔模式启动' : '画笔模式停止');
    };

    // 清除画布
    const clearCanvas = () => {
        fabricObj.value.clear();
        console.log('画布已清除！');
        drawingEnabled.value = false;
    };

    // 保存画布数据
    const saveCanvas = () => {
        savedCanvasData.value = fabricObj.value.toJSON();
        pointArr.value.push(pointArr.value[0]); // 将第一个点添加为最后一个点，确保封闭
        savedCanvasData.value.objects.map(item => {
            if (item.type === 'polyline') item.points = pointArr.value;
        });
    };

    // 加载保存的画布数据
    const loadCanvas = () => {
        if (savedCanvasData.value.objects && savedCanvasData.value.objects.length > 0) {
            fabricObj.value.loadFromJSON(savedCanvasData.value, () => {
                // 加载后，禁用所有图片对象的拖动
                fabricObj.value.getObjects('image').forEach((img) => {
                    img.set({
                        selectable: false, // 禁止选择
                        evented: false, // 禁止事件（如拖动、点击）
                    });
                });
                fabricObj.value.renderAll(); // 重新渲染画布
            });
            console.log('画布数据已加载！');
        } else {
            alert('没有保存的数据可加载！');
        }
    };

    // 鼠标按下事件，记录点击的坐标
    const handleMouseDown = (e) => {
        if (drawingEnabled.value) {
            const mouseFrom = {
                x: e.pointer.x,
                y: e.pointer.y,
            };

            // 保存坐标
            pointArr.value.push({ x: mouseFrom.x, y: mouseFrom.y });

            // 画线
            const point = {
                x: mouseFrom.x,
                y: mouseFrom.y,
            };
            roofPoints.value.push(point);
            drawLine(point);
        }
    };

    // 鼠标移动事件，更新线条
    const handleMouseMove = (e) => {
        if (lines.value.length && drawingEnabled.value) {
            const x = e.pointer.x;
            const y = e.pointer.y;
            lines.value[lines.value.length - 1].set({
                x2: x,
                y2: y,
            });
            fabricObj.value.renderAll();
        }
    };

    // 鼠标双击事件，结束绘制
    const handleMouseDblClick = () => {
        if (drawingEnabled.value) {
            fabricObj.value.off('mouse:down');
            lines.value.forEach((item) => fabricObj.value.remove(item));
            createRoof();
            fabricObj.value.add(roof.value);
            fabricObj.value.renderAll();

            resetDrawingData();
            console.log('绘制结束！pointArr', pointArr.value);
        }
    };

    // 创建屋顶（闭合的多边形）
    const createRoof = () => {
        const left = Math.min(...roofPoints.value.map((p) => p.x));
        const top = Math.min(...roofPoints.value.map((p) => p.y));
        roofPoints.value.push(roofPoints.value[0]); // 闭合多边形

        roof.value = new fabric.Polyline(roofPoints.value, {
            fill: '',
            strokeWidth: 2,
            stroke: 'red',
            left: left,
            top: top,
        });
    };

    // 绘制红色线条
    const drawLine = (point) => {
        const points = [point.x, point.y, point.x, point.y];
        lines.value.push(new fabric.Line(points, {
            strokeWidth: 2,
            selectable: false,
            stroke: 'red',
        }));
        fabricObj.value.add(lines.value[lines.value.length - 1]);
    };

    // 重置绘图数据
    const resetDrawingData = () => {
        roofPoints.value.length = 0;
        lines.value.length = 0;
        drawingEnabled.value = false;
    };

    // 初始化应用
    const initializeApp = () => {
        loadImage(); // 加载初始图片
        initializeEvents();
    };

    // 页面挂载时初始化
    onMounted(() => {
        fabricObj.value = new fabric.Canvas(canvasRef.value); // 通过 ref 获取 canvas 元素
        initializeApp();
    });
</script>

<style scoped>
    .canvas {
        display: flex;
        border: 1px solid black;
        box-sizing: content-box;
        margin-top: 10px;
    }

    .thumbnail-container {
        margin-top: 10px;
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 10px;
        /* 缩略图之间的间隔 */
        flex-wrap: wrap;
        /* 容器换行 */
    }

    .thumbnail {
        width: 100px;
        height: 60px;
        object-fit: cover;
        /* 保持图片比例，裁剪图片填充容器 */
        border: 1px solid #ccc;
        cursor: pointer;
        transition: border 0.3s ease;
    }

    .thumbnail.selected {
        border: 3px solid blue;
        /* 选中的缩略图添加蓝色边框 */
    }
</style>
