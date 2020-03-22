<template>
  <div
    class="formulate-input"
    :data-classification="classification"
    :data-has-errors="hasErrors"
    :data-is-showing-errors="hasVisibleErrors"
    :data-type="type"
  >
    <div class="formulate-input-wrapper">
      <slot
        v-if="context.hasLabel && context.labelPosition === 'before'"
        name="label"
        v-bind="context"
      >
        <label
          class="formulate-input-label formulate-input-label--before"
          :for="context.attributes.id"
          v-text="context.label"
        />
      </slot>
      <slot
        name="element"
        v-bind="context"
      >
        <component
          :is="context.component"
          :context="context"
        >
          <slot v-bind="context" />
        </component>
      </slot>
      <slot
        v-if="context.hasLabel && context.labelPosition === 'after'"
        name="label"
        v-bind="context.label"
      >
        <label
          class="formulate-input-label formulate-input-label--after"
          :for="context.attributes.id"
          v-text="context.label"
        />
      </slot>
    </div>
    <div
      v-if="help"
      class="formulate-input-help"
      v-text="help"
    />
    <FormulateErrors
      v-if="!disableErrors"
      :type="`input`"
      :errors="explicitErrors"
      :field-name="nameOrFallback"
      :validation-errors="validationErrors"
      :show-validation-errors="showValidationErrors"
    />
  </div>
</template>

<script>
import context from './libs/context'
import { shallowEqualObjects, parseRules, snakeToCamel } from './libs/utils'
import nanoid from 'nanoid/non-secure'

export default {
  name: 'FormulateInput',
  inheritAttrs: false, // 组件将不会把未被注册的props呈现为普通的HTML属性，$attrs 将所有传入的prop都拿到
  inject: {
    formulateFormSetter: { default: undefined },
    formulateFormRegister: { default: undefined },
    getFormValues: { default: () => () => ({}) }
  },
  // 自定义组件的v-model
  model: {
    prop: 'formulateValue',
    event: 'input'
  },
  props: {
    type: {
      type: String,
      default: 'text'
    },
    name: {
      type: [String, Boolean],
      default: true
    },
    /* eslint-disable */
    formulateValue: {
      default: ''
    },
    value: {
      default: false
    },
    /* eslint-enable */
    options: {
      type: [Object, Array, Boolean],
      default: false
    },
    optionGroups: {
      type: [Object, Boolean],
      default: false
    },
    id: {
      type: [String, Boolean, Number],
      default: false
    },
    label: {
      type: [String, Boolean],
      default: false
    },
    labelPosition: {
      type: [String, Boolean],
      default: false
    },
    help: {
      type: [String, Boolean],
      default: false
    },
    debug: {
      type: Boolean,
      default: false
    },
    errors: {
      type: [String, Array, Boolean],
      default: false
    },
    validation: {
      type: [String, Boolean, Array],
      default: false
    },
    validationName: {
      type: [String, Boolean],
      default: false
    },
    error: {
      type: [String, Boolean],
      default: false
    },
    errorBehavior: {
      type: String,
      default: 'blur',
      validator: function (value) {
        return ['blur', 'live'].includes(value)
      }
    },
    showErrors: {
      type: Boolean,
      default: false
    },
    imageBehavior: {
      type: String,
      default: 'preview'
    },
    uploadUrl: {
      type: [String, Boolean],
      default: false
    },
    uploader: {
      type: [Function, Object, Boolean],
      default: false
    },
    uploadBehavior: {
      type: String,
      default: 'live'
    },
    preventWindowDrops: {
      type: Boolean,
      default: true
    },
    showValue: {
      type: [String, Boolean],
      default: false
    },
    validationMessages: {
      type: Object,
      default: () => ({})
    },
    validationRules: {
      type: Object,
      default: () => ({})
    },
    checked: {
      type: [String, Boolean],
      default: false
    },
    disableErrors: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      defaultId: nanoid(9),
      localAttributes: {},
      internalModelProxy: this.getInitialValue(), //设当前值
      behavioralErrorVisibility: (this.errorBehavior === 'live'),
      formShouldShowErrors: false,
      validationErrors: [],
      pendingValidation: Promise.resolve()
    }
  },
  // initLifecycle(vm)
  // initEvents(vm)
  // initRender(vm)
  // callHook(vm, 'beforeCreate')
  // initInjections(vm) // resolve injections before data/props
  // initState(vm) //初始化
  // initProvide(vm) // resolve provide after data/props
  // callHook(vm, 'created')

  // initState
