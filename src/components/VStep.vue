<template>
  <div class="v-step" :id="'v-step-' + hash" :ref="'v-step-' + hash">
    <slot name="header">
      <div v-if="step.header" class="v-step__header">
        <div v-if="step.header.title" v-html="step.header.title"></div>
      </div>
    </slot>

    <slot name="content">
      <div class="v-step__content">
        <div v-if="step.content" v-html="step.content"></div>
        <div v-else>This is a demo step! The id of this step is {{ hash }} and it targets {{ step.target }}.</div>
      </div>
    </slot>

    <slot name="actions">
      <div class="v-step__buttons">
        <button @click.prevent="skip" v-if="!isLast && isButtonEnabled('buttonSkip')" class="v-step__button v-step__button-skip">{{ labels.buttonSkip }}</button>
        <button @click.prevent="previousStep" v-if="!isFirst && isButtonEnabled('buttonPrevious')" class="v-step__button v-step__button-previous">{{ labels.buttonPrevious }}</button>
        <button @click.prevent="nextStep" v-if="!isLast && isButtonEnabled('buttonNext')" class="v-step__button v-step__button-next">{{ labels.buttonNext }}</button>
        <button @click.prevent="finish" v-if="isLast && isButtonEnabled('buttonStop')" class="v-step__button v-step__button-stop">{{ labels.buttonStop }}</button>
      </div>
    </slot>

    <div class="v-step__arrow" :class="{ 'v-step__arrow--dark': step.header && step.header.title }"></div>
  </div>
</template>

<script>
import { createPopper } from '@popperjs/core'
import jump from 'jump.js'
import sum from 'hash-sum'
import { DEFAULT_STEP_OPTIONS, HIGHLIGHT } from '../shared/constants'

export default {
  name: 'v-step',
  props: {
    step: {
      type: Object
    },
    previousStep: {
      type: Function
    },
    nextStep: {
      type: Function
    },
    stop: {
      type: Function
    },
    skip: {
      type: Function,
      default: function () {
        this.stop()
      }
    },
    finish: {
      type: Function,
      default: function () {
        this.stop()
      }
    },
    isFirst: {
      type: Boolean
    },
    isLast: {
      type: Boolean
    },
    labels: {
      type: Object
    },
    enabledButtons: {
      type: Object
    },
    highlight: {
      type: Boolean
    },
    stopOnFail: {
      type: Boolean
    },
    debug: {
      type: Boolean
    }
  },
  data () {
    return {
      hash: sum(this.step.target),
      targetElement: document.querySelector(this.step.target)
    }
  },
  computed: {
    params () {
      return {
        ...DEFAULT_STEP_OPTIONS,
        ...{ highlight: this.highlight }, // Use global tour highlight setting first
        ...{ enabledButtons: Object.assign({}, this.enabledButtons) },
        ...this.step.params // Then use local step parameters if defined
      }
    }
  },
  methods: {
    createStep () {
      if (this.debug) {
        console.log('[Vue Tour] The target element ' + this.step.target + ' of .v-step[id="' + this.hash + '"] is:', this.targetElement)
      }

      if (this.targetElement) {
        this.enableScrolling()
        this.createHighlight()

        /* eslint-disable no-new */
        createPopper(
          this.targetElement,
          this.$refs['v-step-' + this.hash],
          this.params
        )
      } else {
        if (this.debug) {
          console.error('[Vue Tour] The target element ' + this.step.target + ' of .v-step[id="' + this.hash + '"] does not exist!')
        }
        this.$emit('targetNotFound', this.step)
        if (this.stopOnFail) {
          this.stop()
        }
      }
    },
    enableScrolling () {
      if (this.params.enableScrolling) {
        if (this.step.duration || this.step.offset) {
          let jumpOptions = {
            duration: this.step.duration || 1000,
            offset: this.step.offset || 0,
            callback: undefined,
            a11y: false
          }

          jump(this.targetElement, jumpOptions)
        } else {
          // Use the native scroll by default if no scroll options has been defined
          this.targetElement.scrollIntoView({ behavior: 'smooth' })
        }
      }
    },
    isHighlightEnabled () {
      if (this.debug) {
        console.log(`[Vue Tour] Highlight is ${this.params.highlight ? 'enabled' : 'disabled'} for .v-step[id="${this.hash}"]`)
      }
      return this.params.highlight
    },
    createHighlight () {
      if (this.isHighlightEnabled()) {
        document.body.classList.add(HIGHLIGHT.CLASSES.ACTIVE)
        const transitionValue = window.getComputedStyle(this.targetElement).getPropertyValue('transition')

        // Make sure our background doesn't flick on transitions
        if (transitionValue !== 'all 0s ease 0s') {
          this.targetElement.style.transition = `${transitionValue}, ${HIGHLIGHT.TRANSITION}`
        }

        this.targetElement.classList.add(HIGHLIGHT.CLASSES.TARGET_HIGHLIGHTED)
        // The element must have a position, if it doesn't have one, add a relative position class
        if (!this.targetElement.style.position) {
          this.targetElement.classList.add(HIGHLIGHT.CLASSES.TARGET_RELATIVE)
        }
      } else {
        document.body.classList.remove(HIGHLIGHT.CLASSES.ACTIVE)
      }
    },
    removeHighlight () {
      if (this.isHighlightEnabled()) {
        const target = this.targetElement
        const currentTransition = this.targetElement.style.transition
        this.targetElement.classList.remove(HIGHLIGHT.CLASSES.TARGET_HIGHLIGHTED)
        this.targetElement.classList.remove(HIGHLIGHT.CLASSES.TARGET_RELATIVE)
        // Remove our transition when step is finished.
        if (currentTransition.includes(HIGHLIGHT.TRANSITION)) {
          setTimeout(() => {
            target.style.transition = currentTransition.replace(`, ${HIGHLIGHT.TRANSITION}`, '')
          }, 0)
        }
      }
    },
    isButtonEnabled (name) {
      return this.params.enabledButtons.hasOwnProperty(name) ? this.params.enabledButtons[name] : true
    }
  },
  mounted () {
    this.createStep()
  },
  destroyed () {
    this.removeHighlight()
  }
}
</script>
