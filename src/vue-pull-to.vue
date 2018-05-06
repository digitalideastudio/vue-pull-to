<template>
  <div class="vue-pull-to-wrapper"
       :style="{ height: wrapperHeight, transform: `translate3d(0, ${diff}px, 0)` }">
    <div style="position:absolute; top: 0; border: 1px solid red;z-index: 999999;background: #fff;">
      <p>Direction: {{ direction }}</p>
      <p>Distance: {{ distance }}</p>
      <p>State: {{ state }}</p>
      <p>StartScrollTop: {{ startScrollTop }}</p>
    </div>
    <div v-if="topLoadMethod"
         :style="{ height: `${topBlockHeight}px`, marginTop: `${-topBlockHeight}px` }"
         class="action-block">
      <slot name="top-block"
            :state="state"
            :state-text="topText">
        <p class="default-text">{{ topText }}</p>
      </slot>
    </div>
    <div class="scroll-container"
         ref="scrollEl"
         @scroll="handleScroll"
         @touchstart="handleTouchStart"
         @touchmove="handleTouchMove"
         @touchend="handleTouchEnd">
      <slot></slot>
    </div>
  </div>
</template>

<script type="text/babel">
  import { throttle } from './utils';
  import { TOP_DEFAULT_CONFIG, BOTTOM_DEFAULT_CONFIG } from './config';

  export default {
    name: 'vue-pull-to',
    props: {
      distanceIndex: {
        type: Number,
        default: 2
      },
      topBlockHeight: {
        type: Number,
        default: 50
      },
      wrapperHeight: {
        type: String,
        default: '100%'
      },
      topLoadMethod: {
        type: Function
      },
      bottomLoadMethod: {
        type: Function
      },
      isThrottleTopPull: {
        type: Boolean,
        default: true
      },
      isThrottleBottomPull: {
        type: Boolean,
        default: true
      },
      isThrottleScroll: {
        type: Boolean,
        default: true
      },
      isTopBounce: {
        type: Boolean,
        default: true
      },
      isBottomBounce: {
        type: Boolean,
        default: true
      },
      topConfig: {
        type: Object,
        default: () => {
          return {};
        }
      },
      bottomConfig: {
        type: Object,
        default: () => {
          return {};
        }
      }
    },
    data() {
      return {
        scrollEl: null,
        startScrollTop: 0,
        startY: 0,
        currentY: 0,
        distance: 0,
        direction: 0,
        diff: 0,
        beforeDiff: 0,
        topText: '',
        bottomText: '',
        state: 'idle',
        bottomReached: false,
        topReached: true,
        throttleEmitTopPull: null,
        throttleEmitBottomPull: null,
        throttleEmitScroll: null,
      };
    },
    computed: {
      _topConfig: function () {
        return Object.assign({}, TOP_DEFAULT_CONFIG, this.topConfig);
      },
      _bottomConfig: function () {
        return Object.assign({}, BOTTOM_DEFAULT_CONFIG, this.bottomConfig);
      }
    },
    watch: {
      state(val) {
        if (this.direction === 'down') {
          this.$emit('top-state-change', val);
        } else {
          this.$emit('bottom-state-change', val);
        }
      }
    },
    methods: {
      actionPull() {
        this.state = 'pull';
        this.direction === 'down'
          ? this.topText = this._topConfig.pullText
          : this.bottomText = this._bottomConfig.pullText;
      },
      actionTrigger() {
        this.state = 'trigger';
        this.direction === 'down'
          ? this.topText = this._topConfig.triggerText
          : this.bottomText = this._bottomConfig.triggerText;
      },
      actionLoading() {
        this.state = 'loading';
        if (this.direction === 'down') {
          this.topText = this._topConfig.loadingText;
          /* eslint-disable no-useless-call */
          this.topLoadMethod.call(this, this.actionLoaded);
          this.scrollTo(this._topConfig.stayDistance);
        } else {
          this.bottomText = this._bottomConfig.loadingText;
          this.bottomLoadMethod.call(this, this.actionLoaded);
          this.scrollTo(-this._bottomConfig.stayDistance);
        }
      },
      actionLoaded(loadState = 'done') {
        this.state = `loaded-${loadState}`;
        let loadedStayTime;
        if (this.direction === 'down') {
          this.topText = loadState === 'done'
            ? this._topConfig.doneText
            : this._topConfig.failText;
          loadedStayTime = this._topConfig.loadedStayTime;
        } else {
          this.bottomText = loadState === 'done'
            ? this._bottomConfig.doneText
            : this._bottomConfig.failText;
          loadedStayTime = this._bottomConfig.loadedStayTime;
        }
        setTimeout(() => {
          this.scrollTo(0);

          // reset state
          setTimeout(() => {
            this.state = 'idle';
          }, 200);
        }, loadedStayTime);
      },
      scrollTo(y, duration = 200) {
        this.$el.style.transition = `${duration}ms`;
        this.diff = y;
        setTimeout(() => {
          this.$el.style.transition = '';
        }, duration);
      },

      handleTouchStart(event) {
        this.startY = event.touches ? event.touches[0].clientY : event.clientY;
        this.beforeDiff = this.diff;
      },

      handleTouchMove(event) {
        this.currentY = event.touches ? event.touches[0].clientY : event.clientY;
        this.distance = (this.currentY - this.startY) / this.distanceIndex + this.beforeDiff;
        this.direction = this.distance > 0 ? 'down' : 'up';

        if (this.startScrollTop <= 0 && this.direction === 'down' && this.isTopBounce) {
          event.preventDefault();
          event.stopPropagation();
          this.diff = this.distance;
          // this.isThrottleTopPull ? this.throttleEmitTopPull(this.diff) : this.$emit('top-pull', this.diff);

          if (typeof this.topLoadMethod !== 'function') return;

          if (this.distance < this._topConfig.triggerDistance &&
            this.state !== 'pull' && this.state !== 'loading') {
            this.actionPull();
          } else if (this.distance >= this._topConfig.triggerDistance &&
            this.state === 'pull') {
            this.actionTrigger();
          }
        }
      },

      handleTouchEnd(event) {
        event.stopPropagation();
        if (this.diff !== 0) {
          if (this.state === 'trigger') {
            this.actionLoading();
            return;
          }

          // pull cancel
          this.scrollTo(0);
        }
      },

      handleScroll(event) {
        this.startScrollTop = this.scrollEl.scrollTop;
      },

      throttleEmit(delay, mustRunDelay = 0, eventName) {
        const throttleMethod = function () {
          const args = [...arguments];
          args.unshift(eventName);
          this.$emit.apply(this, args);
        };

        return throttle(throttleMethod, delay, mustRunDelay);
      },

      createThrottleMethods() {
        this.throttleEmitTopPull = this.throttleEmit(200, 300, 'top-pull');
        this.throttleEmitBottomPull = this.throttleEmit(200, 300, 'bottom-pull');
        this.throttleEmitScroll = this.throttleEmit(100, 150, 'scroll');
      },

      init() {
        // this.createThrottleMethods();
        this.scrollEl = this.$el.querySelector('.scroll-container');
      }
    },
    mounted() {
      this.init();
    }
  };
</script>

<style scoped>
  .vue-pull-to-wrapper {
    display: flex;
    flex-direction: column;
    height: 100%;
  }

  .scroll-container {
    flex: 1;
    overflow-y: auto;
    -webkit-overflow-scrolling: touch;
  }

  .vue-pull-to-wrapper .action-block {
    position: relative;
    width: 100%;
  }

  .default-text {
    height: 100%;
    line-height: 50px;
    text-align: center;
  }
</style>
