<template>
  <div class="login-container">
    <div class="login-box">
      <!-- ref:类似之前获取dom对象的作用;这里是获取loginFormRef组件 ;:model搜集完整的form表单元素,只是为了校验所有元素时方便用的,不用一个个写所有被校验的元素 ;:rules-->
      <el-form ref="loginFormRef" :model="loginForm" :rules="loginFormRules">
        <img src="./logo_index.png" alt />
        <el-form-item prop="mobile">
          <el-input v-model="loginForm.mobile" placeholder="请输入手机号码">
            <i slot="prefix" class="iconfont iconshouji"></i>
          </el-input>
        </el-form-item>
        <el-form-item prop="code">
          <el-input v-model="loginForm.code" placeholder="请输入校验码">
            <i slot="prefix" class="iconfont iconxiaoyan1"></i>
          </el-input>
        </el-form-item>
        <el-form-item prop="xieyi" style="text-align:left">
          <el-checkbox v-model="loginForm.xieyi">我已阅读并同意用户协议和隐私条款</el-checkbox>
        </el-form-item>
        <el-form-item>
          <!-- loading设置按钮等待效果,disabled使得按钮禁止单击 -->
          <el-button :loading="loginForm.isActive" :disabled="loginForm.isActive" type="primary" @click="login()" style="width:100%">登录</el-button>
        </el-form-item>
      </el-form>
    </div>
  </div>
</template>

<script>
import '@/assets/css/global.css'
import './gt'
import '@/assets/font/iconfont.css'
export default {
  data () {
    // 协议是true或者false,不能用require,只能用自定义校验var checkXieyi = (rule, value, callback)...通过validator使用该自定义校验方法
    var checkXieyi = (rule, value, callback) => {
      /* if (!value) {
        return callback(new Error('请遵守协议'))
      }
      callback() */
      value ? callback() : callback(new Error('请遵守协议'))
    }
    return {
      catObj: null,
      loginForm: {
        mobile: '17812166236',
        code: '246810',
        xieyi: true,
        isActive: false
      },
      loginFormRules: {
        mobile: [
          { required: true, message: '手机号码不能为空' },
          { pattern: /^1[35789]\d{9}$/, message: '手机号码格式不对' }
        ],
        code: [{ required: true, message: '校验码不能为空' }],
        xieyi: [{ validator: checkXieyi }]
      }
    }
  },
  methods: {
    login () {
      // console.log(this)
      // 只对表单的input框显然不够,因为点击登录还是会自动跳转到home页面,需要用validate方法在跳转之前对整个表单进行校验
      // validate对整个表单进行校验的方法,参数为一个回调函数.valid为实参
      this.$refs.loginFormRef.validate(valid => {
        // console.log(valid)
        // valid:true 校验通过
        // valid:false 校验失败
        if (!valid) {
          // 只有路由对象才能调用push方法
          return false
        }
        if (this.catObj !== null) {
          return this.catObj.verify()
        }
        this.isActive = true
        var pro = this.$http({
          url: '/mp/v1_0/captchas/' + this.loginForm.mobile,
          method: 'get'
        })
        pro
          .then(result => {
            // result:axios向后端服务器发起请求，后端服务器会返回极验请求的秘钥信息
            console.log(result)
            // challenge: "474907ce64de47184367c59532121d7a"
            // gt: 'f00de9ed073bd781c94509932a309159'
            // new_captcha: true
            // success: 1
            // 从result里边解构下述的data对象出来(对象结构赋值)
            let { data } = result.data
            // 极验秘钥信息 + 用户行为 一并提交给 极验服务器 做判断，查验行为是否正确
            // 调用initGeetest() 函数，生成极验窗口
            window.initGeetest(
              {
                // 以下配置参数来自服务端 SDK
                gt: data.gt,
                challenge: data.challenge,
                offline: !data.success,
                new_captcha: true,
                product: 'float'// 设置窗口样式,还有其他样式
                // popup（弹出式）,float（浮动式）,custom（与popup类似，但是可以自定义弹出区域）
                // 用户必须指定一个页面中的元素作为后续验证弹出的容器，这时验证码会在这块区域内居中显示,
                // 然后调用这个方法captchaObj.appendTo("#captchaBox");
                // 隐藏按钮式； 对于bind类型，需要注意此时appendTo接口调用将无效，验证时调用verify实例方法进行验证。
              },
              (captchaObj) => {
                // 这里可以调用验证实例 captchaObj 的实例方法
                captchaObj.onReady(() => {
                  // 验证码ready之后才能调用verify方法显示验证码
                  this.catObj = captchaObj
                  captchaObj.verify()// 显示验证码窗口
                  this.isActive = false
                })
                  .onSuccess(() => {
                    // 行为校验正确的处理
                    console.log(this) // 这里都要变为箭头函数,this才指向VueComponent
                    this.loginAct()
                  })
                  .onError(() => {
                    // your code
                  })
              }
            )
          })
          .catch(err => {
            console.log(err)
          })
      })
    },
    loginAct () {
      // 真实校验
      let pro = this.$http({
        url: '/mp/v1_0/authorizations',
        method: 'POST',
        data: this.loginForm
      })
      pro
        .then(result => {
          if (result.data.message === 'OK') {
            console.log(result)
            // 服务器端返回的账号数据信息是一个对象，内部有 id/name/photo/token等
            window.sessionStorage.setItem(
              'userinfo',
              JSON.stringify(result.data.data)
            )
            this.$router.push('/home')
          }
        })
        .catch(err => {
          console.log(this)
          // this.$message.error是引用的Message消息提示组件框
          this.$message.error('手机号码或验证码错误:' + err)
        })
    }
  }
}
</script>

<style lang="less" scoped>
.login-container {
  height: 100%;
  background-color: grey;
  display: flex;
  justify-content: center;
  align-items: center;
  background-image: url("./login_bg.jpg");
  .login-box {
    width: 410px;
    height: 340px;
    background-color: #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    .el-form {
      width: 70%;
      text-align: center;
      img {
        width: 60%;
        margin: 10px auto;
      }
    }
  }
}
</style>
