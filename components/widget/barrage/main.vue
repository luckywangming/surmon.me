<template>
  <div class="global-barrage" :class="{ active: barrageState }">
    <div class="barrage-box">
      <div class="list-box" ref="barrageBox">
        <ul class="barrages-list">
          <barrage-item
            :id="barrage.id"
            :key="barrage.id"
            :barrage="barrage"
            :delay="config.delay"
            @end="handleAnimationEnd"
            v-for="barrage in barrages"
          />
        </ul>
      </div>
      <div
        class="input-box filter"
        :class="{ 'motion-blur-vertical': transitioning }"
        @animationstart="handleInputAnimationStart"
        @animationend="handleInputAnimationEnd"
      >
        <div class="input-inner">
          <div class="size">
            <div class="active size" :class="'s-' + sizeIndex">{{ currentSize }}</div>
            <ul class="size list">
              <li
                class="item"
                :key="index"
                :class="'s-' + index"
                v-for="(size, index) in sizes"
                @click="sizeIndex = index"
              >{{ size }}</li>
            </ul>
          </div>
          <div class="color">
            <div class="active color" :class="'color-' + colorIndex">{{ currentColor }}</div>
            <ul class="color list">
              <li
                class="item"
                :key="index"
                :class="'color-' + index"
                v-for="(color, index) in colors"
                @click="colorIndex = index"
              >{{ color }}</li>
            </ul>
          </div>
          <input
            type="text"
            class="input"
            v-model="barrage"
            placeholder="Here we go"
            @keyup.enter="sendbarrage"
          >
          <div class="count">
            <span>{{ counts.users }} {{ isEnLang ? 'U' : '人' }}</span>
            <span>&nbsp;|&nbsp;</span>
            <span>{{ counts.count }} {{ isEnLang ? 'C' : ' 发' }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import socket from '~/plugins/socket.io'
  import BarrageItem from './item'
  export default {
    name: 'barrage',
    components: {
      BarrageItem
    },
    data() {
      const sizes = this.isEnLang ? ['Strong', 'Large', 'Normal'] : ['粗大', '很大', '大'];
      const colors = this.isEnLang
        ? ['Green', 'Green2', 'Red', 'Purple', 'Pink', 'Yellow', 'White', 'Black']
        : ['老王绿', '原谅绿', '姨妈红', '友情紫', '百合粉', '东莞黄', '李太白', '非常黑'];
      return {
        sizes,
        colors,
        socket,
        counts: {
          users: 0,
          count: 0
        },
        config: {
          delay: 10,
          moveDelay: 3
        },
        barrage: '',
        barrages: [],
        moveTimer: null,
        barrageLimit: 0,
        sizeIndex: sizes.length - 1,
        colorIndex: colors.length - 1,
        transitioning: false
      }
    },
    computed: {
      isEnLang() {
        return this.$store.getters['global/isEnLang']
      },
      currentColor() {
        return this.colors[this.colorIndex]
      },
      currentSize() {
        return this.sizes[this.sizeIndex]
      },
      barrageState() {
        return this.$store.state.global.onBarrage
      }
    },
    beforeMount() {
      this.socket.emit('barrage-last-list', barrages => {
        barrages.forEach((b, i) => {
          b.id = i + 1
        })
        // 生成随机的时间，push 进不同的内容，而不是一次性赋值
        const moveBarrages = () => {
          if (barrages.length) {
            // console.log('moveBarrages， 还有', barrages.length)
            this.barrages.push(barrages[0])
            barrages.splice(0, 1)
            if (barrages.length) {
              this.moveTimer = setTimeout(moveBarrages, parseInt(this.randomPer(this.config.moveDelay), 0) * 100)
            }
          }
        }
        moveBarrages()
        this.barrageLimit = barrages.length + 2
      })
      this.socket.emit('barrage-count', counts => {
        this.counts = counts
      })
      this.socket.on('barrage-update-count', counts => {
        this.counts = counts
      })
      this.socket.on('barrage-create', barrage => {
        this.barrages.push(barrage)
      })
    },
    beforeDestroy() {
      if (this.moveTimer) {
        clearTimeout(this.moveTimer)
      }
      this.clearBarrages()
    },
    methods: {
      // 弹幕输入容器动画周期
      handleInputAnimationStart() {
        this.transitioning = true
      },
      handleInputAnimationEnd() {
        this.transitioning = false
      },
      // 发布弹幕
      sendbarrage() {
        const text = this.barrage.trim()
        if (!text) return
        const barrage = {
          text,
          style: {
            size: this.sizeIndex,
            color: this.colorIndex
          },
          date: new Date().getTime()
        }
        this.socket.emit('barrage-send', barrage)
        barrage.id = this.barrageLimit++
        this.barrages.push(barrage)
        this.counts.count += 1
        this.barrage = ''
      },
      // 时间转换
      transferDate(timestamp) {
        return new Date(timestamp).toLocaleString()
      },
      // 计算随机数
      randomPer(pre = 3) {
        const rnd = seed => {
          seed = (seed * 9301 + 49297) % 233280
          return seed / (233280.0)
        }
        const rand = number => {
          const seed = new Date().getTime()
          return rnd(seed) * number + Math.random()
        }
        return rand(pre)
      },
      // 清空动画队列
      clearBarrages() {
        this.barrages = []
      },
      handleAnimationEnd(id) {
        const targetIndex = this.barrages.findIndex(barrage => barrage.id === id)
        if (targetIndex > -1) {
          this.barrages.splice(targetIndex, 1)
        }
      }
    }
  }
</script>

<style lang="scss">
  // 字体尺寸
  .size-0 {
    font-size: 3em;
    font-weight: bolder;
  }

  .size-1 {
    font-size: 3em;
  }

  .size-2 {
    font-size: 2em;
  }

  // 字体颜色
  .color-0 {
    color: chartreuse;
  }

  .color-1 {
    color: green;
  }

  .color-2 {
    color: red;
  }

  .color-3 {
    color: darkviolet;
  }

  .color-4 {
    color: pink;
  }

  .color-5 {
    color: yellow;
  }

  .color-6 {
    color: white;
  }

  .color-7 {
    color: black;
  }
</style>

<style lang="scss" scoped>

  @keyframes barrage-in {
    0%  { transform: translate3d(0, -100%, 0) }
    25%, 50%, 75%, 100% { transform: translate3d(0, 0, 0) }
    37%  { transform: translate3d(0, -24%, 0) }
    62% { transform: translate3d(0, -12%, 0) }
    88% { transform: translate3d(0, -5%, 0) }
  }

  @keyframes barrage-out {
    0%  { transform: translate3d(0, 0, 0) }
    100%  { transform: translate3d(0, -100%, 0) }
  }

  @keyframes input-box-in {
    0%  { transform: translate3d(0, -2000%, 0) }
    65%  { transform: translate3d(0, 100%, 0) }
    80%  { transform: translate3d(0, -80%, 0) }
    100%  { transform: translate3d(0, 0, 0) }
  }

  .global-barrage {
    position: fixed;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    z-index: 8;
    transform: translate3d(0, -100%, 0);
    animation-duration: .5s; 
    animation-fill-mode: both; 
    animation-name: barrage-out;
    background-color: $module-hover-bg-darken-20;
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
    perspective: 1000;
    -webkit-perspective: 1000;
    @include hidden();
    @include backdrop-blur();

    &.active {
      @include visible();
      animation-duration: 1.556s;
      animation-fill-mode: both; 
      animation-name: barrage-in;
      animation-timing-function: ease;

      > .barrage-box {

        > .input-box {
          animation-delay: 1.6s;
          animation-duration: .5s; 
          animation-fill-mode: both; 
          animation-name: input-box-in;
          animation-timing-function: ease;
        }
      }
    }

    > .barrage-box {
      width: 100%;
      height: 100%;
      display: block;
      position: relative;
      padding-top: $header-height;

      > .list-box {
        height: 100%;
        width: 100%;
        display: block;
        position: relative;

        > .barrages-list {
          list-style: square;
          display: block;
          width: 100%;
          height: 100%;
          padding: 0;
          margin: 0;
          user-select: none;

          @keyframes barrages-list-out {
            0%  { transform: translate3d(100%, 0, 0) }
            90% { transform: translate3d(-100%, 0, 0) }
          }

          .barrages-list-leave-active {
            animation-duration: 30s; 
            animation-fill-mode: both; 
            animation-name: barrages-list-out;
          }
        }
      }

      > .input-box {
        display: block;
        position: absolute;
        bottom: 40%;
        width: 100%;
        z-index: 999;
        transition: transform 5s;
        transform: translate3d(0, -2000%, 0);

        > .input-inner {
          display: flex;
          margin: 0 auto;
          width: 42rem;
          height: 4rem;
          background-color: $module-bg-opacity-8;

          > .count {
            width: auto;
            height: 4rem;
            line-height: 4rem;
            padding: 0 $gap;
            text-align: center;
          }

          > .size,
          > .color {
            width: 7rem;
            position: relative;

            &:hover {

              > .list {
                @include visible();
              }
            }

            > .active {
              cursor: pointer;
              display: block;
              width: 100%;
              height: 4rem;
              line-height: 4rem;
              text-align: center;
            }

            > .list {
              margin: 0;
              padding: 0;
              list-style: none;
              position: absolute;
              bottom: 4rem;
              left: 0;
              width: 100%;
              height: auto;
              background-color: $module-bg-opacity-8;
              @include hidden();

              > .item {
                height: 3rem;
                line-height: 3rem;
                text-align: center;
                cursor: pointer;

                &:hover {
                  background-color: $module-bg-opacity-8;
                }
              }
            }
          }

          > .input {
            width: auto;
            margin: 0 auto;
            flex-grow: 1;
            padding: 1rem;
            background-color: $module-bg-opacity-5;

            &:hover,
            &:focus {
              background-color: $module-bg-opacity-9;
            }
          }
        }
      }
    }
  }
</style>
