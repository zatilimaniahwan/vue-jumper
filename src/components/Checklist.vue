<template>
  <component :is="tagName">
    <transition :name="transition" :enter-active-class="enterActiveClass" :leave-active-class="leaveActiveClass" @after-leave="doDestroy">
      <span
        ref="popper"
        :class="rootClass"
        v-show="!disabled && showPopper"
        >
       <slot name="contents">{{ content }}</slot>
      </span>
    </transition>
    <slot name="reference"></slot>
  </component>
</template>

<script>
  import Popper from 'popper.js'
  import jump from 'jump.js'
/**
 * Funtion for ready state
 * @param {Object} element - DOM
 * @param {String} event - name of event
 * @param {Object} handler
 * @returns {*}
 */
  function on (element, event, handler) {
    if (element && event && handler) {
      document.addEventListener ? element.addEventListener(event, handler, false) : element.attachEvent('on' + event, handler)
    }
  }
/**
 * Function for destroy state
 * @param {Object} element - DOM
 * @param {String} event - name of event
 * @param {Object} handler
 * @returns {*}
 */
  function off (element, event, handler) {
    if (element && event) {
      document.removeEventListener ? element.removeEventListener(event, handler, false) : element.detachEvent('on' + event, handler)
    }
  }

  export default {
    props: {
      tagName: {
        type: String,
        default: 'span'
      },
      show: {
        type: Boolean
      },
      trigger: {
        type: String,
        default: 'hover',
        validator: value => [
          'focus'
        ].indexOf(value) > -1
      },
      delayOnMouseOver: {
        type: Number,
        default: 10
      },
      delayOnMouseOut: {
        type: Number,
        default: 10
      },
      disabled: {
        type: Boolean,
        default: false
      },
      content: String,
      enterActiveClass: String,
      leaveActiveClass: String,
      boundariesSelector: String,
      reference: {},
      forceShow: {
        type: Boolean,
        default: false
      },
      dataValue: {
        default: null
      },
      appendToBody: {
        type: Boolean,
        default: false
      },
      visibleArrow: {
        type: Boolean,
        default: true
      },
      transition: {
        type: String,
        default: ''
      },
      stopPropagation: {
        type: Boolean,
        default: false
      },
      preventDefault: {
        type: Boolean,
        default: false
      },
      options: {
        type: Object,
        default () {
          return {}
        }
      },
      rootClass: {
        type: String,
        default: ''
      }
    },

    data () {
      return {
        referenceElm: null,
        popperJS: null,
        showPopper: false,
        currentPlacement: '',
        popperOptions: {
          placement: 'bottom',
          computeStyle: {
            gpuAcceleration: false
          }
        }
      }
    },

    watch: {
      /**
       * Object to showPopper
       * @param {Boolean} value - status of object
       */
      showPopper (value) {
        if (value) {
          if (this.show === true) {
            this.enableScrolling(this.referenceElm)
          }
          this.$emit('show', this)
          if (this.popperJS) {
            this.popperJS.enableEventListeners()
          }
          this.updatePopper()
        } else {
          if (this.popperJS) {
            this.popperJS.disableEventListeners()
          }
          this.$emit('hide', this)
        }
      },
      /**
       * Trigger without click /hover
       * @param {Boolean} value -status of object
       */
      forceShow: {
        handler (value) {
          this[value ? 'doShow' : 'doClose']()
        },
        immediate: true
      },
      /**
       * Hide component from display
       * @param {Boolean} value -status of object
       */
      disabled (value) {
        if (value) {
          this.showPopper = false
        }
      }
    },
    created () {
      this.appendedArrow = false
      this.appendedToBody = false
      this.popperOptions = Object.assign(this.popperOptions, this.options)
    },

    mounted () {
      // Data for current element
      this.referenceElm = this.reference || this.$slots.reference[0].elm
      // Data for contents of popper
      this.popper = this.$slots.contents[0].elm
      // Trigger the event
      switch (this.trigger) {
        case 'focus':
          on(this.referenceElm, 'focus', this.onMouseOver)
          on(this.popper, 'focus', this.onMouseOver)
          break
      }
    },

    methods: {
      /**
       * Enable  to scroll
       * @param {String} element - get from this.referenceElm
       * @returns {*}
       */
      enableScrolling (element) {
        jump(element, {
          offset: -200,
          a11y: true
        })
      },
      /**
       * Show popper
       * @returns {*}
       */
      doShow () {
        this.showPopper = true
      },
      /**
       * Close popper
       * @returns {*}
       */
      doClose () {
        this.showPopper = false
      },
      /**
       * Destroy popper by removing the content
       * @returns {Boolean}
       */
      doDestroy () {
        if (this.showPopper) {
          return
        }

        if (this.popperJS) {
          this.popperJS.destroy()
          this.popperJS = null
        }

        if (this.appendedToBody) {
          this.appendedToBody = false
          document.body.removeChild(this.popper.parentElement)
        }
      },
      /**
       * Define popper with contents and options
       * @returns {*}
       */
      createPopper () {
        this.$nextTick(() => {
          if (this.visibleArrow) {
            this.appendArrow(this.popper)
          }

          if (this.appendToBody && !this.appendedToBody) {
            this.appendedToBody = true
            document.body.appendChild(this.popper.parentElement)
          }

          if (this.popperJS && this.popperJS.destroy) {
            this.popperJS.destroy()
          }

          if (this.boundariesSelector) {
            const boundariesElement = document.querySelector(this.boundariesSelector)

            if (boundariesElement) {
              this.popperOptions.modifiers = Object.assign({}, this.popperOptions.modifiers)
              this.popperOptions.modifiers.preventOverflow = Object.assign({}, this.popperOptions.modifiers.preventOverflow)
              this.popperOptions.modifiers.preventOverflow.boundariesElement = boundariesElement
            }
          }

          this.popperOptions.onCreate = () => {
            this.$emit('created', this)
            this.$nextTick(this.updatePopper)
          }
          this.popperJS = new Popper(this.referenceElm, this.popper, this.popperOptions)
        })
      },
      /**
       * Destroy popper when trigger
       * @returns {*}
       */
      destroyPopper () {
        off(this.referenceElm, 'focus', this.doShow)
        this.showPopper = false
        this.doDestroy()
      },
      /**
       * Append arrow with popper
       * @returns {*}
       */
      appendArrow (element) {
        if (this.appendedArrow) {
          return
        }

        this.appendedArrow = true

        const arrow = document.createElement('div')
        arrow.setAttribute('x-arrow', '')
        arrow.className = 'popper__arrow'
        element.appendChild(arrow)
      },
      /**
       * Update info for popper
       * @returns {*}
       */
      updatePopper () {
        this.popperJS ? this.popperJS.scheduleUpdate() : this.createPopper()
      },
      /**
       * Run when the element is focus
       * @returns {*}
       */
      onMouseOver () {
        clearTimeout(this._timer)
        this._timer = setTimeout(() => {
          this.showPopper = true
        }, this.delayOnMouseOver)
      },
      /**
       * Contents inside the element
       * @returns {*}
       */
      elementContains (elm, otherElm) {
        if (typeof elm.contains === 'function') {
          return elm.contains(otherElm)
        }

        return false
      }
    },
    /**
     * Trigger when element had been destroyed
     * @returns {*}
     */
    destroyed () {
      this.destroyPopper()
    }
  }