//   if (opts.props) initProps(vm, opts.props)//初始化Props
// if (opts.methods) initMethods(vm, opts.methods)//初始化methods
// if (opts.data) {
//   initData(vm)} else {
//   observe(vm._data = {}, true /* asRootData */)}//初始化data
// if (opts.computed) initComputed(vm, opts.computed)//初始化computed

  computed: {
    // 把传入的都搞好了先
    ...context, //初始整个实例
    classification () {
      const classification = this.$formulate.classify(this.type) // vue实例会带上的方法
      return (classification === 'box' && this.options) ? 'group' : classification
    },
    component () {
      return (this.classification === 'group') ? 'FormulateInputGroup' : this.$formulate.component(this.type)
    },
    parsedValidationRules () {
      const parsedValidationRules = {}
      Object.keys(this.validationRules).forEach((key) => {
        parsedValidationRules[snakeToCamel(key)] = this.validationRules[key]
      })
      return parsedValidationRules
    },
    messages () {
      const messages = {}
      Object.keys(this.validationMessages).forEach((key) => {
        messages[snakeToCamel(key)] = this.validationMessages[key]
      })
      return messages
    }
  },
  watch: {
    '$attrs': {
      handler (value) {
        this.updateLocalAttributes(value)
      },
      deep: true
    },
    internalModelProxy (newValue, oldValue) {
      this.performValidation()
      if (!this.isVmodeled && !shallowEqualObjects(newValue, oldValue)) {
        this.context.model = newValue
      }
    },
    formulateValue (newValue, oldValue) {
      if (this.isVmodeled && !shallowEqualObjects(newValue, oldValue)) {
        this.context.model = newValue
      }
    }
  },
  created () {
    this.applyInitialValue() // 把值赋到对象上
    // 注册表明在form里面使用
    if (this.formulateFormRegister && typeof this.formulateFormRegister === 'function') {
      this.formulateFormRegister(this.nameOrFallback, this)
    }
    this.updateLocalAttributes(this.$attrs)
    this.performValidation()
  },
  methods: {
    getInitialValue () {
      // Manually request classification, pre-computed props
      var classification = this.$formulate.classify(this.type)
      classification = (classification === 'box' && this.options) ? 'group' : classification
      if (classification === 'box' && this.checked) {
        return this.value || true
      } else if (Object.prototype.hasOwnProperty.call(this.$options.propsData, 'value') && classification !== 'box') {
        return this.value
      } else if (Object.prototype.hasOwnProperty.call(this.$options.propsData, 'formulateValue')) {
        return this.formulateValue
      }
      return ''
    },
    applyInitialValue () {
      // This should only be run immediately on created and ensures that the
      // proxy and the model are both the same before any additional registration.
      if (
        !shallowEqualObjects(this.context.model, this.internalModelProxy) &&
        // we dont' want to set the model if we are a sub-box of a multi-box field
        (Object.prototype.hasOwnProperty(this.$options.propsData, 'options') && this.classification === 'box')
      ) {
        this.context.model = this.internalModelProxy
      }
    },
    updateLocalAttributes (value) {
      if (!shallowEqualObjects(value, this.localAttributes)) {
        this.localAttributes = value
      }
    },
    performValidation () {
      // 把校验函数和参数处理好
      const rules = parseRules(this.validation, this.$formulate.rules(this.parsedValidationRules))
      // 通过promise的all把所有该校验的都一次过校验完后再触发渲染
      this.pendingValidation = Promise.all(
        rules.map(([rule, args]) => {
          var res = rule({
            value: this.context.model,
            getFormValues: this.getFormValues.bind(this),
            name: this.context.name
          }, ...args)
          res = (res instanceof Promise) ? res : Promise.resolve(res)
          return res.then(res => res ? false : this.getValidationMessage(rule, args))
        })
      )
        .then(result => result.filter(result => result))
        .then(errorMessages => { this.validationErrors = errorMessages })
      return this.pendingValidation
    },
    getValidationMessage (rule, args) {
      return this.getValidationFunction(rule)({
        args,
        name: this.mergedValidationName,
        value: this.context.model,
        vm: this,
        formValues: this.getFormValues()
      })
    },
    getValidationFunction (rule) {
      let ruleName = rule.name.substr(0, 1) === '_' ? rule.name.substr(1) : rule.name
      ruleName = snakeToCamel(ruleName)
      if (this.messages && typeof this.messages === 'object' && typeof this.messages[ruleName] !== 'undefined') {
        switch (typeof this.messages[ruleName]) {
          case 'function':
            return this.messages[ruleName]
          case 'string':
            return () => this.messages[ruleName]
        }
      }
      return (context) => this.$formulate.validationMessage(rule.name, context, this)
    },
    hasValidationErrors () {
      return new Promise(resolve => {
        this.$nextTick(() => {
          this.pendingValidation.then(() => resolve(!!this.validationErrors.length))
        })
      })
    }
  }
}
</script>
