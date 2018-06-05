<template>
 <div>{{this}}</div>
</template>

<style>
</style>

<script>
export default {
  name: 'QResizeObservable',
  props: {
    debounce: {
      type: Number,
      default: 100
    }
  },
  data () {
    return this.hasObserver
      ? {}
      : { url: this.$q.platform.is.ie ? null : 'about:blank' }
  },
  methods: {
    onResize () {
      this.timer = null

      if (!this.$el || !this.$el.parentNode) {
        return
      }

      const
        parent = this.$el.parentNode,
        size = {
          width: parent.offsetWidth,
          height: parent.offsetHeight
        }

      if (size.width === this.size.width && size.height === this.size.height) {
        return
      }

      this.size = size
      this.$emit('resize', this.size)
    },
    trigger (immediately) {
      if (immediately || this.debounce === 0) {
        this.onResize()
      }
      else if (!this.timer) {
        this.timer = setTimeout(this.onResize, this.debounce)
      }
    }
  },
  render (h) {
    if (this.hasObserver) {
      return
    }

    return h('object', {
      style: this.style,
      attrs: {
        type: 'text/html',
        data: this.url,
        'aria-hidden': true
      },
      on: {
        load: () => {
          this.$el.contentDocument.defaultView.addEventListener('resize', this.trigger, listenOpts.passive)
          this.trigger(true)
        }
      }
    })
  },
  beforeCreate () {
    this.size = { width: -1, height: -1 }
    this.hasObserver = typeof ResizeObserver !== 'undefined'

    if (!this.hasObserver) {
      this.style = `${this.$q.platform.is.ie ? 'visibility:hidden;' : ''}display:block;position:absolute;top:0;left:0;right:0;bottom:0;height:100%;width:100%;overflow:hidden;pointer-events:none;z-index:-1;`
    }
  },
  mounted () {
    if (this.hasObserver) {
      this.observer = new ResizeObserver(this.trigger)
      this.observer.observe(this.$el.parentNode)
      return
    }

    this.trigger(true)

    if (this.$q.platform.is.ie) {
      this.url = 'about:blank'
    }
  },
  beforeDestroy () {
    clearTimeout(this.timer)

    if (this.hasObserver) {
      this.observer.unobserve(this.$el.parentNode)
      return
    }

    if (this.$el.contentDocument) {
      this.$el.contentDocument.defaultView.removeEventListener('resize', this.trigger, listenOpts.passive)
    }
  }
}
  export const listenOpts = {}
Object.defineProperty(listenOpts, 'passive', {
  configurable: true,
  get () {
    let passive

    try {
      var opts = Object.defineProperty({}, 'passive', {
        get () {
          passive = { passive: true }
        }
      })
      window.addEventListener('qtest', null, opts)
      window.removeEventListener('qtest', null, opts)
    }
    catch (e) {}

    listenOpts.passive = passive
    return passive
  },
  set (val) {
    Object.defineProperty(this, 'passive', {
      value: val
    })
  }
})

export function leftClick (e) {
  return e.button === 0
}

export function middleClick (e) {
  return e.button === 1
}

export function rightClick (e) {
  return e.button === 2
}

export function getEventKey (e) {
  return e.which || e.keyCode
}

export function position (e) {
  let posx, posy

  if (e.touches && e.touches[0]) {
    e = e.touches[0]
  }
  else if (e.changedTouches && e.changedTouches[0]) {
    e = e.changedTouches[0]
  }

  if (e.clientX || e.clientY) {
    posx = e.clientX
    posy = e.clientY
  }
  else if (e.pageX || e.pageY) {
    posx = e.pageX - document.body.scrollLeft - document.documentElement.scrollLeft
    posy = e.pageY - document.body.scrollTop - document.documentElement.scrollTop
  }
  else {
    const offset = targetElement(e).getBoundingClientRect()
    posx = ((offset.right - offset.left) / 2) + offset.left
    posy = ((offset.bottom - offset.top) / 2) + offset.top
  }

  return {
    top: posy,
    left: posx
  }
}

export function targetElement (e) {
  let target

  if (e.target) {
    target = e.target
  }
  else if (e.srcElement) {
    target = e.srcElement
  }

  // defeat Safari bug
  if (target.nodeType === 3) {
    target = target.parentNode
  }

  return target
}

export function getEventPath (e) {
  if (e.path) {
    return e.path
  }
  if (e.composedPath) {
    return e.composedPath()
  }

  const path = []
  let el = e.target

  while (el) {
    path.push(el)

    if (el.tagName === 'HTML') {
      path.push(document)
      path.push(window)
      return path
    }

    el = el.parentElement
  }
}

// Reasonable defaults
const
  PIXEL_STEP = 10,
  LINE_HEIGHT = 40,
  PAGE_HEIGHT = 800

export function getMouseWheelDistance (e) {
  var
    sX = 0, sY = 0, // spinX, spinY
    pX = 0, pY = 0 // pixelX, pixelY

  // Legacy
  if ('detail' in e) { sY = e.detail }
  if ('wheelDelta' in e) { sY = -e.wheelDelta / 120 }
  if ('wheelDeltaY' in e) { sY = -e.wheelDeltaY / 120 }
  if ('wheelDeltaX' in e) { sX = -e.wheelDeltaX / 120 }

  // side scrolling on FF with DOMMouseScroll
  if ('axis' in e && e.axis === e.HORIZONTAL_AXIS) {
    sX = sY
    sY = 0
  }

  pX = sX * PIXEL_STEP
  pY = sY * PIXEL_STEP

  if ('deltaY' in e) { pY = e.deltaY }
  if ('deltaX' in e) { pX = e.deltaX }

  if ((pX || pY) && e.deltaMode) {
    if (e.deltaMode === 1) { // delta in LINE units
      pX *= LINE_HEIGHT
      pY *= LINE_HEIGHT
    }
    else { // delta in PAGE units
      pX *= PAGE_HEIGHT
      pY *= PAGE_HEIGHT
    }
  }

  // Fall-back if spin cannot be determined
  if (pX && !sX) { sX = (pX < 1) ? -1 : 1 }
  if (pY && !sY) { sY = (pY < 1) ? -1 : 1 }

  /*
   * spinX  -- normalized spin speed (use for zoom) - x plane
   * spinY  -- " - y plane
   * pixelX -- normalized distance (to pixels) - x plane
   * pixelY -- " - y plane
   */
  return {
    spinX: sX,
    spinY: sY,
    pixelX: pX,
    pixelY: pY
  }
}

export function stopAndPrevent (e) {
  e.preventDefault()
  e.stopPropagation()
}


</script>
