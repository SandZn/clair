<template lang="pug">
.c-tip(
  @mouseenter="show",
  @mouseleave="hide",
  @focus.capture="show",
  @blur.capture="hide",
  @click="show"
)
  slot
  c-portal(role="tooltip", :aria-hidden="'' + !visible")
    transition(
      v-if="!disabled",
      @before-enter="beforeEnter",
      @enter="enter",
      @after-enter="afterEnter",
      @leave="leave",
      @after-leave="afterLeave"
    )
      .c-tip__container(
        :class="theme === 'light' && 'c-tip__container--light'"
        ref="tip",
        v-show="visible",
        @mouseenter="show",
        @mouseleave="hide"
      )
        i.c-tip__arrow(:class="arrowClass")
        div(v-if="content") {{ content }}
        slot(name="content")
</template>

<script>
import throttle from 'lodash/throttle'
import { contains, VueTypes } from '@util'
import zIndex from '@util/zIndexManager'
import PortalComponent from '../portal'

import './index.css'

const OPPOSITE_DIRECTION = {
  top: 'bottom',
  bottom: 'top',
  left: 'right',
  right: 'left'
}

const SHOW_MATCH_MAP = {
  hover: 'mouseenter',
  focus: 'focus',
  click: 'click'
}

const HIDE_MATCH_MAP = {
  hover: 'mouseleave',
  focus: 'blur',
  click: 'click'
}

const defaultDelayTime = 100

export default {
  name: 'c-tip',

  components: {
    'c-portal': PortalComponent
  },

  props: {
    theme: VueTypes.oneOf(['dark', 'light']).def('dark'),
    trigger: {
      type: [String, Array],
      default: 'hover',
      validator (val) {
        const validTypes = ['hover', 'click', 'focus']
        return [].concat(val).every(v => validTypes.includes(v))
      }
    },
    disabled: VueTypes.bool.def(false),
    content: VueTypes.string.def(''),
    maxWidth: VueTypes.string.def('300px'),
    showDelay: VueTypes.number.def(defaultDelayTime),
    hideDelay: VueTypes.number.def(defaultDelayTime),
    position: VueTypes.oneOf(['top', 'right', 'bottom', 'left']).def('bottom')
  },

  data () {
    return {
      visible: false,
      tidIn: null,
      tidOut: null
    }
  },

  computed: {
    arrowClass () {
      const position = OPPOSITE_DIRECTION[this.position]
      return `c-tip__arrow--${position}`
    },
    triggers () {
      const { trigger } = this
      const triggers = [].concat(trigger)
      return triggers
    }
  },

  methods: {
    show ({ type = 'click' }) {
      const { triggers } = this
      for (let i = 0; i < triggers.length; i++) {
        if (SHOW_MATCH_MAP[triggers[i]] === type) {
          this.clearTimeout()
          this.visible = true
          break
        }
      }
    },

    hide ({ type = 'click' } = {}) {
      const { triggers } = this
      for (let i = 0; i < triggers.length; i++) {
        if (HIDE_MATCH_MAP[triggers[i]] === type) {
          this.clearTimeout()
          this.tidOut = setTimeout(() => {
            this.visible = false
          }, this.hideDelay)
          break
        }
      }
    },

    resize () {
      this.handleResize(this.$refs.tip)
    },

    beforeEnter ({ style }) {
      style.display = 'block'
      style.visibility = 'hidden'
      style.zIndex = zIndex.next()
    },

    enter ({ style }, done) {
      style.opacity = 0

      this.tidIn = setTimeout(() => {
        style.maxWidth = this.maxWidth
        style.visibility = 'visible'
        style.opacity = 1
        this.$nextTick(done)
      }, this.showDelay)
    },

    afterEnter (el) {
      this.handleResize(el)
    },

    leave ({ style }) {
      style.opacity = 0
      style.visibility = 'hidden'
      this.clearTimeout()
    },

    afterLeave ({ style }) {
      style.cssText = ''
      style.display = 'none'
    },

    clearTimeout () {
      clearTimeout(this.tidOut)
      clearTimeout(this.tidIn)
    },

    handleResize (el) {
      if (!el || !el.style || !this.visible) {
        return
      }

      // SEE https://imququ.com/post/document-scrollingelement-in-chrome.html
      const { scrollLeft, scrollTop } = document.scrollingElement || document.body

      const elRect = this.$el.getBoundingClientRect()
      const tipRect = this.$refs.tip.getBoundingClientRect()
      const { style } = el

      // eslint-disable-next-line
      switch (this.position) {
        case 'top':
          style.top = `${scrollTop + elRect.top - tipRect.height}px`
          style.left = `${scrollLeft + elRect.left + (elRect.width - tipRect.width) / 2}px`
          style.marginTop = '-10px'
          style.marginLeft = ''
          return

        case 'bottom':
          style.top = `${scrollTop + elRect.top + elRect.height}px`
          style.left = `${scrollLeft + elRect.left + (elRect.width - tipRect.width) / 2}px`
          style.marginTop = '10px'
          style.marginLeft = ''
          return

        case 'left':
          style.top = `${scrollTop + elRect.top - (tipRect.height - elRect.height) / 2}px`
          style.left = `${scrollLeft + elRect.left - tipRect.width}px`
          style.marginLeft = '-10px'
          style.marginTop = ''
          return

        case 'right':
          style.top = `${scrollTop + elRect.top - (tipRect.height - elRect.height) / 2}px`
          style.left = `${scrollLeft + elRect.left + elRect.width}px`
          style.marginLeft = '10px'
          style.marginTop = ''
      }
    },

    clickOutside ({ target }) {
      if (!this.visible) {
        return
      }

      const el = this.$el
      const { tip } = this.$refs
      const isOutside = !contains(el, target) && !contains(tip, target)
      if (isOutside && this.visible) {
        this.hide()
      }
    }
  },

  updated () {
    if (this.visible) {
      this.$nextTick(this.resize)
    }
  },

  mounted () {
    this.resize = this.resize.bind(this)
    this.clickOutside = this.clickOutside.bind(this)
    this.winResize = throttle(this.resize, this.$clair.defaultThrottleTime)
    window.addEventListener('resize', this.winResize)
    document.body.addEventListener('click', this.clickOutside)
  },

  beforeDestroy () {
    this.clearTimeout()
    window.removeEventListener('resize', this.winResize)
    document.body.removeEventListener('click', this.clickOutside)
  }
}
</script>
