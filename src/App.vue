<template>
    <div id="app">
        <simple-draw class="my-dc"/>
    </div>
</template>

<script>
    //TODO:
    //Faster flood fill algorithm
    //Color picker when clicking on selected color
    //Better icons

    import SimpleDraw from './components/SimpleDraw.vue'

    export default {
        name: 'App',
        components: {
            SimpleDraw
        }
    }
</script>

<style>
    html, body {
        background-color: black;
        margin: 0;
        padding: 0;
        height: 100%;
    }

    * {
        /*cursor: url('assets/brush.png'), grab !important;*/
    }

    .my-dc {
        width: calc(100% - 20px);
        height: calc(100% - 20px);
        margin: 10px;
        background-color: rgba(255, 255, 255, 0.15);
        border-radius: 10px;
        overflow: hidden;
        position: fixed;
    }

    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        color: #2c3e50;
        height: 100%;
    }
</style>
