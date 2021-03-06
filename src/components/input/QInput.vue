<template>
  <q-input-frame
    class="q-input"

    :prefix="prefix"
    :suffix="suffix"
    :stack-label="stackLabel"
    :float-label="floatLabel"
    :error="error"
    :warning="warning"
    :disable="disable"
    :inverted="inverted"
    :dark="dark"
    :hide-underline="hideUnderline"
    :before="before"
    :after="after"
    :color="color"

    :focused="focused"
    :length="length"
    :top-addons="isTextarea"

    @click="__onClick"
    @focus="__onFocus"
  >
    <slot name="before"></slot>

    <template v-if="isTextarea">
      <div class="col row relative-position">
        <q-resize-observable @resize="__updateArea()"></q-resize-observable>
        <textarea
          class="col q-input-target q-input-shadow absolute-top"
          ref="shadow"
          :value="model"
          v-bind="$attrs"
        ></textarea>

        <textarea
          ref="input"
          class="col q-input-target q-input-area"

          :name="name"
          :placeholder="inputPlaceholder"
          :disabled="disable"
          :readonly="readonly"
          :maxlength="maxLength"
          v-bind="$attrs"

          :value="model"
          @input="__set"

          @focus="__onFocus"
          @blur="__onInputBlur"
          @keydown="__onKeydown"
          @keyup="__onKeyup"
        ></textarea>
      </div>
    </template>

    <input
      v-else
      ref="input"
      class="col q-input-target q-no-input-spinner"
      :class="[`text-${align}`]"

      :name="name"
      :placeholder="inputPlaceholder"
      :disabled="disable"
      :readonly="readonly"
      :maxlength="maxLength"
      :step="computedStep"
      v-bind="$attrs"

      :type="inputType"
      :value="model"
      @input="__set"

      @focus="__onFocus"
      @blur="__onInputBlur"
      @keydown="__onKeydown"
      @keyup="__onKeyup"
    />

    <q-icon
      v-if="isPassword && !noPassToggle && length"
      slot="after"
      :name="$q.icon.input[showPass ? 'showPass' : 'hidePass']"
      class="q-if-control"
      @mousedown.native="__clearTimer"
      @touchstart.native="__clearTimer"
      @click.native="togglePass"
    ></q-icon>

    <q-icon
      v-if="keyboardToggle"
      slot="after"
      :name="$q.icon.input[showNumber ? 'showNumber' : 'hideNumber']"
      class="q-if-control"
      @mousedown.native="__clearTimer"
      @touchstart.native="__clearTimer"
      @click.native="toggleNumber"
    ></q-icon>

    <q-icon
      v-if="editable && clearable && length"
      slot="after"
      :name="$q.icon.input.clear"
      class="q-if-control"
      @mousedown.native="__clearTimer"
      @touchstart.native="__clearTimer"
      @click.native="clear"
    ></q-icon>

    <q-spinner
      v-if="isLoading"
      slot="after"
      size="24px"
      class="q-if-control"
    ></q-spinner>

    <slot name="after"></slot>
    <slot></slot>
  </q-input-frame>
</template>

<script>
import FrameMixin from '../../mixins/input-frame'
import InputMixin from '../../mixins/input'
import inputTypes from './input-types'
import { frameDebounce } from '../../utils/debounce'
import { between } from '../../utils/format'
import { QInputFrame } from '../input-frame'
import { QResizeObservable } from '../observables'
import { QSpinner } from '../spinner'

export default {
  name: 'q-input',
  mixins: [FrameMixin, InputMixin],
  components: {
    QInputFrame,
    QSpinner,
    QResizeObservable
  },
  props: {
    value: { required: true },
    type: {
      type: String,
      default: 'text',
      validator: t => inputTypes.includes(t)
    },
    clearable: Boolean,
    noPassToggle: Boolean,
    numericKeyboardToggle: Boolean,
    readonly: Boolean,

    decimals: Number,
    step: Number,
    upperCase: Boolean
  },
  data () {
    return {
      showPass: false,
      showNumber: true,
      model: this.value,
      shadow: {
        val: this.model,
        set: this.__set,
        loading: false,
        hasFocus: () => {
          return document.activeElement === this.$refs.input
        },
        register: () => {
          this.watcher = this.$watch('model', val => {
            this.shadow.val = val
          })
        },
        unregister: () => {
          this.watcher()
        },
        getEl: () => {
          return this.$refs.input
        }
      }
    }
  },
  watch: {
    value (v) {
      this.model = v
      this.isNumberError = false
    }
  },
  provide () {
    return {
      __input: this.shadow
    }
  },
  computed: {
    isNumber () {
      return this.type === 'number'
    },
    isPassword () {
      return this.type === 'password'
    },
    isTextarea () {
      return this.type === 'textarea'
    },
    isLoading () {
      return this.loading || this.shadow.loading
    },
    pattern () {
      if (this.isNumber) {
        return this.$attrs.pattern || '[0-9]*'
      }
    },
    keyboardToggle () {
      return this.$q.platform.is.mobile &&
        this.isNumber &&
        this.numericKeyboardToggle &&
        length
    },
    inputType () {
      if (this.isPassword) {
        return this.showPass ? 'text' : 'password'
      }
      return this.isNumber
        ? (this.showNumber ? 'number' : 'text')
        : this.type
    },
    length () {
      return this.model !== null && this.model !== undefined
        ? ('' + this.model).length
        : 0
    },
    computedStep () {
      return this.step || (this.decimals ? 10 ** -this.decimals : 'any')
    }
  },
  methods: {
    togglePass () {
      this.showPass = !this.showPass
      clearTimeout(this.timer)
      this.focus()
    },
    toggleNumber () {
      this.showNumber = !this.showNumber
      clearTimeout(this.timer)
      this.focus()
    },

    __clearTimer () {
      this.$nextTick(() => clearTimeout(this.timer))
    },

    __setModel (val) {
      clearTimeout(this.timer)
      this.focus()
      this.__set(val || (this.isNumber ? null : ''), true)
    },
    __set (e, force) {
      let val = e && e.target ? e.target.value : e

      if (this.isNumber) {
        const forcedValue = val
        val = parseFloat(val)
        if (isNaN(val)) {
          this.isNumberError = true
          if (force) {
            this.$emit('input', forcedValue)
          }
          return
        }
        this.isNumberError = false
        if (Number.isInteger(this.decimals)) {
          val = parseFloat(val.toFixed(this.decimals))
        }
      }
      else if (this.upperCase) {
        val = val.toUpperCase()
      }

      this.$emit('input', val)
      this.model = val
    },
    __updateArea () {
      const shadow = this.$refs.shadow
      if (shadow) {
        let h = shadow.scrollHeight
        const max = this.maxHeight || h
        this.$refs.input.style.minHeight = `${between(h, 19, max)}px`
      }
    }
  },
  mounted () {
    this.__updateArea = frameDebounce(this.__updateArea)
    if (this.isTextarea) {
      this.__updateArea()
      this.watcher = this.$watch('model', this.__updateArea)
    }
  },
  beforeDestroy () {
    if (this.watcher !== void 0) {
      this.watcher()
    }
  }
}
</script>
