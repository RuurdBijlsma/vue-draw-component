<template>
    <div class="draw-canvas">
        <div class="canvas-container" @keypress="handleKey">
            <canvas class="canvas"
                    ref="canvas"
                    :width="canvasWidth"
                    :height="canvasHeight"
                    @touchstart="startTouch"
                    @mousedown="startMove"
                    @resize="canvasResize"
            >Canvas is not supported
            </canvas>
        </div>
        <div class="controls">
            <div class="colors">
                <div class="selected-color" :style="`background-color: ${activeRgb}`"></div>
                <div class="color-grid">
                    <div v-for="(row, i) in colors" class="color-row" :key="i">
                        <div v-for="[r,g,b,a] in row"
                             class="single-color"
                             :style="`background-color: rgb(${r},${g},${b})`"
                             @click="selectColor(r,g,b,a)"
                             :key="r+g+b+a"></div>
                    </div>
                </div>
            </div>
            <div class="tools">
                <div class="brush" :active="this.activeTool === 'brush'" @click="selectTool('brush')"></div>
                <div class="fill" :active="this.activeTool === 'fill'" @click="selectTool('fill')"></div>
                <div class="erase" :active="this.activeTool === 'eraser'" @click="selectTool('eraser')"></div>
            </div>
            <div class="brush-sizes">
                <div v-for="size in brushSizes" :key="size"
                     :active="activeBrush === size"
                     @click="selectBrush(size)"
                     :style="`
                        height: ${size}px;
                        width: ${size}px;
                        background-color: ${activeRgb};
                        margin-top: ${5 + 25 - size / 2}px;
                    `"
                ></div>
            </div>
            <div class="clear-canvas" @click="clearCanvas">
            </div>
        </div>
    </div>
</template>

<script>
    import CommandStack from "@/js/CommandStack";
    import DrawableCommand from "@/js/DrawableCommand";
    import ClearCommand from "@/js/ClearCommand";


    export default {
        name: 'DrawCanvas',
        props: {},
        data: () => ({
            activeTool: 'brush',
            activeColor: [0, 0, 0, 255],
            activeBrush: 13,
            colors: [
                [
                    [255, 255, 255, 255],
                    [193, 193, 193, 255],
                    [239, 19, 11, 255],
                    [255, 113, 0, 255],
                    [255, 228, 0, 255],
                    [0, 204, 0, 255],
                    [0, 178, 255, 255],
                    [35, 31, 211, 255],
                    [163, 0, 186, 255],
                    [211, 124, 170, 255],
                    [160, 82, 45]
                ], [
                    [0, 0, 0, 255],
                    [76, 76, 76, 255],
                    [116, 11, 7, 255],
                    [194, 56, 0, 255],
                    [232, 162, 0, 255],
                    [0, 85, 16, 255],
                    [0, 86, 158, 255],
                    [14, 8, 101, 255],
                    [85, 0, 105, 255],
                    [167, 85, 116, 255],
                    [99, 48, 13, 255],
                ]
                ,],
            brushSizes: [7, 13, 25, 50],
            fingers: {},
            context: null,
            drawables: [],
            animationId: -1,
            canvasWidth: 10,
            canvasHeight: 10,
            commands: [],
            commandIndex: 0,
        }),
        mounted() {
            let canvas = this.$refs.canvas;
            this.context = canvas.getContext('2d');
            this.canvasResize();
            window.addEventListener('resize', this.canvasResize, false);

            this.animationId = requestAnimationFrame(this.render);

            console.log(CommandStack);
            document.addEventListener('keypress', this.handleKey, false);
            document.addEventListener('mousemove', this.move, false);
            document.addEventListener('touchmove', this.moveTouch, false);
            document.addEventListener('mouseup', this.endMove, false);
            document.addEventListener('touchend', this.endTouch, false);
        },
        beforeDestroy() {
            console.log("Destroying");
            window.removeEventListener('resize', this.canvasResize);
            document.removeEventListener('keypress', this.handleKey);
            document.removeEventListener('mousemove', this.move);
            document.removeEventListener('touchmove', this.moveTouch);
            document.removeEventListener('mouseup', this.endMove);
            document.removeEventListener('touchend', this.endTouch);
            cancelAnimationFrame(this.animationId);
            CommandStack.reset();
        },
        methods: {
            render() {
                this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight);
                this.context.lineCap = 'round';
                this.context.lineJoin = 'round';

                for (let drawable of this.drawables) {
                    switch (drawable.type) {
                        case 'line':
                            this.context.beginPath();
                            this.context.strokeStyle = drawable.color;
                            this.context.lineWidth = drawable.brush;
                            for (let [x, y] of drawable.line)
                                this.context.lineTo(x, y);
                            this.context.stroke();
                            break;
                        case 'square':
                            this.context.fillStyle = drawable.color;
                            this.context.fillRect(...drawable.position, ...drawable.shape);
                            break;
                        case 'fill':
                            this.context.drawImage(drawable.canvas, 0, 0);
                            break;
                        default:
                            console.warn("No case set");
                            break;
                    }
                }
                this.animationId = requestAnimationFrame(this.render);
            },
            startTool(x, y, finger) {
                if (this.activeTool === 'brush') {
                    let line = [[x, y]];
                    let drawable = {
                        type: 'line',
                        brush: this.activeBrush,
                        color: this.activeRgb,
                        line,
                    };
                    finger.drawable = drawable;
                    this.drawables.push(drawable);
                } else if (this.activeTool === 'eraser') {
                    let line = [[x, y]];
                    let drawable = {
                        type: 'line',
                        brush: this.activeBrush,
                        color: 'rgb(255,255,255)',
                        line,
                    };
                    finger.drawable = drawable;
                    this.drawables.push(drawable);
                } else if (this.activeTool === 'fill') {
                    console.log("fill start");
                }
            },
            moveTool(x, y, finger) {
                if (this.activeTool === 'brush' || this.activeTool === 'eraser') {
                    finger.drawable.line.push([x, y]);
                } else if (this.activeTool === 'fill') {
                    console.log("fill move");
                }
            },
            endTool(x, y, finger) {
                if (this.activeTool === 'brush' || this.activeTool === 'eraser') {
                    finger.drawable.line.push([x, y]);
                    CommandStack.addCommand(new DrawableCommand(this.drawables, finger.drawable));
                    delete finger.drawable;
                } else if (this.activeTool === 'fill') {
                    this.fill(x, y, this.activeColor);
                    console.log("fill done");
                }
            },
            getNeighbours(width, height, x, y) {
                return [
                    [x + 1, y + 0],//right
                    [x - 1, y + 0],//left
                    [x + 0, y + 1],//bottom
                    [x + 0, y - 1],//top
                ].filter(([nX, nY]) =>
                    !(nX === x && nY === y) &&
                    nX >= 0 && nX < width &&
                    nY >= 0 && nY < height
                );
            },
            colorEquals(colorA, colorB) {
                return colorA[0] === colorB[0] &&
                    colorA[1] === colorB[1] &&
                    colorA[2] === colorB[2] &&
                    colorA[3] === colorB[3];
            },
            colorDistance(colorA, colorB) {
                return Math.abs(colorA[0] - colorB[0]) +
                    Math.abs(colorA[1] - colorB[1]) +
                    Math.abs(colorA[2] - colorB[2]) +
                    Math.abs(colorA[3] - colorB[3]);
            },
            fill(startX, startY, color = this.activeColor) {
                let image = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight);
                let breadth = [[startX, startY]];
                let visited = {};
                let i = (startY * this.canvasWidth + startX) * 4;
                let startColor = image.data.slice(i, i + 4);
                console.log(startColor, color)
                visited[startX] = {};
                visited[startX][startY] = true;
                let fillCanvas = document.createElement('canvas');
                fillCanvas.width = this.canvasWidth;
                fillCanvas.height = this.canvasHeight;
                let fillContext = fillCanvas.getContext('2d');
                let fillImage = fillContext.getImageData(0, 0, this.canvasWidth, this.canvasHeight);
                while (breadth.length > 0) {
                    let neighbours = [];
                    for (let [x, y] of breadth) {
                        let subNeighbours = this.getNeighbours(this.canvasWidth, this.canvasHeight, x, y);
                        // console.log(subNeighbours, x, y);
                        for (let [nX, nY] of subNeighbours) {
                            let i = (nY * this.canvasWidth + nX) * 4;
                            let neighbourColor = image.data.slice(i, i + 4);
                            let neighbourColor2 = fillImage.data.slice(i, i + 4);
                            if (this.colorEquals(neighbourColor, startColor) && //Neighbour is not different than start color
                                !this.colorEquals(neighbourColor2, color)) { //Neighbour isn't already filled in with fill color
                                //Yes, fill here
                                fillImage.data[i] = color[0];
                                fillImage.data[i + 1] = color[1];
                                fillImage.data[i + 2] = color[2];
                                fillImage.data[i + 3] = color[3];
                                // console.log('fill', nX, nY);
                                neighbours.push([nX, nY]);
                            }
                        }
                    }
                    breadth = neighbours;
                }
                fillContext.putImageData(fillImage, 0, 0);
                let drawable = {
                    type: 'fill',
                    canvas: fillCanvas,
                };
                // this.drawables.push(drawable);
                CommandStack.executeCommand(new DrawableCommand(this.drawables, drawable));
                console.log(fillCanvas);
            },
            clearCanvas() {
                CommandStack.executeCommand(new ClearCommand(this.drawables));
            },
            selectColor(r, g, b, a = 255) {
                this.activeColor = [r, g, b, a];
            },
            selectTool(tool) {
                this.activeTool = tool;
            },
            selectBrush(brushSize) {
                this.activeBrush = brushSize;
            },
            handleKey(e) {
                e.preventDefault();
                if (e.code === 'KeyZ' && e.ctrlKey && e.shiftKey)
                    CommandStack.redo();
                else if (e.code === 'KeyZ' && e.ctrlKey)
                    CommandStack.undo();
            },
            startMove(e, fingerIndex = 0) {
                if (!this.fingers[fingerIndex])
                    this.fingers[fingerIndex] = {};
                let finger = this.fingers[fingerIndex];
                finger.down = true;
                let {top, left} = this.$refs.canvas.getBoundingClientRect();
                let x = e.pageX - left;
                let y = e.pageY - top;
                this.startTool(x, y, finger);
            },
            move(e, fingerIndex = 0) {
                if (this.fingers[fingerIndex] && this.fingers[fingerIndex].down) {
                    let {top, left} = this.$refs.canvas.getBoundingClientRect();
                    let x = e.pageX - left;
                    let y = e.pageY - top;
                    this.moveTool(x, y, this.fingers[fingerIndex]);
                }
            },
            endMove(e, fingerIndex = 0) {
                if (this.fingers[fingerIndex]) {
                    if (this.fingers[fingerIndex].down) {
                        let {top, left} = this.$refs.canvas.getBoundingClientRect();
                        let x = e.pageX - left;
                        let y = e.pageY - top;
                        this.endTool(x, y, this.fingers[fingerIndex]);
                    }
                    this.fingers[fingerIndex].down = false;
                }
            },
            startTouch(e) {
                for (let touch of e.touches)
                    this.startMove(touch, touch.identifier)
            },
            moveTouch(e) {
                for (let touch of e.touches)
                    this.startMove(touch, touch.identifier)
            },
            endTouch(e) {
                for (let touch of e.touches)
                    this.startMove(touch, touch.identifier)
            },
            canvasResize() {
                console.log('resize');
                let canvas = this.$refs.canvas;
                let {width, height} = canvas.getBoundingClientRect();
                this.canvasWidth = width;
                this.canvasHeight = height;
            },
        }
        ,
        computed: {
            activeRgb() {
                let [r, g, b] = this.activeColor;
                return `rgb(${r},${g},${b})`
            }
        }
    }
</script>

<style scoped>
    .canvas {
        width: 100%;
        height: 100%;
    }

    .draw-canvas {
        min-width: 610px;
    }

    .canvas-container {
        background-color: white;
        width: 100%;
        height: calc(100% - 80px);
    }

    .controls {
        height: 60px;
        padding: 10px;
        display: flex;
        justify-content: space-between;
    }

    .colors {
        display: flex;
    }

    .selected-color {
        height: 50px;
        width: 50px;
        border-radius: 11px;
        margin: 5px 0;
    }

    .color-grid {
        display: flex;
        flex-direction: column;
        margin: 5px 10px;
    }

    .color-row {
        display: flex;
    }

    .single-color {
        width: 21px;
        height: 21px;
        border-radius: 50%;
        margin: 2px;
        cursor: pointer;
    }


    .tools {
        display: flex;
        margin-right: 10px;
    }

    .tools > div {
        margin: 10px 5px;
        height: 40px;
        width: 40px;
        background-size: cover;
        background-repeat: no-repeat;
        background-position: center;
        border-radius: 50%;
        cursor: pointer;
        transition: background-color 0.2s;
    }

    .tools > div:hover {
        background-color: rgba(255, 255, 255, 0.3);
    }

    .tools > div:active {
        box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.5);
    }

    .tools > div[active="true"] {
        box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.7);
    }

    .brush {
        background-image: url(../assets/brush.png);
    }

    .fill {
        background-image: url(../assets/fill.png);
    }

    .erase {
        background-image: url(../assets/eraser.png);
    }

    .brush-sizes {
        display: flex;
    }

    .brush-sizes > div {
        margin: 5px;
        border-radius: 50%;
        cursor: pointer;
    }

    .brush-sizes > div[active="true"] {
        box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.7);
    }

    .clear-canvas {
        margin: 5px;
        height: 50px;
        width: 50px;
        background-size: cover;
        background-repeat: no-repeat;
        background-position: center;
        border-radius: 50%;
        cursor: pointer;
        transition: background-color 0.2s;
        background-image: url(../assets/trash.png);
    }

    .clear-canvas:hover {
        background-color: rgba(255, 255, 255, 0.3);
    }

    .clear-canvas:active {
        background-color: maroon;
    }
</style>
