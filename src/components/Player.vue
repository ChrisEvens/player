<template>
  <div class="player">
    <div class="player-inner">
      <div class="song-info">
        <h1 class="band">Jamie XX</h1>
        <p class="song-title" style="font-family: 'Rock 3D', cursive; font-size: 4em">Gosh</p>
      </div>
      <div class="audio">
        <audio ref="source" crossorigin="anonymous" src="~@/assets/Jamie_xx_Gosh.mp3" v-on:ended="pause" v-on:loadedmetadata="init"></audio>
        <div class="audio-wrapper">
          <canvas ref="myCanvas" v-on:mousedown="mousedown" v-on:mouseup="mouseup" width="500" height="500"></canvas>
          <div class="controls">
            <v-btn v-if="paused" :disabled="!ready" fab x-large color="#C4A29E" v-on:click="play"><v-icon color="white">mdi-play</v-icon></v-btn>
            <v-btn v-else fab x-large color="#C4A29E" v-on:click="pause"><v-icon color="white">mdi-pause</v-icon></v-btn>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data () {
    return {
      audioElement: {},
      source: {},
      audioCtx: {},
      analyser: {},
      c: {},
      ctx: {},
      audioData: new Array(128),
      size: 500,
      i: 0,
      requestId: '',
      paused: true,
      seeking: false,
      seekTo: 0,
      bass: [],
      mouseLocation: {},
      ready: false,
      currentRun: 1,
      gainNode: {},
    }
  },
  computed: {
    radius () {
      return this.size / 5
    },
    url () { return this.$store.state.showcase.songUrl },
  },
  methods: {
    init () {
      this.audioElement = this.$refs.source
      if (this.currentRun === 1) { // skip 0
        this.draw(this.audioData)
        document.onmousemove = (event) => {
          this.mouseLocation = { x: event.clientX, y: event.clientY }
        }
      } else if (this.currentRun === 2) { // skip zero
        // // this.audioElement = this.$refs.source
        this.audioCtx = new (window.AudioContext || window.webkitAudioContext)()
        this.analyser = this.audioCtx.createAnalyser()
        this.analyser.fftSize = 256
        this.source = this.audioCtx.createMediaElementSource(this.audioElement)
        this.gainNode = this.audioCtx.createGain()
        this.gainNode.gain.value = 0.01
        this.source.connect(this.analyser)
        this.analyser.connect(this.gainNode)
        this.gainNode.connect(this.audioCtx.destination)
        this.source.connect(this.audioCtx.destination)
        this.audioData = new Uint8Array(this.analyser.frequencyBinCount)
        this.analyser.getByteFrequencyData(this.audioData)
        this.start()
      } else if (this.currentRun > 2) {
        this.start()
      }
      this.currentRun++
    },
    getPosition (el) {
      var xPos = 0
      var yPos = 0

      while (el) {
        if (el.tagName === 'BODY') {
          // deal with browser quirks with body/window/document and page scroll
          var xScroll = el.scrollLeft || document.documentElement.scrollLeft
          var yScroll = el.scrollTop || document.documentElement.scrollTop

          xPos += (el.offsetLeft - xScroll + el.clientLeft)
          yPos += (el.offsetTop - yScroll + el.clientTop)
        } else {
          // for all other non-BODY elements
          xPos += (el.offsetLeft - el.scrollLeft + el.clientLeft)
          yPos += (el.offsetTop - el.scrollTop + el.clientTop)
        }

        el = el.offsetParent
      }
      return {
        x: xPos,
        y: yPos,
      }
    },
    arrayAverage (arr) {
    // Find the sum
      var sum = 0
      for (var i in arr) {
        sum += arr[i]
      }
      // Get the length of the array
      var numbersCnt = arr.length
      // Return the average / mean.
      return (sum / numbersCnt)
    },
    play () {
      this.audioElement.play()
      this.paused = false
      this.init()
    },
    pause () {
      this.paused = true
      this.audioElement.pause()
      this.stop()
    },
    skip () {
      this.audioElement.currentTime = this.audioElement.duration / 2
    },
    loopingFunction () {
      this.requestId = undefined
      this.analyser.getByteFrequencyData(this.audioData)
      this.draw(this.audioData)

      this.start()
    },
    start () {
      if (!this.requestId) {
        this.requestId = window.requestAnimationFrame(this.loopingFunction)
      }
    },
    stop () {
      if (this.requestId) {
        window.cancelAnimationFrame(this.requestId)
        this.requestId = undefined
        this.draw(new Array(this.audioData.length).fill(0))
      }
    },
    draw (dataRaw) {
    // call first function and pass in a callback function which
    // first function runs when it has completed
      this.drawHelper(dataRaw, () => {
        this.ready = true
      })
    },
    drawHelper (dataRaw, _callback) {
      var data = [...dataRaw].reverse()
      data = data.slice(data.length / 4, data.length - 2)
      this.bass = this.arrayAverage(data.slice(data.length - 4))
      this.c = this.$refs.myCanvas
      this.ctx = this.c.getContext('2d')
      this.ctx.clearRect(0, 0, this.c.clientWidth, this.c.clientHeight) // x,y,width,height

      this.drawTicks(data, this.ctx, this.c, this.bass)
      this.drawInnerCircle(data, this.ctx, this.c, this.bass)
      this.drawSeeker(this.ctx, this.c, this.bass)
      _callback()
    },
    drawSeeker (ctx, c, bass) {
      var coef = 0.2
      var zeroed = ((coef * bass) > 40) ? coef * bass - 40 : 0
      var radius = this.radius + zeroed - 20

      var midx = c.clientWidth / 2
      var midy = c.clientHeight / 2
      var duration = this.audioElement.duration
      var angle
      if (!this.seeking) {
        var time = this.audioElement.currentTime

        angle = ((time / duration) * 2 * Math.PI)
      } else {
        var mouseX = this.mouseLocation.x - (midx + this.getPosition(c).x)
        var mouseY = this.mouseLocation.y - (midy + this.getPosition(c).y)
        angle = Math.atan(mouseY / mouseX)

        // Get quadrant
        if ((this.mouseLocation.x > (midx + this.getPosition(c).x)) && (this.mouseLocation.y < (midy + this.getPosition(c).y))) { // Reverse order Quadrant One
          angle = angle + (2 * Math.PI)
        } else if ((this.mouseLocation.x < (midx + this.getPosition(c).x)) && (this.mouseLocation.y > (midy + this.getPosition(c).y))) {
          angle = angle + Math.PI
        } else if ((this.mouseLocation.x < (midx + this.getPosition(c).x)) && (this.mouseLocation.y < (midy + this.getPosition(c).y))) {
          angle = angle + Math.PI
        }
      }
      this.seekTo = angle * (duration / (2 * Math.PI))
      var x = midx + (radius * Math.cos(angle))
      var y = midy + (radius * Math.sin(angle))

      ctx.fillStyle = '#C4A29E'

      ctx.beginPath()
      ctx.arc(x, y, 10, 0, (2 * Math.PI))
      ctx.fill()
    },
    drawInnerCircle (data, ctx, c, bass) {
      var time = this.audioElement.currentTime
      var duration = this.audioElement.duration

      var coef = 0.2
      var zeroed = ((coef * bass) > 40) ? coef * bass - 40 : 0
      var radius = this.radius + zeroed - 20

      if (this.seeking) {
        time = this.seekTo
      }

      var midx = c.clientWidth / 2
      var midy = c.clientHeight / 2

      ctx.beginPath()
      ctx.lineWidth = '2'
      ctx.arc(midx, midy, radius, 0, (2 * Math.PI))
      ctx.stroke()

      ctx.beginPath()
      ctx.lineWidth = '6'
      ctx.arc(midx, midy, radius, 0, (2 * Math.PI) * (time / duration))
      ctx.stroke()
    },
    drawTicks (data, ctx, c, bass) {
      // Draws the ticks to the canvas
      var coef = 0.2
      var zeroed = ((coef * bass) > 40) ? coef * bass - 40 : 0
      var radius = this.radius + zeroed
      this.reactiveRadius = radius
      var midx = c.clientWidth / 2
      var midy = c.clientHeight / 2
      var step = 2 * Math.PI / data.length
      for (let i = 0; i < data.length; i++) {
        var backOne = Math.abs(i - 1)

        var calculated = (data[backOne] +
                          data[i + 0] +
                          data[(i + 1) % data.length] / 4)

        var normalizer = 0.1 + 0.9 * ((data.length - i) / data.length)

        calculated = data[i] * normalizer / 2

        ctx.beginPath()
        ctx.lineWidth = '3'
        ctx.strokeStyle = '#C4A29E'

        var angle = i * step

        var element = calculated

        var x = midx + (radius * Math.cos(angle))
        var y = midy + (radius * Math.sin(angle))

        var x2 = midx + ((radius + 16) * Math.cos(angle))
        var y2 = midy + ((radius + 16) * Math.sin(angle))

        var x4 = midx + ((radius + 16 + element + (0.1 * bass)) * Math.cos(angle))
        var y4 = midy + ((radius + 16 + element + (0.1 * bass)) * Math.sin(angle))

        ctx.moveTo(x, y)
        ctx.lineTo(x2, y2)
        ctx.lineTo(x4, y4)
        ctx.stroke() // Draw it
      }
    },
    mouseup () {
      if (this.seeking) {
        if (this.audioElement.paused) {
          this.stop()
        }
        this.audioElement.currentTime = this.seekTo
      }
      this.seeking = false
    },
    mousedown (e) {
      var midx = this.c.clientWidth / 2
      var midy = this.c.clientHeight / 2
      var distance = Math.sqrt(Math.pow(this.getMousePos(e).x - midx, 2) + Math.pow(this.getMousePos(e).y - midy, 2))
      if (distance < this.reactiveRadius && distance > (this.reactiveRadius - 40)) {
        if (this.audioElement.paused) {
          this.start()
        }
        this.seeking = true
      }
    },
    getMousePos (evt) {
      var rect = this.c.getBoundingClientRect()
      return {
        x: evt.clientX - rect.left,
        y: evt.clientY - rect.top,
      }
    },
  },
}
</script>

<style lang="scss" scoped>
.player{
  display: grid;
  place-content: center;
  height: 100vh;
  .player-inner{
    .song-info{

    }
    .audio{

    }
  }
}
.audio-wrapper {
  position: relative;
  width: 500px;
  background-color: #002A32;
  border-radius: 20px;
}
canvas {
  display: block;
}
.controls {
  position: absolute;
  left: 50%;
  top:50%;
  transform: translate(-50%, -50%);
  color: #f1f1f1;
  transition: .5s ease;
  color: white;
  text-align: center;
}
</style>
