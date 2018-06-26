<template>
  <input
    :placeholder="placeholder"
    @blur="onBlurHandler"
    @input="onInputHandler"
    ref="numeric"
    type="tel"
    v-model="amount"
    v-if="!readOnly"
  >
  <span
    v-else
    ref="readOnly"
  >{{ amount }}</span>
</template>

<script>
import accounting from 'accounting-js'

export default {
  name: 'VueNumeric',

  props: {
    /**
     * Currency symbol.
     */
    currency: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Maximum value allowed.
     */
    max: {
      type: Number,
      default: Number.MAX_SAFE_INTEGER || 9007199254740991,
      required: false,
    },

    /**
     * Minimum value allowed.
     */
    min: {
      type: Number,
      default: Number.MIN_SAFE_INTEGER || -9007199254740991,
      required: false
    },

    /**
     * Enable/Disable minus value.
     */
    minus: {
      type: Boolean,
      default: false,
      required: false
    },

    /**
     * Input placeholder.
     */
    placeholder: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Value when the input is empty
     */
    emptyValue: {
      type: [Number, String],
      default: '',
      required: false
    },

    /**
     * Number of decimals.
     * Decimals symbol are the opposite of separator symbol.
     */
    precision: {
      type: Number,
      default: 0,
      required: false
    },

    /**
     * Thousand separator type.
     * Separator props accept either . or , (default).
     */
    separator: {
      type: String,
      default: ',',
      required: false
    },

    /**
     * Forced thousand separator.
     * Accepts any string.
     */
    thousandSeparator: {
      default: undefined,
      required: false,
      type: String
    },

    /**
     * Forced decimal separator.
     * Accepts any string.
     */
    decimalSeparator: {
      default: undefined,
      required: false,
      type: String
    },

    /**
      * The output type used for v-model.
      * It can either be String or Number (default).
      */
    outputType: {
      required: false,
        type: String,
        default: 'Number'
    },

    /**
     * v-model value.
     */
    value: {
      type: null,
      default: 0,
      required: true
    },

    /**
     * Hide input and show value in text only.
     */
    readOnly: {
      type: Boolean,
      default: false,
      required: false
    },

    /**
     * Class for the span tag when readOnly props is true.
     */
    readOnlyClass: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Position of currency symbol
     * Symbol position props accept either 'suffix' or 'prefix' (default).
     */
    currencySymbolPosition: {
      type: String,
      default: 'prefix',
      required: false
    }
  },

  data: () => {
    return {
      amount: this.value
    }
  },

  computed: {
    /**
     * Define decimal separator based on separator props.
     * @return {String} '.' or ','
     */
    decimalSeparatorSymbol () {
      if (typeof this.decimalSeparator !== 'undefined') return this.decimalSeparator
      if (this.separator === ',') return '.'
      return ','
    },

    /**
     * Define thousand separator based on separator props.
     * @return {String} '.' or ','
     */
    thousandSeparatorSymbol () {
      if (typeof this.thousandSeparator !== 'undefined') return this.thousandSeparator
      if (this.separator === '.') return '.'
      if (this.separator === 'space') return ' '
      return ','
    },

    /**
     * Define format position for currency symbol and value.
     * @return {String} format
     */
    symbolPosition () {
      if (!this.currency) return '%v'
      return this.currencySymbolPosition === 'suffix' ? '%v %s' : '%s %v'
    }
  },

  watch: {
    /**
     * Watch for value change
     * @param {Number} newValue
     */
    value (newValue) {
      if (newValue !== this.amount) this.amount = this.process(newValue)
    },

    /**
     * When readOnly is true, replace the span tag class.
     * @param {Boolean} newValue
     * @param {Boolean} oldValue
     */
    readOnly (newValue, oldValue) {
      if (oldValue === false && newValue === true) {
        this.$nextTick(() => {
          this.$refs.readOnly.className = this.readOnlyClass
        })
      }
    },

    /**
     * Immediately reflect separator changes
     */
    separator () {
      this.amount = this.process(this.value)
    },

    /**
     * Immediately reflect currency changes
     */
    currency () {
      this.amount = this.process(this.value)
    },

    /**
     * Immediately reflect precision changes
     */
    precision () {
      this.amount = this.process(this.value)
    }
  },

  mounted () {
    // Set default value props when placeholder undefined.
    this.$nextTick(() => {
      this.amount = this.process(this.value)

      // Support vue-float-label
      if (!this.readOnly) {
        this.$nextTick(() => {
          let numericEl = this.$refs.numeric
          const event = document.createEvent('HTMLEvents')
          event.initEvent('input', true, false)
          numericEl.dispatchEvent(event)
        });
      }
    });

    // Set read-only span element's class
    if (this.readOnly) this.$refs.readOnly.className = this.readOnlyClass
  },

  methods: {
    /**
     * Handle blur event.
     * @param {Object} evt
     */
    onBlurHandler (evt) {
      this.$emit('blur', evt)
      if (evt.target.value.length && !this.unformat(this.amount)) {
        this.amount = 0
        this.$emit('input', 0)
      } else {
        this.amount = this.process(this.amount)
        this.update(this.amount)
      }
    },

    /**
     * Handle input event.
     */
    onInputHandler (evt) {
    },

    /**
     * Validate value before update the component.
     * @param {Number} value
     */
    process (value) {
      let processedValue = this.unformat(value)
      if (value >= this.max) processedValue = this.max
      if (value <= this.min) processedValue = this.min
      if (!this.minus && value < 0) this.min >= 0 ? processedValue = this.min : processedValue = 0

      return this.format(processedValue)
    },

    /**
     * Update parent component model value.
     * @param {Number} value
     */
    update (value) {
      let emitValue = this.unformat(value);
      if (!emitValue) {
        this.$emit('input', '');
        this.amount = '';
      } else {
        const fixedValue = accounting.toFixed(emitValue, this.precision)
        const output = this.outputType.toLowerCase() === 'string' ? fixedValue : emitValue
        this.$emit('input', output)
      }
    },

    /**
     * Format value using symbol and separator.
     * @param {Number} value
     * @return {String}
     */
    format (value) {
      return accounting.formatMoney(value, {
        symbol: this.currency,
        format: this.symbolPosition,
        precision: Number(this.precision),
        decimal: this.decimalSeparatorSymbol,
        thousand: this.thousandSeparatorSymbol
      })
    },

    /**
     * Remove symbol and separator.
     * @param {Number} value
     * @return {Number}
     */
    unformat (value) {
      const toUnformat = typeof value === 'string' && value === '' ? this.emptyValue : value
      return accounting.unformat(toUnformat, this.decimalSeparatorSymbol)
    }
  }
}
</script>