</script>
<style>
  .popper {
    width: auto;
    background-color: #fafafa;
    color: #212121;
    text-align: left;
    padding: 10px;
    display: inline-block;
    border-radius: 3px;
    position: absolute;
    font-size: 14px;
    font-weight: normal;
    border: 1px #ebebeb solid;
    z-index: 200000;
    -moz-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    -webkit-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  }
  .popper-mobile {
     width: auto;
    background-color: #fafafa;
    color: #212121;
    text-align: left;
    padding: 10px;
    display: inline-block;
    border-radius: 3px;
    position: absolute;
    font-size: 12px;
    font-weight: normal;
    border: 1px #ebebeb solid;
    z-index: 200000;
    -moz-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    -webkit-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  }
  .popper-mobile-centrix {
    width: auto;
    background-color: #fafafa;
    color: #212121;
    text-align: left;
    padding: 10px;
    margin-left: 90px;
    display: inline-block;
    border-radius: 3px;
    position: absolute;
    font-size: 12px;
    font-weight: normal;
    border: 1px #ebebeb solid;
    z-index: 200000;
    -moz-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    -webkit-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  }
  .popper .popper__arrow {
    width: 0;
    height: 0;
    border-style: solid;
    position: absolute;
    margin: 5px;
  }

  .popper[x-placement^="top"] {
    margin-bottom: 5px;
  }

  .popper[x-placement^="top"] .popper__arrow {
    border-width: 5px 5px 0 5px;
    border-color: #fafafa transparent transparent transparent;
    bottom: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
  }

  .popper[x-placement^="bottom"] {
    margin-top: 5px;
  }

  .popper[x-placement^="bottom"] .popper__arrow {
    border-width: 0 5px 5px 5px;
    border-color: transparent transparent #fafafa transparent;
    top: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
    margin-left: 15px;
  }

  .popper[x-placement^="right"] {
    margin-left: 5px;
  }

  .popper[x-placement^="right"] .popper__arrow {
    border-width: 5px 5px 5px 0;
    border-color: transparent #fafafa transparent transparent;
    left: -5px;
    top: calc(50% - 50px);
    margin-left: 0;
    margin-right: 0;
  }

  .popper[x-placement^="left"] {
    margin-right: 5px;
  }

  .popper[x-placement^="left"] .popper__arrow {
    border-width: 5px 0 5px 5px;
    border-color: transparent transparent transparent #fafafa;
    right: -5px;
    top: calc(50% - 5px);
    margin-left: 0;
    margin-right: 0;
  }
</style>
