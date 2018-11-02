<template>
  <div class="ox-login" v-loading="isLoading">
    <div class="ox-login__logo">
        <img v-if="!isPhone" src="/src/assets/Login_Logo.png"/>
        <img v-if="isPhone" src="/src/assets/logo.png"/>
    </div>
    <div class="ox-login__body">
      <ox-container>
        <ox-header height="60" class="ox-login__legend">
          欢迎登录
        </ox-header>
        <ox-main class="ox-login-main">
          <ox-form ref="loginForm" :model="form" :rules="rules" class="loginForm">
            <ox-form-item prop="userName">
              <ox-input v-model="form.userName" auto-complete="off" placeholder="请输入您的手机号" prefix-icon="ox-icon-mobile-phone" class="ox-login__input"></ox-input>
            </ox-form-item>
            <ox-form-item prop="passWord">
              <ox-input v-model="form.passWord" type="password" placeholder="请输入密码" prefix-icon="ox-icon-login_sercet" class="ox-login__input"></ox-input>
            </ox-form-item>
            <ox-form-item class="ox-checkbox-item">
              <ox-checkbox label="七天免登录，公共场合禁止启用"  v-model="landFlag"></ox-checkbox>
              <a href="#" class="ox-forget-password">忘记密码?</a>
            </ox-form-item>
            <ox-form-item>
              <ox-button type="primary" @click="onSubmit" class="ox-login-btn">登录</ox-button>
            </ox-form-item>
            <ox-form-item>
              <div class="ox-sign">没有账号？<a href="#" class="ox-register-link">立即注册</a></div>
              </ox-form-item>
          </ox-form>
        </ox-main>
      </ox-container>
    </div>
  </div>
</template>

<script>
import config from '@/config/config'
import axios from 'axios'
export default {
  data () {
    let validatePhone = (rule, value, callback) => {
      let reg = 11 && /^((13|14|15|17|18)[0-9]{1}\d{8})$/
      if (value === '') {
        callback(new Error('请输入手机号'))
      } else if (!reg.test(this.form.userName)) {
        callback(new Error('请输入正确的手机号'))
      } else {
        callback()
      }
    }
    return {
      screenWidth: 991,
      isPhone: false,
      svgHtml: '',
      landFlag: true,
      form: {
        userName: '',
        passWord: '',
        captcha: '',
        encryptCaptcha: ''
      },
      rules: {
        userName: [
          {
            validator: validatePhone,
            type: 'number'
          }
        ],
        passWord: [
          {
            required: true,
            message: '请输入密码'
          }
        ],
        captcha: [
          {
            required: true,
            message: '请输入验证码',
            trigger: 'blur'
          }
        ]
      },
      isLoading: false
    }
  },
  created () {
    this.getCaptcha()
  },
  mounted () {
    const that = this
    this.handleInit(that.screenWidth)
    window.onresize = () => {
      this.handleInit(that.screenWidth)
    }
  },
  methods: {
    handleInit (screenWidth) {
      if (document.body.clientWidth < (screenWidth || this.screenWidth)) {
        this.isPhone = true
      } else {
        this.isPhone = false
      }
    },
    // api获取验证码
    getCaptcha () {
      this.encryptCaptcha = ''
      axios.get('http://172.31.65.242:9100/employee/captcha')
        .then((response) => {
          this.svgHtml = response.data.data.data
          this.encryptCaptcha = response.data.data.text
        })
        .catch((error) => {
          // console.log(error)
        })
    },
    // 发送登录信息到后台验证
    checkAction (callback) {
      // 加密密码
      var NodeRSA = require('node-rsa')
      let pubKey = new NodeRSA(config.pubKey)
      let encryptPassword = pubKey.encrypt(this.form.passWord, 'base64')
      let params = {
        system: config.systemAuth,
        phone: this.form.userName,
        password: encryptPassword,
        captcha: this.form.captcha,
        encryptCaptcha: this.encryptCaptcha
      }
      axios.get('http://172.31.65.242:9100/employee/login', {params: params})
        .then((response) => {
          this.isLoading = false
          callback(response)
        })
        .catch((error) => {
          // console.log(error)
        })
    },
    // 提交
    onSubmit () {
      this.$refs.loginForm.validate(result => {
        if (result) {
          this.isLoading = true
          this.checkAction((response) => {
            this.isLoading = false
            if (response.data.success === true) {
              let user = {
                nameEN: response.data.user.nameEN,
                nameZH: response.data.user.nameZH,
                privileges: response.data.user.privileges,
                token: response.data.token
              }
              this.$store.commit('userInfo/update', {
                user: user
              })
              if (this.landFlag) {
                let date = new Date()
                date.setDate(date.getDate() + 7)
                this.$cookies.set('user', JSON.stringify(user), {expires: date.toGMTString()})
              } else {
                this.$cookies.set('user', JSON.stringify(user))
              }
              this.$router.push('/')
            } else {
              this.svgHtml = response.data.captcha.data
              this.encryptCaptcha = response.data.captcha.text
              this.$message({
                message: response.data.message,
                type: 'warning'
              })
            }
          })
        }
      })
    }

  }
}
</script>

<style lang='scss'>
@import url(./login.scss);

</style>
