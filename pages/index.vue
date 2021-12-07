<template>
    <div class="grid-container">
        <div class="stat-header">
            <div class="game-count">
                <!--{{ storage.games ? storage.games : 0 }}-->
                <a-slider
                    v-model="size"
                    :disabled="state !== 0"
                    style="width: 125px;"
                    :min="10"
                    :max="20"
                    :step="5"
                    :marks="marks"
                />
            </div>
            <div class="highscores">
                {{ Object.keys(icons).map(s => storage['highscore-' + size + '-' + s] || 0) }}
            </div>
        </div>
        <div class="game-header">
            <a-button :disabled="state === 0" class="reset-button" @click="restart">
                {{ state !== 2 ? 'Reset' : 'New Game' }}
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
                @change="start"
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
            <div v-for="x in size" :key="x" class="row">
                <div v-for="y in size" :key="y" class="tile" />
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
const namespace = 'wormy.dev'

export default {
    layout: 'default',
    data () {
        return {
            /* grid: this.getGrid(), */
            /* left | top | right | bottom */
            worm: Array(3).fill().map((_u, n) => [5, n]),
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
            },
            storage: {},
            size: 15,
            marks: {
                10: '10x10',
                15: '15x15',
                20: '20x20'
            }
        }
    },
    computed: {
        head () {
            return this.worm[this.worm.length - 1]
        },
        length () {
            return this.worm.length
        },
        pixels () {
            return 360 / this.size
        }
    },
    watch: {
        apple (value) {
            if (value.join('.') !== [this.size + 1, this.size + 1].join('.')) {
                this.addPart(this.apple, '#cb2724')
            }
        },
        state (value) {
            if (value === 1) {
                this.count('games')
            }
        },
        size () {
            this.restart()
        }
    },
    mounted () {
        this.focusInput()
        const c = document.getElementById('myCanvas')
        const ctx = c.getContext('2d')
        this.vueCanvas = ctx
    },
    created () {
        this.$nextTick(() => {
            this.count('games', 'get')
            this.count('visits')
            this.updateCanvas()
            Object.keys(this.icons).forEach(s => this.getHighscore(this.size, s))
            this.start()
        })
    },
    methods: {
        /*
        getGrid () {
            return Array(this.size)
                .fill()
                .map((_u, y) => Array(this.size)
                    .fill()
                    .map((_u, x) => [y, x]))
        },
         */
        start () {
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
                if (this.worm.map(wxy => wxy.join('.')).slice(1).includes(next.join('.'))) {
                    this.state = 2
                    this.saveScore(this.size, this.speed, this.score)
                } else {
                    this.moveHead(this.head, next)
                    this.worm.push(next)
                    if (this.head.join('.') === this.apple.join('.')) {
                        this.score++
                        if (this.length !== this.size * this.size) {
                            let random = this.generate()
                            while (this.worm.map(wxy => wxy.join('.')).includes(random.join('.'))) {
                                random = this.generate()
                            }
                            this.apple = random
                        } else {
                            // No more space for apple.
                            this.apple = [this.size + 1, this.size + 1]
                            this.state = 2
                            this.saveScore(this.size, this.speed, this.score)
                            this.win()
                        }
                    } else {
                        if (this.head.join('.') !== this.worm[0].join('.')) {
                            this.removePart(this.worm[0])
                        }
                        this.worm.shift()
                    }
                }
                if (Math.abs(this.dir - this.nextDir) !== 2) {
                    this.dir = this.nextDir
                }
            }, 1000 / this.speed)
        },
        count (key, endpoint = 'hit') {
            const xhr = new XMLHttpRequest()
            xhr.open('GET', `https://api.countapi.xyz/${endpoint}/${namespace}/${key}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                this.$set(this.storage, key, value)
            }
            xhr.send()
        },
        getHighscore (size, speed) {
            const xhr = new XMLHttpRequest()
            const key = `highscore-${size}-${speed}`
            xhr.open('GET', `https://api.countapi.xyz/get/${namespace}/${key}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                this.$set(this.storage, key, value)
                if (!this.storage[key] && this.storage[key] !== 0) {
                    this.create(key)
                }
            }
            xhr.send()
        },
        saveScore (size, speed, score) {
            const key = `highscore-${size}-${speed}`
            const currentValue = this.storage[key] || 0
            if (score > currentValue) {
                const xhr = new XMLHttpRequest()
                xhr.open('GET', `https://api.countapi.xyz/set/${namespace}/${key}?value=${score}`)
                xhr.responseType = 'json'
                xhr.onload = ({ target }) => {
                    const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                    this.$set(this.storage, key, value)
                }
                xhr.send()
            }
        },
        create (key) {
            const xhr = new XMLHttpRequest()
            xhr.open('GET', `https://api.countapi.xyz/create?namespace=${namespace}&key=${key}&value=0&enable_reset=1`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                console.log(target.response)
            }
            xhr.send()
        },
        focusInput () {
            this.$refs.controlInput.focus()
        },
        win () {
            const factor = this.size / 10
            const square = this.worm.filter(([y, x]) => y < 10 && x < 10)

            for (let i = 0; i < this.worm.length; i++) {
                this.addPart(this.worm[i])
                setTimeout(() => {
                    this.removePart(this.worm[i], this.pixels, this.pixels)
                }, 20 / factor * i)
                setTimeout(() => {
                    this.addPart(this.worm[i], '#cb2724')
                }, 2000 + 20 / factor * i)
                setTimeout(() => {
                    this.removePart(this.worm[i], this.pixels, this.pixels)
                }, 4500 + 20 / factor * i)
            }

            const egdes = [[0, 0], [0, 1], [0, 8], [0, 9], [1, 0], [1, 9], [8, 0], [8, 9], [9, 0], [9, 1], [9, 8], [9, 9]]
            const smile = [[1, 3], [1, 6], [2, 3], [2, 6], [3, 3], [3, 6], [4, 3], [4, 6], [6, 1], [6, 8], [7, 2], [7, 7], [8, 3], [8, 4], [8, 5], [8, 6]]

            for (let i = 0; i < square.length; i++) {
                setTimeout(() => {
                    if (!egdes.map(yx => yx.join('.')).includes(square[i].join('.'))) {
                        this.addPart(square[i].map(c => factor * c), '#eeda1c', this.pixels * factor, this.pixels * factor)
                    }
                    if (smile.map(yx => yx.join('.')).includes(square[i].join('.'))) {
                        this.addPart(square[i].map(c => factor * c), '#000000', this.pixels * factor, this.pixels * factor)
                    }
                }, 6000 + 30 * i)
            }
        },
        removePart ([y, x], w = this.pixels, h = this.pixels) {
            this.vueCanvas.clearRect(x * this.pixels, y * this.pixels, w, h)
        },
        addPart ([y, x], color = '#22a417', w = this.pixels, h = this.pixels) {
            this.vueCanvas.beginPath()
            this.vueCanvas.fillStyle = color
            this.vueCanvas.fillRect(x * this.pixels, y * this.pixels, w, h)
        },
        drawEyes ([y, x]) {
            this.addPart([y + 1 / 6, x + 13 / 36], '#b3d9aa', this.pixels / 18, this.pixels / 3.6)
            this.addPart([y + 1 / 6, x + 21 / 36], '#b3d9aa', this.pixels / 18, this.pixels / 3.6)
        },
        moveHead (oldHead, newHead) {
            this.removePart(oldHead)
            this.addPart(oldHead)
            this.addPart(newHead)
            this.drawEyes(newHead)
        },
        updateCanvas () {
            this.removePart([0, 0], this.pixels * this.size, this.pixels * this.size)
            this.addPart(this.apple, '#cb2724')
            this.worm.forEach(([y, x]) => {
                this.addPart([y, x])
                if (x === this.head[1] && y === this.head[0]) {
                    this.drawEyes([y, x])
                }
            })
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
            Object.keys(this.icons).forEach(s => this.getHighscore(this.size, s))
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

    .stat-header {
        width: 360px;
        justify-content: space-between;
        align-items: center;
        display: flex;
        flex-direction: row;
        margin-top: 5px;
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
