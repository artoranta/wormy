<template>
    <div class="grid-container">
        <div class="game-header">
            <a-button :disabled="state !== 2" class="reset-button" @click="restart">
                New Game
            </a-button>
            <div class="score-container">
                <h1 style="color: #47494E; margin-bottom: 0;">
                    {{ score }}
                </h1>
            </div>
            <a-radio-group
                v-model="speed"
                class="speed-radio"
                button-style="solid"
                @change="move"
            >
                <a-radio-button :value="5" :disabled="state !== 0 && speed !== 5">
                    <a-icon :type="icons[5]" />
                </a-radio-button>
                <a-radio-button :value="7" :disabled="state !== 0 && speed !== 7">
                    <a-icon :type="icons[7]" />
                </a-radio-button>
                <a-radio-button :value="10" :disabled="state !== 0 && speed !== 10">
                    <a-icon :type="icons[10]" />
                </a-radio-button>
            </a-radio-group>
        </div>
        <div class="grid">
            <div v-for="x in 10" :key="x" class="row">
                <div v-for="y in 10" :key="y" class="tile" />
            </div>
            <!--<div v-for="y in grid" :key="y[0].join('.')" class="row">
                <div v-for="xy in y" :key="xy.join('.')" class="tile">
                    <a-icon v-if="head.join('.') === xy.join('.')" class="worm-eyes" type="pause" />
                    <div v-if="worm.map(wxy => wxy.join('.')).includes(xy.join('.'))" class="worm" />
                    <div v-if="apple.join('.') === xy.join('.')" class="apple" />
                </div>
            </div>-->
            <canvas
                id="myCanvas"
                width="360px"
                height="360px"
                style="position: absolute;"
            >
                Your browser does not support the canvas element.
            </canvas>
            <input
                ref="controlInput"
                class="control-input"
                onblur="this.focus()"
                autofocus
                inputmode="none"
                @keyup="setDir($event)"
            >
        </div>
        <div class="control-buttons">
            <button class="rotate-360 right-left-button" style="padding-right: 30px;" @click="setDir({ keyCode: 37 })">
                <a-icon class="icons" type="left" />
            </button>
            <div class="lower-buttons">
                <button class="rotate-360 up-down-button" style="padding-bottom: 30px;" @click="setDir({ keyCode: 38 })">
                    <a-icon class="icons" type="up" />
                </button>
                <button class="rotate-180 up-down-button" style="padding-bottom: 30px;" @click="setDir({ keyCode: 40 })">
                    <a-icon class="icons" type="up" />
                </button>
            </div>
            <button class="rotate-180 right-left-button" style="padding-right: 30px;" @click="setDir({ keyCode: 39 })">
                <a-icon class="icons" type="left" />
            </button>
        </div>
    </div>
</template>

<script>
export default {
    layout: 'default',
    data () {
        return {
            size: 10,
            grid: Array(10)
                .fill()
                .map((_u, y) => Array(10)
                    .fill()
                    .map((_u, x) => [y, x])),
            worm: Array(3).fill().map((_u, n) => [5, n]),
            /* left | top | right | bottom */
            dir: 1,
            speed: 7,
            apple: [5, 5],
            state: 0,
            score: 0,
            lastDir: 1,
            nextDir: 1,
            interval: null,
            icons: {
                5: 'bug',
                7: 'car',
                10: 'rocket'
            }
        }
    },
    computed: {
        head () {
            return this.worm[this.worm.length - 1]
        },
        length () {
            return this.worm.length
        }
    },
    watch: {
        apple (val) {
            this.fullName = val + ' ' + this.lastName
        }
    },
    mounted () {
        this.focusInput()
        const c = document.getElementById('myCanvas')
        const ctx = c.getContext('2d')
        this.vueCanvas = ctx
    },
    created () {
        // this.move()
        this.$nextTick(() => {
            this.updateCanvas()
            this.move()
        })
    },
    methods: {
        move () {
            clearInterval(this.interval)
            this.interval = setInterval(() => {
                if (this.state !== 1) {
                    return
                }
                let next
                switch (this.dir) {
                case 0:
                    this.lastDir = 0
                    next = [this.head[0], (this.head[1] + (this.size - 1)) % this.size]
                    break
                case 1:
                    this.lastDir = 1
                    next = [(this.head[0] + (this.size - 1)) % this.size, this.head[1]]
                    break
                case 2:
                    this.lastDir = 2
                    next = [this.head[0], (this.head[1] + 1) % this.size]
                    break
                case 3:
                    this.lastDir = 3
                    next = [(this.head[0] + 1) % this.size, this.head[1]]
                    break
                default:
                    next = [this.head[0], (this.head[1] + 1) % this.size]
                }
                if (this.worm.map(wxy => wxy.join('.')).slice(1).includes(next.join('.')) || this.length === this.size * this.size) {
                    this.state = 2
                } else {
                    this.moveHead(this.head, next)
                    this.worm.push(next)
                    if (this.head.join('.') === this.apple.join('.')) {
                        this.score++
                        let random = this.generate()
                        while (this.worm.map(wxy => wxy.join('.')).includes(random.join('.'))) {
                            random = this.generate()
                        }
                        this.removePart(this.apple)
                        this.apple = random
                        this.addPart(...this.apple, '#cb2724')
                    } else {
                        this.removePart(...this.worm[0])
                        this.worm.shift()
                    }
                }
                if (Math.abs(this.dir - this.nextDir) !== 2) {
                    this.dir = this.nextDir
                }
                // this.updateCanvas()
            }, 1000 / this.speed)
        },
        focusInput () {
            this.$refs.controlInput.focus()
        },
        removePart (y, x) {
            this.vueCanvas.clearRect(x * 36, y * 36, 36, 36)
        },
        addPart (y, x, color = '#22a417', w = 36, h = 36) {
            this.vueCanvas.beginPath()
            this.vueCanvas.fillStyle = color
            this.vueCanvas.fillRect(x * 36, y * 36, w, h)
        },
        moveHead (oldHead, newHead) {
            // Head.
            this.removePart(...oldHead)
            this.addPart(...oldHead)
            this.addPart(...newHead)
            // Eyes.
            this.addPart(newHead[0] + 6 / 36, newHead[1] + 13 / 36, '#b3d9aa', 2, 10)
            this.addPart(newHead[0] + 6 / 36, newHead[1] + 21 / 36, '#b3d9aa', 2, 10)
        },
        updateCanvas () {
            this.vueCanvas.clearRect(0, 0, 360, 360)

            this.worm.forEach(([y, x]) => {
                this.addPart(y, x)
                if (x === this.head[1] && y === this.head[0]) {
                    this.addPart(y + 6 / 36, x + 13 / 36, '#b3d9aa', 2, 10)
                    this.addPart(y + 6 / 36, x + 21 / 36, '#b3d9aa', 2, 10)
                }
            })

            this.addPart(...this.apple, '#cb2724')
        },
        setDir ({ keyCode }) {
            if ((keyCode < 37 && keyCode > 40) || this.state === 2) {
                return
            }
            this.state = 1
            if (Math.abs(this.lastDir - (keyCode - 37)) !== 2) {
                if (keyCode - 37 === this.lastDir) {
                    this.nextDir = keyCode - 37
                } else {
                    this.dir = keyCode - 37
                    this.nextDir = keyCode - 37
                }
            } else {
                this.nextDir = keyCode - 37
            }
        },
        generate () {
            return [Math.floor(Math.random() * this.size), Math.floor(Math.random() * this.size)]
        },
        restart () {
            this.state = 0
            this.score = 0
            this.lastDir = 1
            this.dir = 1
            this.nextDir = 1
            this.worm = Array(3).fill().map((_u, n) => [5, n])
            this.apple = [5, 5]
            this.updateCanvas()
        }
    }
}
</script>

