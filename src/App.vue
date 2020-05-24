<template>
    <div id="app">
        <draw-canvas class="my-dc"/>
    </div>
</template>

<script>
    //TODO:
    //Flood fill
    //Eraser
    //Better icons

    import DrawCanvas from './components/DrawCanvas.vue'

    export default {
        name: 'App',
        components: {
            DrawCanvas
        }
    }
</script>

<style>
    html, body {
        background-color: black;
        margin: 0px;
        padding: 0px;
        height:100%;
    }

    .my-dc {
        width: calc(100% - 20px);
        height: calc(100% - 20px);
        margin:10px;
        background-color:rgba(255,255,255,0.15);
        border-radius:10px;
        overflow: hidden;
    }

    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        color: #2c3e50;
        height:100%;
    }
</style>
