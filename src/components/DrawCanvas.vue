<template>
    <div class="draw-canvas">
        <div class="canvas-container" @keypress="handleKey">
            <canvas class="canvas"
                    ref="canvas"
                    :width="canvasWidth"
                    :height="canvasHeight"
                    @touchstart="startTouch"
                    @touchmove="moveTouch"
                    @touchend="endTouch"
                    @mousedown="startMove"
                    @mousemove="move"
                    @mouseup="endMove"
                    @resize="canvasResize"
            >Canvas is not supported
            </canvas>
        </div>
        <div class="controls">
            <div class="selected-color" :style="`background-color: ${activeRgb}`"></div>
            <div class="colors">
                <div v-for="(row, i) in colors" class="color-row" :key="i">
                    <div v-for="[r,g,b] in row"
                         class="single-color"
                         :style="`background-color: rgb(${r},${g},${b})`"
                         @click="selectColor(r,g,b)"
                         :key="r+g+b"></div>
                </div>
            </div>
            <div class="tools">
                <div class="brush" :active="this.activeTool === 'brush'" @click="selectTool('brush')"></div>
                <div class="fill" :active="this.activeTool === 'fill'" @click="selectTool('fill')"></div>
                <div class="erase" :active="this.activeTool === 'erase'" @click="selectTool('erase')"></div>
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
    import LineCommand from "@/js/LineCommand";
    import ClearCommand from "@/js/ClearCommand";


    export default {
        name: 'DrawCanvas',
        props: {},
        data: () => ({
            activeTool: 'brush',
            activeColor: [0, 0, 0],
            activeBrush: 13,
            colors: [
                [
                    [255, 255, 255],
                    [193, 193, 193],
                    [239, 19, 11],
                    [255, 113, 0],
                    [255, 228, 0],
                    [0, 204, 0],
                    [0, 178, 255],
                    [35, 31, 211],
                    [163, 0, 186],
                    [211, 124, 170],
                    [160, 82, 45]
                ], [
                    [0, 0, 0],
                    [76, 76, 76],
                    [116, 11, 7],
                    [194, 56, 0],
                    [232, 162, 0],
                    [0, 85, 16],
                    [0, 86, 158],
                    [14, 8, 101],
                    [85, 0, 105],
                    [167, 85, 116],
                    [99, 48, 13],
                ]
            ],
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
        },
        beforeDestroy() {
            console.log("Destroying");
            window.removeEventListener('resize', this.canvasResize);
            document.removeEventListener('keypress', this.handleKey);
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
                        default:
                            console.warn("No case set");
                            break;
                    }
                }
                this.animationId = requestAnimationFrame(this.render);
            },
            startTool(x, y, finger) {
                let line = [[x, y]];
                let drawable = {
                    type: 'line',
                    brush: this.activeBrush,
                    color: this.activeRgb,
                    line,
                };
                finger.drawable = drawable;
                this.drawables.push(drawable);
            },
            moveTool(x, y, finger) {
                finger.drawable.line.push([x, y]);
            },
            endTool(x, y, finger) {
                finger.drawable.line.push([x, y]);
                CommandStack.addCommand(new LineCommand(this.drawables, finger.drawable));
                delete finger.drawable;
            },
            clearCanvas() {
                CommandStack.executeCommand(new ClearCommand(this.drawables));
            },
            selectColor(r, g, b) {
                this.activeColor = [r, g, b];
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
                let x = e.pageX - this.$refs.canvas.offsetLeft;
                let y = e.pageY - this.$refs.canvas.offsetTop;
                this.startTool(x, y, finger);
            },
            move(e, fingerIndex = 0) {
                if (this.fingers[fingerIndex] && this.fingers[fingerIndex].down) {
                    let x = e.pageX - this.$refs.canvas.offsetLeft;
                    let y = e.pageY - this.$refs.canvas.offsetTop;
                    this.moveTool(x, y, this.fingers[fingerIndex]);
                }
            },
            endMove(e, fingerIndex = 0) {
                let x = e.pageX - this.$refs.canvas.offsetLeft;
                let y = e.pageY - this.$refs.canvas.offsetTop;
                this.endTool(x, y, this.fingers[fingerIndex]);
                if (this.fingers[fingerIndex])
                    this.fingers[fingerIndex].down = false;
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

    .selected-color {
        height: 50px;
        width: 50px;
        border-radius: 11px;
        margin: 5px 0;
    }

    .colors {
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