<style lang="scss">
    .grid-container {
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        flex-direction: column;
        min-height: 100vh;
        background: #eaeaea;
        background: linear-gradient(90deg, #ffffff, #eeeeee 100%);
    }

    .grid {
        position: relative;
        width: 360px;
        height: 360px;
        display: flex;
        flex-direction: column;
        box-shadow: 3px 3px 8px #a3a4a8;
    }

    .row {
        width: 100%;
        flex-grow: 1;
        display: flex;
    }

    .tile {
        position: relative;
        background-color: #d7d7d7;
        border: 1px solid #bbbbbb;
        flex-grow: 1;
    }

    .worm {
        position: absolute;
        height: 100%;
        width: 100%;
        background-color: #22a417;
    }

    .apple {
        position: absolute;
        height: 100%;
        width: 100%;
        background-color: #cb2724;
    }

    .worm-eyes {
        position: absolute;
        color: #b3d9aa;
        height: 100%;
        width: 100%;
        z-index: 1000;
    }

    .control-input {
        position: absolute;
        opacity: 0;
        top: 0;
        left: 0;
        width: 360px;
    }

    .game-header {
        width: 360px;
        height: 60px;
        justify-content: space-between;
        align-items: center;
        display: flex;
        flex-direction: row;
        margin-bottom: 5px;
    }

    .reset-button {
        min-width: 140px;
        box-shadow: 2px 2px 5px 0 #cdcdcd;
        cursor: pointer;
    }

    .score-container {
        text-align: center;
        color: #47494E;
    }

    .speed-radio {
        min-width: 140px;
        cursor: pointer;
        display: flex;
        justify-content: flex-end;
    }

    .control-buttons {
        display: flex;
        flex-direction: row;
        align-items: center;
        padding: 20px;
    }

    .lower-buttons {
        display: flex;
        flex-direction: column;
        margin-left: -73px;
        margin-right: -73px;
    }

    .icons {
        font-size: 18px;
        color: #717171;
        z-index: 100;
    }

    .up-down-button {
        width: 150px;
        height: 100px;
        margin: -1px;
        cursor: pointer;
        border-radius: 4px;
        border: 1px solid #e8e8e8;
        background-color: #e8e8e8;
        clip-path: polygon(0% 0, 100% 0, 100% 25px, 50% 100%, 0 25px);
    }

    .up-down-button:before {
        content: " ";
        position: absolute;
        z-index: -1;
        top: 1px;
        left: 1px;
        right: 1px;
        bottom: 1px;
        background-color: #fcfcfc;
        clip-path: polygon(0% 0, 100% 0, 100% 23px, 50% 100%, 0 23px);
    }

    .right-left-button {
        width: 100px;
        height: 150px;
        margin: -1px;
        cursor: pointer;
        border-radius: 4px;
        border: 1px solid #e8e8e8;
        background-color: #e8e8e8;
        clip-path: polygon(0% 0, 25px 0, 100% 50%, 25px 100%, 0 100%);
    }

    .right-left-button:before {
        content: " ";
        position: absolute;
        z-index: -1;
        top: 1px;
        left: 1px;
        right: 1px;
        bottom: 1px;
        background-color: #fcfcfc;
        clip-path: polygon(0% 0, 23px 0, 100% 50%, 23px 100%, 0 100%);
    }

    .rotate-180 {
        transform: rotate(180deg)
    }

    .rotate-360 {
        transform: rotate(360deg)
    }

</style>
