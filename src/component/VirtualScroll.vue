<template>
  <div id="wrapper">
    <div class="virtual-scroller">
      <!-- the pulldown refresh area -->
      <div class="refresh-loader" v-if="pulldown" v-show="pullState.length > 0" :style="getPullerStyle">
        <slot name="pullRefresh">
          <i class="iconfont" :class="getPullerClass"></i>
          <span class="text-default">{{pullText}}</span> 
        </slot>
      </div>
      <!-- the virtual scroll list area -->
      <ul class="scroll-list">
        <li class="list-item" v-for="item in pool" :key="item[uniqueKey]" :style="getItemStyle(item)">
          <slot name="content" :item="item" />
        </li>
      </ul>
      <!-- the bottom infinite loader area -->
      <div class="infinite-loader" v-if="infinite" v-show="infiniteLoading" :style="getSpinnerStyle">
        <slot name="infiniteLoader">
          <i class="spinner-default"></i>
          <span class="text-default">{{infiniteText}}</span>
        </slot>
      </div>
    </div>
  </div>
</template>

<script>
import IScroll from '../iscroll-probe.js'
import '../assets/icon/iconfont.css'

export default {
  name: 'virtual-scroll',
  props: {
    items: {
      type: Array,
      required: true
    },
    uniqueKey: {
      type: String,
      required: true,
      default: 'id'
    },
    infinite: Boolean,
    pulldown: Boolean,
    pulldownText: {
      type: Object,
      default: () => {
        return {
          begin: '下拉刷新',
          trigger: '释放更新',
          refresh: '更新中...',
          complete: '更新完成',
          error: '更新失败'
        }
      }
    },
    infiniteText: {
      type: String,
      default: '加载中...'
    },
    variable: {
      type: Boolean,
      default: false
    },
    distance: {
      type: Number,
      default: 50
    },
    bufferSize: [String, Number],
    iscrollOptions: {
      type: Object,
      default: () => {
        return {
          scrollbars: true,
          interactiveScrollbars: true,
          probeType: 3,
          mouseWheel: true,
          mouseWheelSpeed: 1
        }
      }
    },
    loadMore: Function,
    pullRefresh: Function
  },
  data () {
    return {
      myScroll: null,
      viewInited: false,
      wrapperHeight: 0,
      itemHeight: 0,
      buffer: 0,
      poolLength: 0,
      pool: [],
      infiniteLoading: false,
      infiniteComplete: false,
      pullerTop: 0,
      pullState: '',
      accumulator: 0,
      oriItemLength: 0,
      scrolledItem: null
    }
  },
  created () {
    this.buffer = this.bufferSize || 5
    this.pool = this.items.slice(0, this.buffer)
  },
  mounted () {
    this.$nextTick(() => {
      this.initScroller()
    })
  },
  watch: {
    items (newArr, oldArr) {
      if (!this.viewInited) {
        this.pool = this.items.slice(0, this.buffer)
        this.$nextTick(() => {
          this.generateItemAccumulator(true)
          this.initScrollView()
        })
      }
      if (this.pool.length < this.poolLength) {
        let index = this.items.indexOf(this.pool[0])
        this.pool = this.items.slice(index, this.poolLength)
      }
    }
  },
  computed: {
    getPullerStyle () {
      let top = -this.pullerTop
      return {
        transform: 'translate(0, ' + top + 'px)'
      }
    },
    getPullerClass () {
      return {
        'icon-pulldown': this.pullState === 'begin' || this.pullState === 'trigger',
        'icon-pullup': this.pullState === 'trigger',
        'spinner-default': this.pullState === 'refresh',
        'icon-complete': this.pullState === 'complete',
        'icon-error': this.pullState === 'error'
      }
    },
    getSpinnerStyle () {
      let top = this.variable ? this.accumulator : this.itemHeight * this.items.length
      return {
        transform: 'translate(0, ' + top + 'px)'
      }
    },
    pullText () {
      return this.pulldownText[this.pullState] || ''
    }
  },
  methods: {
    generateItemAccumulator (init) {
      if (!this.variable) return
      if (init) {
        this.accumulator = 0
        this.items.forEach((item) => {
          this.$set(item, '_top', this.accumulator)
          this.accumulator += item.height
        })
      } else {
        this.items.slice(this.oriItemLength).forEach((item) => {
          this.$set(item, '_top', this.accumulator)
          this.accumulator += item.height
        })
      }
    },
    initScroller () {
      this.generateItemAccumulator(true)
      this.myScroll = new IScroll('#wrapper', this.iscrollOptions)
      this.initScrollView()
      this.initEvents()
    },
    initScrollView () {
      this.wrapperHeight = this.$el.clientHeight
      if (this.items.length === 0) return
      if (!this.variable) {
        this.itemHeight = this.$el.querySelector('.list-item').offsetHeight
        this.poolLength = Math.ceil(this.wrapperHeight / this.itemHeight) + 2 * this.buffer
        this.pool = this.items.slice(0, this.poolLength)
        this.updateScrollView()
        this.resetScroller()
      } else {
        let initSize = this.getScrolledIndex(this.wrapperHeight)
        this.poolLength = initSize + 2 * this.buffer
        this.pool = this.items.slice(0, this.poolLength)
        this.resetScroller()
      }
      this.viewInited = true
    },
    initEvents () {
      if (this.myScroll) {
        this.myScroll.on('scroll', this.handleScrollEvent)
        this.myScroll.on('scrollEnd', this.handleScrollEndEvent)
        this.myScroll.on('pullDownEnd', this.handlePullDownEndEvent)
      }
    },
    destroy () {
      if (this.myScroll) {
        this.myScroll.destroy()
        this.myScroll = null
      }
    },
    refresh () {
      if (this.myScroll) {
        this.destroy()
        this.$nextTick(() => {
          this.initScroller()
        })
      }
    },
    resetParams () {
      this.infiniteLoading = false
      this.infiniteComplete = false
    },
    getScrolledIndex (target) {
      let start = 0
      let end = this.items.length - 1
      while (start <= end) {
        let mid = parseInt(start + (end - start) / 2)
        let item = this.items[mid]
        if (target >= item._top && target < item._top + item.height) {
          return mid
        } else if (target < item._top) {
          end = mid - 1
        } else {
          start = mid + 1
        }
      }
      return -1
    },
    handleScrollEvent () {
      if (!this.infiniteComplete && !this.infiniteLoading && this.myScroll.directionY > 0 && this.myScroll.maxScrollY > this.myScroll.y - this.distance) {
        this.triggerLoadmore()
      } else if (this.pulldown && this.myScroll.y > 0) {
        this.triggerPulldownRefresh()
      } else {
        this.updateScrollView()
      }
    },
    handleScrollEndEvent () {
      if (this.pullState === 'complete' || this.pullState === 'error' || this.pullState === 'begin') {
        this.pullState = ''
        this.myScroll.pullState = ''
      }
    },
    handlePullDownEndEvent () {
      if (this.pullState === 'trigger') {
        this.pullState = 'refresh'
        this.myScroll.pullState = 'refresh'
        this.$emit('pullRefresh', this.pullStateManager)
      }
    },
    triggerLoadmore () {
      if (!this.infinite) return
      this.isPulling = false
      this.infiniteLoading = true
      this.oriItemLength = this.items.length
      this.$nextTick(() => {
        this.resetScroller(this.$el.querySelector('.infinite-loader').offsetHeight)
      })
      this.$emit('loadMore', this.infiniteStateManager)
    },
    triggerPulldownRefresh () {
      if (!this.pulldown) return
      if (this.infiniteLoading || this.pullState === 'refresh' || this.pullState === 'complete' || this.pullState === 'error') return
      if (!this.pullState || this.myScroll.y <= this.pullerTop) {
        this.pullState = 'begin'
        this.myScroll.pullerHeight = 0
        if (!this.pullerTop) {
          this.$nextTick(() => {
            this.pullerTop = this.$el.querySelector('.refresh-loader').offsetHeight
          })
        }
      } else if (this.myScroll.y > this.pullerTop) {
        this.pullState = 'trigger'
        this.myScroll.pullerHeight = this.pullerTop
      }
    },
    infiniteStateManager (state) {
      switch (state) {
        case 'loaded':
          this.infiniteLoading = false
          this.generateItemAccumulator(false)
          this.$nextTick(() => {
            this.resetScroller()
            this.updateScrollView()
          })
          break
        case 'error':
          this.infiniteLoading = false
          this.$nextTick(() => {
            this.resetScroller()
          })
          break
        case 'complete':
          this.infiniteLoading = false
          this.infiniteComplete = true
          this.generateItemAccumulator(false)
          this.$nextTick(() => {
            this.resetScroller()
          })
          break
      }
    },
    pullStateManager (state) {
      if (state === 'complete') {
        this.myScroll.pullerHeight = 0
        this.pullState = 'complete'
        this.generateItemAccumulator(true)
        setTimeout(() => {
          this.resetScroller(null, 600)
          this.resetParams()
        }, 500)
      } else {
        this.myScroll.pullerHeight = 0
        this.pullState = 'error'
        setTimeout(() => {
          this.resetScroller(null, 600)
          this.resetParams()
        }, 500)
      }
    },
    resetScroller (loaderHeight, time) {
      let h = loaderHeight || 0
      if (!this.variable) {
        this.myScroll.scrollerHeight = this.itemHeight * this.items.length + h
        this.myScroll.maxScrollY = -this.itemHeight * this.items.length + this.wrapperHeight - h
      } else {
        this.myScroll.scrollerHeight = this.accumulator + h
        this.myScroll.maxScrollY = this.wrapperHeight - this.accumulator - h
      }
      this.myScroll.refresh(time)
    },
    updateScrollView () {
      if (!this.variable) {
        let scrolledLength = Math.max(Math.floor(-this.myScroll.y / this.itemHeight) - this.buffer, 0)
        let majorPhase = Math.floor(scrolledLength / this.pool.length)
        let majorLength = scrolledLength % this.pool.length
        let i = 0
        let top = 0
        while (i < this.pool.length) {
          top = majorPhase * this.pool.length * this.itemHeight + i * this.itemHeight
          if (i < majorLength) {
            top += this.itemHeight * this.pool.length
          }
          if (i < this.pool.length && this.pool[i]._top !== top) {
            this.updateItem(i, top)
          }
          i++
        }
      } else {
        let scrolledIndex = this.getScrolledIndex(-this.myScroll.y)
        let scrolledLength = Math.max(scrolledIndex - this.buffer, 0)
        let majorPhase = Math.floor(scrolledLength / this.pool.length)
        let majorLength = scrolledLength % this.pool.length
        let i = 0
        let newIndex = 0
        while (i < this.pool.length) {
          newIndex = majorPhase * this.pool.length + i
          if (i < majorLength) {
            newIndex += this.pool.length
          }
          if (newIndex < this.items.length && this.pool[i] !== this.items[newIndex]) {
            this.updateItem(i, newIndex)
          }
          i++
        }
      }
    },
    updateItem (i, top) {
      if (!this.variable) {
        let index = top / this.itemHeight
        if (index < this.items.length) {
          let item = this.items[index]
          item._top = top
          this.$set(this.pool, i, item)
        }
      } else {
        let item = this.items[top]
        this.$set(this.pool, i, item)
      }
    },
    getItemStyle (item) {
      return {
        transform: 'translate(0, ' + item._top + 'px)'
      }
    }
  }
}
</script>
<style lang="postcss">
  #wrapper {
    position: absolute;
    top: 0px;
    bottom: 0px;
    left: 0px;
    width: 100%;
    z-index: 1;
    overflow: hidden;
    & .refresh-loader {
      text-align: center;
      padding: 5px 0;
      & i {
        font-size: 20px;
        color: #999;
        vertical-align: middle;
      }
      & .icon-pulldown {
        transition: transform .3s;
        transform: rotate(180deg);
        display: inline-block;
      }
      & .icon-pullup {
        transform: rotate(0deg);
        transition: transform .3s;
      }
      & .icon-complete, & .icon-error {
        font-size: 23px;
      }
      & .text-default {
        vertical-align: middle;
        color: #999;
        font-size: 14px;
      }
    }
    & .scroll-list {
      list-style: none;
      margin: 0;
      padding: 0;
      & .list-item {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
      }
    }
    & .infinite-loader {
      position: absolute;
      width: 100%;
      z-index: 10000;
      text-align: center;
      padding: 5px 0;
      & .text-default {
        vertical-align: middle;
        font-size: 14px;
        color: #999;
      }
    }
    & .spinner-default {
      display: inline-block;
      width: 18px;
      height: 18px;
      line-height: 28px;
      border-radius: 50%;
      vertical-align: middle;
      position: relative;
      border: 1px solid #999;
      animation: loading-rotating ease 1.5s infinite;
      &:before {
        content: '';
        width: 6px;
        height: 6px;
        position: absolute;
        display: block;
        top: 0;
        left: 50%;
        margin-top: -3px;
        margin-left: -3px;
        background-color: #999;
        border-radius: 50%;
      }
    }
  }
  @keyframes loading-rotating {
    0%{
      transform: rotate(0);
    }
    100%{
      transform: rotate(360deg);
    }
  }
</style>
