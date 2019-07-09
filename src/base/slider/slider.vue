<template>
  <!-- 一般轮播图组件规则：
  1.有一个视口，可以容纳轮播的这些图片
  2.子容器可以左右滑动的列表
  3.有一个dot的小点点 -->
  <div class="slider" ref="slider">
    <div class="slider-group" ref="sliderGroup">
      <slot>
      </slot>
    </div>
    <div class="dots">
      <span class="dot" v-for="(item, index) in dots" :key="index" :class="{active: currentPageIndex === index}"></span>
    </div>
  </div>
</template>

<script type="text/ecmascript-6">
import BetterScroll from 'better-scroll'
import {addClass} from 'common/js/dom'

export default {
  data() {
    return {
      dots: [], // 轮播图的小点点
      currentPageIndex: 0 // 当前页码，默认是0表示第一页
    }
  },
  // 给slider组件几个初始值
  props: {
    loop: { // 是否可以轮播
      type: Boolean,
      default: true
    },
    autoPlay: { // 自动轮播
      type: Boolean,
      default: true
    },
    interval: { // 自动录播间隔
      type: Number,
      default: 2000
    }
  },
  mounted() {
    setTimeout(() => {
      this._setSliderWidth(false) // 横向滚动，要先设置slider的宽度
      this._initDots() // 就那个小点,因为是自动轮播，BScroll会自动在前后加一个sliderGroup，为了保持数量一直，要在BScroll之前初始化
      this._initSlider() // 初始化BetterScroll在mounted里，等dom已经ready的时候

      if (this.autoPlay) {
        this._play() // 自动播放
      }
    }, 20) // 浏览器的刷新通常是17毫秒一次，初始化操作放在20毫秒后，比较保险

    // 监听window的resize事件
    window.addEventListener('resize', () => {
      if (!this.slider || !this.slider.enabled) { // slider 没有初始化或者
        return
      }
      this._setSliderWidth(true)
      this.slider.refresh()
      /*      clearTimeout(this.resizeTimer)
        this.resizeTimer = setTimeout(() => {
          // 判断当前 scroll 是否处于滚动动画过程中
          if (this.slider.isInTransition) {
            this._onScrollEnd()
          } else {
            if (this.autoPlay) {
              this._play()
            }
          }
          this.refresh()
        }, 60) */
    })
  },
  // keep-alive 组件激活时，slider再次被激活，并归于初始化状态
  activated() {
    this.slider.enable()
    let pageIndex = this.slider.getCurrentPage().pageX
    this.slider.goToPage(pageIndex, 0, 0)
    this.currentPageIndex = pageIndex
    if (this.autoPlay) {
      this._play()
    }
  },
  deactivated() {
    this.slider.disable()
    clearTimeout(this.timer)
  },
  beforeDestroy() {
    this.slider.disable()
    clearTimeout(this.timer)
  },
  methods: {
    /**
       * 设置slider的宽度
       */
    _setSliderWidth(isResize) {
      // 获取sliderGroup下children的dom,注意作用域是本组件this
      this.children = this.$refs.sliderGroup.children
      console.log(this.children.length)
      let sliderWidth = 0
      // 轮播图宽度是一屏，slider的宽度=所有轮播图宽度之和
      let clientWidth = this.$refs.slider.clientWidth
      // ※※※计算整个轮播的视口应该有多宽※※※
      for (let i = 0; i < this.children.length; i++) {
        let child = this.children[i]
        // 轮播组件设置样式，让img自适应宽度
        addClass(child, 'slider-item')

        child.style.width = clientWidth + 'px'
        sliderWidth += clientWidth
      }

      // 因为bscroll为了无缝切换轮播，会前后克隆一个child，所以我们算宽度要加两个，如果是窗口改变，那么不增加，之前已经加过了
      if (this.loop && !isResize) {
        sliderWidth += 2 * clientWidth
      }
      this.$refs.sliderGroup.style.width = sliderWidth + 'px'
    },
    /**
       * 初始化better-scroll
       */
    _initSlider() {
      this.slider = new BetterScroll(this.$refs.slider, {
        scrollX: true, // 横向滚动
        scrollY: false, // 不允许纵向滚动
        momentum: false, // 当快速滑动时是否开启滑动惯性
        snap: true,
        snaploop: this.loop, // 是否可以无缝循环轮播
        snapthreshold: 0.3, // 手动切换时的阈值
        snapspeed: 400
        // ,click: true

      })

      // 每次轮播图滚动完毕的时候触发
      this.slider.on('scrollEnd', this._onScrollEnd)

      this.slider.on('touchend', () => {
        if (this.autoPlay) {
          this._play()
        }
      })

      this.slider.on('beforeScrollStart', () => {
        if (this.autoPlay) {
          clearTimeout(this.timer)
        }
      })
    },
    // 滚动完毕触发
    _onScrollEnd() {
      let pageIndex = this.slider.getCurrentPage().pageX // 表示第几个子元素
      // 滚动到第几个图片，从0开始，与index对应
      if (this.loop) {
        pageIndex -= 1 // 默认会在开始复制一个，方便无缝切换
      }
      this.currentPageIndex = pageIndex
      if (this.autoPlay) {
        clearInterval(this.timer)
        this._play()
      }
    },
    /**
       * 初始化小点点
       * @private
       */
    _initDots() {
      this.dots = new Array(this.children.length) // 一个长度为5的空数组
    },
    /**
       * 播放方法
       * @private
       */
    _play() {
      /*      clearTimeout(this.timer)
          this.timer = setTimeout(() => {
            // 滚动到下一页
            this.slider.next()
          }, this.interval) */
      let pageIndex = this.currentPageIndex + 1
      if (this.loop) {
        pageIndex += 1
      }
      this.timer = setTimeout(() => {
        // 滚动到下一页 横向轮播，400毫秒
        this.slider.goToPage(pageIndex, 0, 400)
      }, this.interval)
    },
    refresh() {
      if (this.slider) {
        this._setSliderWidth(true)
        this.slide.refresh()
      }
    }
  }
}
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"

  .slider
    min-height: 1px

    .slider-group
      position: relative
      overflow: hidden
      white-space: nowrap

      .slider-item
        float: left
        box-sizing: border-box
        overflow: hidden
        text-align: center

        a
          display: block
          width: 100%
          overflow: hidden
          text-decoration: none

        img
          display: block
          width: 100%

    .dots
      position: absolute
      right: 0
      left: 0
      bottom: 12px
      text-align: center
      font-size: 0

      .dot
        display: inline-block
        margin: 0 4px
        width: 8px
        height: 8px
        border-radius: 50%
        background: $color-text-l

        &.active
          width: 20px
          border-radius: 5px
          background: $color-text-ll
</style>
