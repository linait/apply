<template>
  <div class="login-container">
    <pic-alert :url='url'></pic-alert>
    <p class="prompt-text">请先用您的手机号登录</p>
    <form class='form-wrap' @submit.prevent="userLoginSubmit()">
      <div class="form-filed">
        <label class="label">手机号</label>
        <input ref="mobileInput" class="value" type='tel' placeholder="请输入手机号" v-model.lazy="myForm.mobile" @keyup.prevent="formatMobile($event)"/>
      </div>
      <div class="form-filed on-border">
        <label class="label">验证码</label>
        <input class="value" type='tel' placeholder="请输入您的验证码" v-model="myForm.verificationCode"/>
        <button class='sms-code' :disabled="!isSendDisable" :class="{'able-activity': isSendDisable}" @click.prevent='sendCode'>{{smsText}}</button>
      </div>
      <button type="submit" class='primary-button login-button' :class="{'btn-disabled': !isLoginDisable}" :disabled="!isLoginDisable">立即登录</button>
    </form>
  </div>
</template>
<script>
  import PicAlert from '../components/picalert'
  import {validMobile, setCaretPosition} from '../utils/util'
  import {getStore} from '../utils/storage'
  import {url} from '../utils/axios'
  import {mapGetters} from 'vuex'
  import _ from 'lodash'
  export default {
    data () {
      return {
        smsCount: 0,
        isLoginDisable: false,
        isSendDisable: false,
        smsText: '发送验证码',
        myForm: {
          mobile: '',
          verificationCode: '',
          captchaCode: '',
          sysSite: '',
          site: ''
        },
        countInterval: '',
        url: url
      }
    },
    computed: {
      ...mapGetters([
        'smsCode'
      ])
    },
    methods: {
      validMobile,
      setCaretPosition,
      getStore,
      // 格式化手机号
      formatMobile (event) {
        const nodeInput = this.$refs.mobileInput
        let position = nodeInput.selectionStart
        let startLen = event.target.value.length
        let value = event.target.value.replace(/\D/g, '')
        if (value.length > 11) {
          value = value.substring(0, 11)
        }
        if (value.length === 11) {
          value = value.replace(/\B(?=(?:\d{4})+$)/g, ' ')
        }
        if (startLen - value.length > 0) {
          position = position - (startLen - value.length - 1)
        } else {
          position = position + Math.abs(startLen - value.length)
        }
        setTimeout(() => {
          this.setCaretPosition(nodeInput, position)
        }, 1)
        nodeInput.value = value
        this.myForm.mobile = value
      },

      // 发送验证码
      sendCode () {
        this.sendSmsCode()
      },

      sendSmsCode (captchaCode) {
        const mobile = this.myForm.mobile.replace(/\D/g, '')
        const appChanel = this.getStore('site')
        const proType = this.getStore('sysSite')
        const captchaId = this.smsCode.verifyCodeCount >= 3 ? 'captchaId' : ''
        const captcha = this.smsCode.verifyCodeCount >= 3 ? captchaCode : ''
        const param = _.assign({}, {mobile, appChanel, captcha, captchaId, proType})
        // const self = this
        this.$store.dispatch('sendSmsCode', {param}).then(({data, code}) => {
          if (code === 'fail' && this.smsCode.verifyCodeCount >= 3) {
            console.log('===')
            this.eventBus.$emit('picAlert/show', true)
            return
          }

          if (code === 'suss' && this.smsCode.verifyCodeCount >= 3) {
            this.eventBus.$emit('picAlert/show', false)
          }

          if (code === 'suss') {
            this.countdown()
          }
        })
      },

      // 60秒倒计时
      countdown () {
        let count = 60
        this.countInterval = setInterval(() => {
          if (count > 0) {
            this.smsText = `${count--}s`
            this.isSendDisable = false
          } else {
            clearInterval(this.countInterval)
            this.smsText = '发送验证码'
            this.isSendDisable = true
          }
        }, 1000)
      },

      // 用户登录
      userLoginSubmit () {
        console.log('login')
        const sysSite = this.getStore('sysSite')
        const site = this.getStore('site')
        const mobile = this.myForm.mobile.replace(/\D/g, '')
        const param = Object.assign(this.myForm, {sysSite, site, mobile})
        let redirect = decodeURIComponent(this.$route.query.redirect || '/')
        this.$store.dispatch('login', {param}).then(({data, code}) => {
          if (code === 'suss') {
            window.clearInterval(this.countInterval)
            this.$router.push({
              path: redirect
            })
          }
        })
      }
    },
    watch: {
      'myForm': {
        deep: true,
        handler: function ({mobile, verificationCode}) {
          this.isSendDisable = this.validMobile(mobile)
          if (this.validMobile(this.myForm.mobile)) {
            if (verificationCode.length > 0) {
              this.isLoginDisable = true
            } else {
              this.isLoginDisable = false
            }
          } else {
            this.isLoginDisable = false
          }
        }
      }
    },
    components: {
      PicAlert
    },
    created () {
      this.eventBus.$on('picAlert/confirm', this.sendSmsCode)
    }
  }
</script>

<style lang="scss">
   @import "../scss/var";
  .login-container {
    overflow: hidden;
    .prompt-text {
        padding: $form-padding-top 0 $form-padding-bottom $primary-padding-bottom;
        padding-left: 0.3rem;
        font-size: $middle-font-size;
    }
    .login-button {
        margin-top: 0.43rem;
    }
    .sms-code {
        height: 1rem;
        position: absolute;
        top: 0;
        right: 0px;
        background-color: $bg-color;
        font-size: $small-font-size;
        padding: 0 0.3rem;
        color: #999;
    }
    .able-activity {
      background-color: #ffefea;
      color: #ff6633
    }
  }
</style>

