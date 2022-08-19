<template>
  <el-dialog
    title="机器手启动连接"
    :modal="true"
    :append-to-body="true"
    v-model:visible="RobotHandStatus"
    :close-on-click-modal="false"
    :close-on-press-escape="false"
    :before-close="handleClose"
    width="480px"
  >
    <div
      class="contentBody"
      v-loading="bodyLoading"
      element-loading-background="rgba(0, 0, 0, 0)"
    >
      <!-- 激活的步骤条 -->
      <section
        class="stepsbox"
        v-if="stepsAcc <= 3"
        :class="{ no_pd: bankData.bankType == 'UBS' }"
      >
        <el-steps
          :active="stepsAcc"
          finish-status="success"
          align-center="center"
        >
          <el-step
            v-for="item of stepList"
            :title="item.name"
            :key="item.code"
            :class="item.actives.includes(item.code) ? 'ischarDep' : ''"
          ></el-step>
        </el-steps>
      </section>
      <!-- 机器手启动 -->
      <section class="isComponent step0" v-if="stepActive == 0">
        <p class="linking">机器手启动连接中，请稍候…</p>
      </section>

      <!-- 密码登录 -->
      <section class="isComponent step2" v-if="stepActive == 'pwd'">
        <div class="cell-box">
          <span>密码后四位：</span>
          <CellCheck
            :key="1"
            :code="pwdDefault"
            v-model:NextBtnStatus="NextBtnStatus"
            @complete="pwdCallBack"
            :reset="resetCode"
          />
          <!-- TODO: 后续迁移以下方密码器组件迁移为主  -->
          <!-- <CellCheckLast/>     -->
        </div>
        <div class="bottom1">
          <el-button
            class="v-define"
            :disabled="NextBtnStatus < 4"
            @click="onNextFun"
            >下一步</el-button
          >
        </div>
      </section>

      <!-- IBK -->
      <section class="isComponent ibk" v-if="stepActive == 'IBK'">
        <div class="cell-box">
          <P style="text-align: center; margin-bottom: 16px"
            ><span style="margin-right: 16px">IB Key</span>{{ IBkey }}
          </P>
        </div>
        <CellCheck
          :maxlength="8"
          :code="IBKDefault"
          v-model:NextBtnStatus="NextBtnStatus"
          @complete="otpCallBackValid"
          :reset="resetCode"
        />
        <p class="error" v-if="cellErrorText">
          {{ cellErrorText }}
        </p>
        <div class="bottom1" :class="{ noBtm: !!cellErrorText }">
          <el-button
            class="v-define"
            :disabled="NextBtnStatus < 8"
            @click="nextYTvlid"
            >下一步</el-button
          >
        </div>
      </section>

      <!-- 新鸿基 -->
      <section class="isComponent ibk" v-if="stepActive == 'XinHongJi'">
        <div class="cell-box">
          <P
            style="
              text-align: center;
              margin-bottom: 10.5px;
              padding: 0 79px;
              line-height: 22px;
            "
            >请在已登记的保安登入装置产生您的一次性登入保安编码,并输入有关编码。</P
          >
        </div>
        <CellCheck
          :maxlength="6"
          :code="cellDefault"
          v-model:NextBtnStatus="NextBtnStatus"
          @complete="otpCallBackValid"
          :reset="resetCode"
        />
        <p class="error" style="padding-left: 110px" v-if="!!cellErrorText">
          {{ cellErrorText }}
        </p>
        <div class="bottom1" :class="{ noBtm: !!cellErrorText }">
          <el-button
            class="v-define"
            :disabled="NextBtnStatus < 6"
            @click="nextOtpFun"
            >下一步</el-button
          >
        </div>
      </section>

      <!-- 招商永隆验证码-->
      <section class="isComponent step1" v-if="stepActive == 'CMB-1'">
        <div class="tops">
          <section
            class="forms"
            style="
              width: 344px;
              justify-content: space-between;
              padding-right: 12px;
            "
            :class="verificationCodeError ? 'formErr' : ''"
          >
            <span style="padding-right: 6px">图形验证码</span>
            <div class="isbody el_input_change">
              <el-input
                size="small"
                v-model="VerificationCode"
                maxlength="6"
                placeholder="请输入图形验证码"
                @blur="clearVer"
                @focus="clearVer"
              />
              <p class="errorText" v-if="verificationCodeError">
                图形验证码输入有误，请重新输入
              </p>
            </div>
          </section>
          <section class="imgbox">
            <img :src="verificationCodeImag" alt="" srcset="" />
            <span @click="CMBUpLoadVerCode()">刷新图像</span>
          </section>
        </div>
        <div class="bottom">
          <!-- <el-button class="v-reset" @click="next">取消</el-button> -->
          <el-button
            class="v-define"
            :disabled="VerificationCode && VerificationCode.length <= 3"
            @click="checkVerificationCode(4)"
            >下一步</el-button
          >
        </div>
      </section>

      <!-- 招商OTP  TODO: 重写style 后续迁移以这个为准  stepActive == 'CMB-2'||stepActive == 'UobKayHianOTP' ||stepActive ==  'CMB-3' -->
      <section
        class="isComponent step2"
        v-if="['CMB-2', 'UobKayHianOTP', 'CMB-3'].includes(stepActive)"
      >
        <div
          class="cell-box"
          :class="{ pd_lt_56: bankData.bankType == 'CMSInternational' }"
        >
          <span>OTP：</span>
          <CellCheck
            :key="2"
            ref="pwdSatusRef"
            :isError="cellErrorText && isShowError"
            :maxlength="6"
            v-model:NextBtnStatus="NextBtnStatus"
            :code="cellDefault"
            @complete="otpCallBackValid"
            :reset="resetCode"
          />
          <span
            v-if="bankData.bankType == 'CMSInternational'"
            class="get_vlid_info_satus"
            :class="!!cellErrorText ? 'active_class' : ''"
            @click="sendAgainFunc"
            >重新发送</span
          >
        </div>
        <p
          class="error"
          v-if="!!cellErrorText && isShowError"
          style="text-align: left"
          :style="{
            'padding-left':
              bankData.bankType == 'CMSInternational' ? '126px' : '131px'
          }"
        >
          {{ cellErrorText }}
        </p>
        <div class="bottom1" :class="{ noBtm: !!cellErrorText && isShowError }">
          <el-button
            class="v-define"
            :disabled="NextBtnStatus < 6"
            @click="nextOtpFun"
            >下一步</el-button
          >
        </div>
      </section>

      <!-- 招商永隆输入OTP  -->
      <!-- <section class="isComponent step2"  v-if="stepActive == 'CMB-2'||stepActive == 'UobKayHianOTP'" >
        <div  class="cell-box">
          <span :class="!!cellErrorText?'mg_bm_12':''">OTP：</span>
          <div>
             <CellCheck :isError="cellErrorText" :isShowBefore ='true' :NextBtnStatus.sync="NextBtnStatus" :code="cellDefault" @complete="otpCellBack" :reset="resetCode" />
             <p class="error"  v-if="!!cellErrorText"  style="padding-left:0;text-align: left;">
            {{cellErrorText}}
          </p>
          </div>
          <span v-if="bankData.bankType !=='CMBWORLD'" class="get_vlid_info_satus" :class="!!cellErrorText?'active_class':''" @click="sendAgainFunc">重新发送</span>
        </div>
        
         <div class="bottom1" :class="{noBtm:!!cellErrorText}">
          <el-button class="v-define" :disabled="NextBtnStatus<6"
            @click="nextOtpFun">下一步</el-button>
        </div>
      </section> -->

      <!-- 盈通输入验证码 -->
      <section
        class="isComponent step1 IBKR"
        v-if="stepActive == 'IBKR-1' || stepActive == 'IBKR-2'"
      >
        <div v-if="selectValid == 'selects'" style="padding: 30px 40px 0">
          <el-select
            style="width: 100%"
            v-model="value"
            placeholder="请选择"
            @change="onChangeSelect(value)"
          >
            <el-option label="短信验证码" :value="1"></el-option>
            <el-option label="Digital Security Card" :value="2"></el-option>
            <el-option label="IB key" :value="3"></el-option>
          </el-select>
        </div>
        <div v-else-if="selectValid == 1">
          <div class="tops">
            <section
              class="forms"
              :class="verificationCodeError ? 'formErr' : ''"
              v-if="stepActive == 'IBKR-1'"
            >
              <span>验证码：</span>
              <div class="isbody">
                <el-input
                  size="small"
                  v-model="VerificationCode"
                  maxlength="8"
                  placeholder="请输入短信验证码"
                >
                </el-input>
                <span class="v-btn-text">{{ IBKR.IBKRNums }}s</span>
              </div>
            </section>
            <section v-else class="choose-method">
              <div class="form-text">请选择验证方式({{ IBKRPhoneCode }})</div>
            </section>
          </div>
          <div class="bottom" v-if="stepActive == 'IBKR-1'">
            <el-button
              class="v-define"
              :disabled="VerificationCode && VerificationCode.length <= 3"
              @click="checkVerificationCode"
              >下一步</el-button
            >
          </div>
          <div class="bottom" v-else>
            <el-button class="v-plain" @click="phoneVer(1)">电话验证</el-button>
            <el-button class="v-define" @click="phoneVer(2)"
              >短信验证</el-button
            >
          </div>
        </div>
      </section>

      <!-- 瑞银二维码 -->
      <section class="isComponent QRcode" v-if="stepActive == 'UBS'">
        <p style="color: #5a5e63">
          请用手机App Access扫描二维码，并输入登录密码
        </p>
        <img :src="QRcodeUrl" alt="" />
      </section>

      <!-- 富途输入验证码   -->
      <section
        class="isComponent step1 futu-class"
        v-if="stepActive == 'FUTU' || stepActive == 'FUTU1'"
      >
        <div class="tops" style="padding-left: 10px">
          <section
            class="forms"
            :class="verificationCodeError ? 'formErr' : ''"
            v-if="futuNeedCode == 1"
          >
            <span style="width: 80px; margin-right: 8px">短信验证码</span>
            <div class="isbody">
              <el-input
                size="small"
                style="width: 260px !important"
                v-model="VerificationCode"
                maxlength="8"
                placeholder="请输入短信验证码"
              >
              </el-input>
              <span
                class="v-btn-text"
                :class="{ hasClick: !canPost }"
                @click="futuInterval(1)"
                >{{
                  futuInfo.countdownText === 'again'
                    ? '重新发送'
                    : futuInfo.countdownText
                }}</span
              >
            </div>
          </section>
          <section
            class="forms center futu-forms-1"
            style="margin-top: 10px"
            v-else
          >
            本次连接无需短信验证，<span class="second-time"
              >{{ futuInfo.passTime }}s</span
            >
            后将自动跳转到下一步
          </section>
        </div>
        <div class="bottom" v-if="futuNeedCode == 1">
          <el-button
            class="v-define"
            :disabled="VerificationCode && VerificationCode.length <= 3"
            @click="checkVerificationCode(1)"
            >下一步</el-button
          >
        </div>
      </section>
      <!-- 富途输入验证码   -->
      <section
        class="isComponent step1 futu-class"
        v-if="stepActive == 'FUTU_NO'"
      >
        <div class="tops" style="padding-left: 10px">
          <section class="forms center futu-forms-1" style="margin-top: 10px">
            本次连接无需图形验证，<span class="second-time"
              >{{ futuInfo.passTime }}s</span
            >
            后将自动跳转到下一步
          </section>
        </div>
      </section>

      <!-- 机器手链接中    v-if="stepActive == 'pwd'3"-->
      <section class="isComponent step3" v-if="stepActive == 3">
        <p class="progressbox">
          机器手已连接<span> {{ progressBar }}%</span>，请稍候…
        </p>
        <div class="progressBar">
          <el-progress
            :percentage="progressBar"
            color="#F38700"
            :show-text="false"
          ></el-progress>
        </div>
      </section>

      <!-- 机器手连接失败  -->
      <section
        class="isComponent psdError"
        style="padding-top: 40px"
        v-if="stepActive == 4"
      >
        <div class="top">
          <!--<i class="el-icon-close"></i>  -->
          <div class="isIcon">
            <svg-icon icon-class="robot_toast_error"></svg-icon>
          </div>
          <p>{{ errorTitle ? '机器手连接失败' : '登录密码错误' }}</p>
        </div>
        <div class="middle">
          <div v-if="!errorTitle">
            <p style="padding: 0 60px">
              登录密码错误，请确保您输入的密码正确，若连续{{
                this.bankData.bankType == 'Futu' ? '10次' : '5次'
              }}错误，您的银行/券商将被锁住
            </p>
            <!-- <p>若连续{{this.bankData.bankType=='Futu'?'8次':'5次'}}错误，您的网上/手机银行服务将被暂停</p> -->
          </div>
          <div v-else>
            <p>{{ errorTitle }}</p>
            <a :href="isSub"></a>
          </div>
        </div>
        <!-- <div class="bottom">
          <el-button class="v-define" @click="toAccountEdit()">去修改</el-button>
        </div> -->
      </section>

      <!-- 机器手连接成功 -->
      <section class="isComponent successLink" v-if="stepActive == 5">
        <div>
          <!-- <i class="el-icon-check"></i> -->
          <div class="isIcon">
            <svg-icon icon-class="robot_toast_success"></svg-icon>
          </div>
          <p>机器手连接成功！</p>
        </div>
      </section>

      <!-- 机器手失败or异常 v-if="stepActive == 6"  -->
      <section
        class="isComponent psdError"
        :class="{ pd_tp: !!netInfo }"
        v-if="stepActive == 6"
      >
        <div class="top">
          <div class="isIcon">
            <svg-icon icon-class="robot_toast_error"></svg-icon>
          </div>
          <p>{{ errorTitle ? errorTitle : '机器手连接失败' }}</p>
        </div>
        <div v-if="netInfo" class="middle">
          <p>{{ netInfo }}</p>
        </div>
      </section>
    </div>
  </el-dialog>
</template><script>
import CellCheck from '@/components/cell-check/index.vue'
// import CellCheckLast from "@/components/cell-check-last/index.vue";
import { robotHandContent } from '@/api/bank-account/index'
import { stepsMenu } from '@/views/bank-account/components/robotHand/robotsMenu'
export default {
  name: 'robotHand',
  components: {
    CellCheck
    // CellCheckLast
  },
  props: {
    visibleStatus: {
      type: Number,
      default: 0
    },
    bankData: {
      type: Object,
      default: {}
    }
  },
  data() {
    return {
      bankType: 0, //0招商永隆  1大华  2盈透  bankEnum
      stepList: [],
      stepActive: 0, //0 启动链接 1 填写验证码  2 Top  3 机器手正在链接 4 机器手连接失败 5 机器手连接成功 6其他异常导致的报错 7盈透验证码错误
      RobotHandStatus: false,
      verificationCodeImag: '',
      VerificationCode: '', //验证码
      verificationCodeError: false, //验证码错误
      cellDefault: ['', '', '', '', '', ''], //默认验证码
      pwdDefault: ['', '', '', ''], //密码后四位
      IBKDefault: ['', '', '', '', '', '', '', ''], //ibk
      resetCode: '', //清空OTP验证码
      cellErrorText: '', //OTP错误验证码
      progressBar: 0, //进度条比例
      bodyLoading: false, //加载状态
      IBKRPhoneCode: null,
      IBKRVisible: false,
      isPWD: false, // 经历登录密码
      selectValid: 0, // 盈透选择验证方式
      timerUBS: null, // 轮询查询二维码
      value: '',
      isSub: '', // 截取地址
      requestType: 4, // 验证码
      netInfo: '',
      IBkey: '',
      pwdContent: '',
      flag: false,
      NextBtnStatus: 0, // 下一步按钮是否可点击
      count: 12,
      isShowError: false, // 聚焦是否显示错误
      isAgainOtp: false, // 是否重新发送

      IBKR: {
        IBKRNums: 60,
        IBKRCount: 0
      },
      futuNeedCode: 0, // 1 富途需要验证码  2.已经提交完验证码，验证是否连接成功
      futuInfo: {
        countdown: 10,
        countdownText: '01:00',
        passTime: 3
      },
      QRcodeUrl: '', //瑞银二维码
      FutuTimer: null,
      canPost: true,
      countdown3s: null, // 非首次连接，3秒倒计时
      successTimer: null // 非首次连接，关闭3秒延迟的提示
    }
  },
  computed: {
    // 当前步骤
    stepsAcc() {
      console.log('步骤条', this.stepActive)
      let _result = 0
      if (this.stepActive == 0) {
        _result = 0
      } else if (['pwd', 'UBS'].includes(this.stepActive)) {
        _result = 1
      } else if (
        [
          'CMB-1',
          'IBKR-1',
          'IBKR-2',
          'FUTU',
          'IBK',
          'UobKayHianOTP',
          'CMB-3',
          'XinHongJi',
          'FUTU_NO'
        ].includes(this.stepActive)
      ) {
        _result = 2
      } else if (['CMB-2', '3', 'FUTU1'].includes(this.stepActive)) {
        _result = 3
      } else if (this.stepActive == 4) {
        _result = 4
      } else {
        _result = 100
      }
      return _result
    }
  },
  watch: {
    visibleStatus(news, olds) {
      if (olds != news) {
        this.RobotHandStatus = true
        switch (this.bankData.bankType) {
          case 'UBS': //瑞银
            this.stepList = stepsMenu.UBS
            break
          case 'CMBWingLung': //招商永隆
            this.stepList = stepsMenu.CMB
            break
          case 'UobKayHian': //大华
            this.stepList = stepsMenu.UOB
            break
          case 'InteractiveBrokers': //盈透
            this.stepList = stepsMenu.IBKR
            break
          case 'Futu': // 富途
            this.stepList = stepsMenu.FUTU
            break
          case 'XinHongJi': // 新鸿基
            this.stepList = stepsMenu.SUNHUNKAI
            break
          case 'CMSInternational': // 招商国际
            this.stepList = stepsMenu.CMBWORLD
            break
          default:
            //招商永隆
            this.stepList = stepsMenu.CMB
            break
        }
      }
      true
    },
    bankData(news, olds) {
      if (!_.isEmpty(news)) {
        if (news.phoneNumber) {
          let code =
            news.phoneNumber.length < 5
              ? '00000' + news.phoneNumber
              : news.phoneNumber
          this.IBKRPhoneCode = '****' + code.substr(-4)
        }
        // let parmas = {
        //   tradingAccountId: Number(news.tradingAccountId),
        //   // requiredServerEventId: 1  // 枚举: 0, 1启动链接 2校验验证码 3输入OTP回调 4短信验证
        //   serverEventId:0  //0-其他 1-发送登录账号密码 2-发送验证码 3-发送一次性密码 4-发送验证码请求方式(例如盈透发送该事件)
        // }
        // this.contentLink(parmas)
        console.log('news.bankType', news.bankType)
        this.stepActive = 0
        if (news.bankType != 'UBS') {
          setTimeout(() => {
            if (news.bankType != 'UBS') {
              this.stepActive = 'pwd'
            }
          }, 200)
        } else {
          let params = {
            tradingAccountId: Number(news.tradingAccountId),
            requiredServerEventId: 1
          }
          this.count = 12
          this.contentLink(params)
        }
      }
    }
  },
  methods: {
    /**
     * 密码下一步
     * @param {*} val
     */
    onNextFun() {
      let params = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 1, //1,发送登录账号密码,
        param: this.pwdContent.join('')
      }
      this.bodyLoading = true
      this.contentLink(params)
      this.isPWD = true
      this.pwdContent = ''
    },
    // 校验验证码
    checkVerificationCode(val) {
      this.flag = false
      let parmas = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 2,
        requestType: val, // 4图形验证码 1手机验证  2短信验证
        param: this.VerificationCode
      }
      this.bodyLoading = true
      // 盈透
      if (this.bankData.bankType == 'InteractiveBrokers') {
        this.stepActive = 3
        this.robotProgress()
        this.bodyLoading = false
        this.IBKR = {
          IBKRNums: 60,
          IBKRCount: 0
        }
        this.IBKRVisible = true
        this.clearAllTimer()
        parmas.requiredServerEventId = 4
      }
      this.contentLink(parmas)
    },
    // otpCellBack(data) {//输入OTP回调
    //   this.pwdContent = data;
    //   this.cellDefault = ['', '', '', '', '', ''];
    // },

    /**
     * OTP下一步
     */
    nextOtpFun() {
      let parmas = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 3, //3发送一次性密码,
        param: this.pwdContent.join('')
      }
      this.bodyLoading = true
      this.contentLink(parmas)
    },

    /**
     * 盈透校验验证码
     */
    otpCallBackValid(data) {
      this.pwdContent = data
    },

    nextYTvlid() {
      let parmas = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 2,
        param: this.pwdContent.join(''),
        requestType: 3
      }
      this.bodyLoading = true
      this.contentLink(parmas)
    },

    /**
     * 重新发送OTP验证码 招商国际
     */
    sendAgainFunc() {
      if (!this.cellErrorText) return
      // this.resetOTP();
      this.cellErrorText = ''
      this.cellDefault = ['', '', '', '', '', '']
      this.$refs.pwdSatusRef.$refs[`firstinput0`][0].focus()
      let parmas = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 3,
        requestType: 3,
        param: ''
      }
      this.bodyLoading = true
      this.isAgainOtp = true
      this.contentLink(parmas)
      // Promise.resolve().then(() => {
      //   this.$message.success('OTP发送成功');
      // })
    },

    /**
     * 填入密码后四位的回调
     */
    pwdCallBack(data) {
      this.pwdContent = data
    },
    toAccountEdit() {
      //修改银行账号密码
      this.$router.push({
        path: '/bank-manage/set',
        query: {
          id: this.bankData.tradingAccountId
        }
      })
    },
    robotProgress() {
      //机器手回调成功
      clearTimeout(this.robotInter)
      this.robotInter = setTimeout(() => {
        clearTimeout(this.robotInter)
        if (this.progressBar <= 99) {
          this.progressBar = this.progressBar + parseInt(Math.random() * 3 + 2)
          this.progressBar = this.progressBar >= 99 ? 99 : this.progressBar
          this.robotProgress()
        } else {
          clearTimeout(this.robotInter)
          this.robotInter = null
        }
      }, 300)
    },
    /**
     * 盈透选择
     */
    onChangeSelect(val) {
      console.log(val)
      if (val == 1) {
        console.log('短信')
      } else if (val == 2) {
        console.log('ibk')
      } else if (val == 3) {
        console.log('虚拟密码')
      }
    },
    // 关闭弹窗
    handleClose() {
      this.RobotHandStatus = false
      this.$emit('upDataConnect')
      this.clearAllTimer()
      setTimeout(() => {
        this.stepActive = 0
        this.isPWD = false
        this.pwdDefault = ['', '', '', '']
        this.verificationCodeImag = ''
        this.QRcodeUrl = ''
        this.VerificationCode = ''
        this.cellDefault = ['', '', '', '', '', '']
        this.IBKDefault = ['', '', '', '', '', '', '', '']
        ;(this.IBkey = ''), (this.resetCode = '')
        this.cellErrorText = ''
        this.errorTitle = ''
        this.isSub = ''
        this.netInfo = ''
        this.pwdContent = ''
        this.flag = false
        this.progressBar = 0
        this.bankType = 0
        this.count = 12
        this.stepList = []
        this.verificationCodeError = false
        this.bodyLoading = false
        this.IBKRPhoneCode = null
        this.IBKRVisible = true
        this.futuNeedCode = 0
        this.futuInfo.passTime = 3
        this.NextBtnStatus = 0
        this.IBKR = {
          IBKRNums: 60,
          IBKRCount: 0
        }
      }, 200)
    },
    // 机器手  启动链接
    contentLink(parmas, frequency) {
      if (_.isEmpty(parmas)) return
      robotHandContent(parmas)
        .then((res) => {
          console.log('链接机器手的状态', res, frequency)
          // 瑞银第一次连接之后会再次连接获取扫码状态，60s前返回则清除定时器
          if (frequency) {
            this.clearAllTimer()
            clearTimeout(this.RYTimer)
            this.RYTimer = null
          }
          this.bodyLoading = false
          let {
            botEventId,
            verificationCode,
            requiredServerEventId,
            verifyTypes
          } = res.data
          let bankType = this.bankData.bankType // CMBWingLung 招商 UobKayHian 大华 InteractiveBrokers 盈透
          // this.futuNeedCode = botEventId == 1 ? 0 : 1;
          switch (botEventId) {
            case -2: //连接机器手失败
              if (bankType == 'UBS' && this.count < 12 && this.count > 0) {
                return
              }
              this.stepActive = 6
              break
            case -1: //账户为连接状态
              // debugger
              this.stepActive = 5
              this.closeDialog()
              this.upDataTop()
              break
            case 0: //验证码获取成功
              if (bankType == 'CMBWingLung') {
                // 招商
                if (verificationCode != undefined) {
                  this.verificationCodeImag =
                    'data:image/png;base64,' + verificationCode
                  // this.stepActive = "CMB-1";
                  this.stepActive = this.isPWD ? 'CMB-1' : 'pwd'
                }
              } else if (bankType == 'InteractiveBrokers') {
                this.stepActive = 'IBKR-1'
                this.IBKRVisible = false
                this.IBKR.IBKRNums = 60
                this.IBKRInterval()
              } else if (bankType == 'UBS') {
                // 瑞银
                if (frequency) {
                  this.stepActive = 2
                  // if (requiredServerEventId == 2) {
                  //   this.stepActive = 5;
                  // }
                } else {
                  this.stepActive = 'UBS'
                  this.QRcodeUrl = 'data:image/png;base64,' + verificationCode
                  this.UBSInterval()
                }
              } else if (bankType == 'CMSInternational') {
                this.errorTitle = ''
                this.stepActive = 6
              }
              break
            case 1: //登录成功后
              if (this.isPWD || bankType == 'UBS') {
                let timer = 0
                if (bankType == 'Futu' && this.futuNeedCode === 0) {
                  this.stepActive = 'FUTU_NO'
                  // this.stepActive = "pwd"
                  let timer_info = null
                  timer = 3000
                  this.futuInfo.passTime = 3
                  this.futuPassInterval()
                  this.successTimer = setTimeout(() => {
                    if (this.successTimer) {
                      clearTimeout(this.successTimer)
                      this.successTimer = null
                    }
                    this.futuInfo.passTime = 3
                    this.futuPassInterval()
                    this.stepActive = 'FUTU1'
                    timer_info = setTimeout(() => {
                      if (timer_info) {
                        clearTimeout(timer_info)
                        timer_info = null
                      }
                      this.stepActive = 5
                      this.upDataTop()
                      // 清空定时器
                      this.closeDialog()
                      clearTimeout(this.robotInter)
                      this.robotInter = null
                    }, timer)
                  }, timer)
                } else if (bankType == 'Futu' && this.futuNeedCode == 1) {
                  this.stepActive = '3'
                  this.robotProgress()
                  timer = 3000
                  this.successTimer = setTimeout(() => {
                    this.stepActive = 5
                    this.upDataTop()
                    // 清空定时器
                    clearTimeout(this.robotInter)
                    this.closeDialog()
                  }, timer)
                }
                if (bankType == 'UobKayHian' || bankType == 'UBS') {
                  this.stepActive = '3'
                  // this.stepActive = 'connecting';
                  this.robotProgress()
                  timer = 3000
                  this.successTimer = setTimeout(() => {
                    this.stepActive = 5
                    this.upDataTop()
                    // 清空定时器
                    clearTimeout(this.robotInter)
                    this.closeDialog()
                  }, timer)
                }
                if (
                  bankType == 'CMBWingLung' ||
                  bankType == 'CMSInternational' ||
                  bankType == 'InteractiveBrokers' ||
                  bankType == 'XinHongJi'
                ) {
                  this.stepActive = '3'
                  this.robotProgress()
                  setTimeout(() => {
                    this.stepActive = 5
                    this.closeDialog()
                  }, 1000)
                }
              } else {
                this.stepActive = 'pwd'
              }
              break
            case 2: // 校验账户密码成功
              if (bankType == 'InteractiveBrokers') {
                // this.stepActive = "IBKR-1";
                this.stepActive = this.isPWD ? 'IBKR-1' : 'pwd'
                if (verifyTypes && verifyTypes.length == 1) {
                  this.selectValid = verifyTypes[0]
                  if (this.selectValid == 3) {
                    this.stepActive = 'IBK'
                    this.IBkey = verificationCode
                  }
                } else if (verifyTypes && verifyTypes.length > 1) {
                  this.selectValid = 'selects'
                }
                if (this.stepActive == 'IBKR-1') {
                  this.IBKR.IBKRNums = 60
                  this.IBKRVisible = false
                  this.IBKRInterval()
                }
              }
              if (bankType === 'Futu' && requiredServerEventId === 2) {
                if (!verificationCode) {
                  this.stepActive = 'FUTU_NO'
                  this.futuInfo.passTime = 3
                  this.futuPassInterval()
                  this.successTimer = setTimeout(() => {
                    if (this.successTimer) {
                      clearTimeout(this.successTimer)
                      this.successTimer = null
                    }
                    this.stepActive = 'FUTU1'
                    this.futuInfo.countdown = 60
                    this.futuInfo.countdownText = '01:00'
                    this.futuNeedCode = 1
                    this.canPost = false
                    this.futuInterval()
                  }, 3000)
                  // this.stepList = []
                  // this.$nextTick(() => {
                  //   this.stepList = stepsMenu.FUTU
                  // })
                } else {
                  // this.stepList = []
                  // this.$nextTick(() => {
                  //   this.stepList = stepsMenu.HaveValid
                  // })
                  this.verificationCodeImag =
                    'data:image/png;base64,' + verificationCode
                  this.stepActive = this.isPWD ? 'CMB-1' : 'pwd'
                  this.futuNeedCode = 0
                }
              }
              if (bankType === 'UobKayHian' && requiredServerEventId == 3) {
                this.stepActive = 'UobKayHianOTP'
              }

              if (bankType == 'CMSInternational') {
                this.stepActive = 'CMB-3'
                if (this.isAgainOtp) {
                  this.isAgainOtp = false
                  this.$message.success('OTP发送成功')
                }
              }

              if (bankType == 'XinHongJi') {
                this.stepActive = 'XinHongJi'
              }
              break
            case 3: //校验验证码成功
              if (this.flag) {
                this.stepActive = 'CMB-1'
                return
              }
              if (bankType == 'CMBWingLung') {
                this.stepActive = 'CMB-2'
              }
              break
            case 4: //校验otp成功
              if (bankType == 'CMBWingLung') {
                this.stepActive = 5
                this.upDataTop()
                // 清空定时器
                clearTimeout(this.robotInter)
                this.closeDialog()
                this.robotInter = null
              }
              break
            case 5: //校验账户密码错误  登录密码错误时的状态
              this.stepActive = 4
              break
            case 6: //校验验证码错误
              if (verificationCode) {
                // 刷新验证码操作
                this.verificationCodeImag =
                  'data:image/png;base64,' + verificationCode
              }
              if (bankType == 'CMBWingLung') {
                // this.verificationCodeError = true;
                this.verificationCodeError = this.flag ? false : true
                // this.CMBUpLoadVerCode();
              } else if (bankType == 'InteractiveBrokers') {
                this.stepActive = 6
                this.errorTitle = 'IB Key错误，请重新连接机器手'
              } else if (bankType == 'Futu') {
                this.verificationCodeError = this.flag ? false : true

                if (!verificationCode) {
                  this.errorTitle = '短信验证码错误,请重新连接机器手'
                  this.stepActive = 6
                }
                // this.$message.error('验证码错误，请重新输入')
              }
              break
            case 7: //检验验证码超时
              if (bankType == 'InteractiveBrokers') {
                if (this.IBKR.IBKRCount >= 2) {
                  this.stepActive = 6
                }
                this.stepActive = 'IBKR-2'
                this.IBKR.IBKRNums = 60
              } else if (bankType == 'Futu') {
                this.$message.error('短信验证码超时，请重新获取')
              } else {
                this.stepActive = 6
              }
              break
            case 8: //校验otp错误
              // if (bankType == "CMBWingLung" || bankType == "UobKayHian") {
              // this.stepActive = bankType == "CMBWingLung" ? "CMB-2" : 'UobKayHianOTP';
              if (bankType == 'CMBWingLung' || bankType == 'CMSInternational') {
                this.stepActive =
                  bankType == 'CMSInternational' ? 'CMB-3' : 'CMB-2'
              } else if (bankType == 'XinHongJi') {
                this.stepActive = 'XinHongJi'
              } else {
                this.stepActive = 'UobKayHianOTP'
              }
              this.cellErrorText =
                bankType == 'XinHongJi'
                  ? '保安编码输入有误,请重新输入'
                  : 'OTP输入有误,请重新输入'
              this.resetOTP()
              break
            case 9: //校验otp超时
              // if (bankType == "CMBWingLung"|| bankType == "UobKayHian") {
              // this.stepActive =bankType == "CMBWingLung"? "CMB-2":'UobKayHianOTP'
              if (bankType == 'CMBWingLung' || bankType == 'CMSInternational') {
                this.stepActive =
                  bankType == 'CMSInternational' ? 'CMB-3' : 'CMB-2'
              } else {
                this.stepActive = 'UobKayHianOTP'
              }
              this.cellErrorText = 'OTP超时,请重新获取'
              this.resetOTP()

              // }
              break
            case 10: // 交易网关链接失败
              this.errorTitle = '连接盈透网关失败'
              this.netInfo = res.data.errorMsg
                ? res.data.errorMsg
                : '请连接盈透客户端后,再次启动机器手连接'
              this.stepActive = 6
              break
            case 11: // 当前证券业务账户未同意免责协议
              this.cellErrorText = res.data.errorMsg
              if (this.cellErrorText) {
                let idx = this.cellErrorText.indexOf('https')
                this.cellErrorText = this.cellErrorText.substr(0, idx)
                this.isSub = this.cellErrorText(idx)
              }
              this.stepActive = 4
              break
            default:
              this.stepActive = 6
              break
          }
        })
        .catch((error) => {
          this.bodyLoading = false
          this.stepActive = 6
        })
    },
    CMBUpLoadVerCode() {
      //刷新图像
      this.flag = true
      let params = {
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requiredServerEventId: 2, // 2发送验证码
        param: '1',
        requestType: 4
      }
      if (this.bankData.bankType != 'Futu') {
        delete params.requestType
      }
      //重置验证码
      this.VerificationCode = ''
      this.verificationCodeError = false
      this.bodyLoading = true
      this.contentLink(params)
    },
    //链接完成更新状态
    upDataTop() {
      this.$emit('upDateRobot')
      this.$emit('upDataConnect')
    },
    clearVer() {
      //清空验证码error
      this.verificationCodeError = false
    },
    resetOTP() {
      //清空OTP验证码
      this.isShowError = true
      this.cellDefault = ['', '', '', '', '', '']
      this.IBKDefault = ['', '', '', '', '', '', '', '']
      this.resetCode = 'clear'
      this.resetTimer = null
      this.resetTimer = setTimeout(() => {
        clearTimeout(this.resetTimer)
        this.resetCode = ''
      }, 200)
    },
    closeDialog() {
      this.$emit('upDateRobot')
      this.$emit('upDataConnect')
      this.closeTimer = null
      this.closeTimer = setTimeout(() => {
        clearTimeout(this.closeTimer)
        this.handleClose()
      }, 3000)
    },
    phoneVer(phType) {
      let parmas = {
        requiredServerEventId: 4,
        tradingAccountId: Number(this.bankData.tradingAccountId),
        requestType: phType
      }
      this.bodyLoading = true
      this.contentLink(parmas)
    },
    // 盈透倒计时
    IBKRInterval() {
      if (this.IBKRVisible) return
      this.IBKRNumsTimer = setTimeout(() => {
        clearTimeout(this.IBKRNumsTimer)
        --this.IBKR.IBKRNums
        if (this.IBKR.IBKRNums < 0) {
          ++this.IBKR.IBKRCount
          this.stepActive = 'IBKR-2'
          this.IBKRNumsTimer = null
          this.IBKR.IBKRNums = 60
          if (this.IBKR.IBKRCount >= 2) {
            this.stepActive = 6
          }
        } else {
          this.IBKRInterval()
        }
      }, 1000)
    },
    /**
     * 瑞银机器手倒计时
     */
    UBSInterval() {
      clearInterval(this.timerUBS)
      this.timerUBS = null
      this.timerUBS = setInterval(() => {
        this.count--
        this.contentLink(
          {
            tradingAccountId: Number(this.bankData.tradingAccountId),
            requiredServerEventId: 2
          },
          1
        )
      }, 5000)
      this.RYTimer = setTimeout(() => {
        this.stepActive = 6
        this.errorTitle = '二维码已过期，请重新连接'
      }, 1000 * 60)
    },
    // 富途倒计时
    futuInterval(type) {
      // type 1. 点击重新发送按钮
      console.log('进来了吗')
      if (type) {
        if (this.futuInfo.countdown <= 0) {
          // 倒计时结束，重置数据
          this.futuInfo.countdown = 60
          // this.futuInfo.countdown = 30
          this.futuInfo.countdownText === '01:00'
        } else if (this.futuInfo.countdownText === 'again') {
          if (!this.canPost) return
          this.canPost = false
          this.contentLink(
            {
              tradingAccountId: Number(this.bankData.tradingAccountId),
              requiredServerEventId: 1,
              requestType: 4
            },
            1
          )
          return
        } else {
          return
        }
      }
      this.FutuTimer = setTimeout(() => {
        clearTimeout(this.FutuTimer)
        --this.futuInfo.countdown
        let mm = Math.floor(this.futuInfo.countdown / 60)
        let ss = Math.floor(this.futuInfo.countdown % 60)
        this.futuInfo.countdownText =
          (mm < 10 ? '0' + mm : mm) + ':' + (ss < 10 ? '0' + ss : ss)
        if (this.futuInfo.countdown <= 0) {
          this.futuInfo.countdownText = 'again'
          this.canPost = true
          // TODO: 60
          this.futuInfo.countdown = 60
          // this.futuInfo.countdown = 30
          clearTimeout(this.FutuTimer)
          return
        } else {
          this.futuInterval()
        }
      }, 1000)
    },
    // 3秒倒计时
    futuPassInterval() {
      if (this.countdown3s) {
        clearTimeout(this.countdown3s)
        this.countdown3s = null
      }
      this.countdown3s = setTimeout(() => {
        --this.futuInfo.passTime
        if (this.futuInfo.passTime <= 0) {
          return
        }
        this.futuPassInterval()
      }, 1000)
    },
    clearAllTimer() {
      clearTimeout(this.resetTimer)
      this.resetTimer = null
      clearTimeout(this.closeTimer)
      this.closeTimer = null
      clearTimeout(this.IBKRNumsTimer)
      this.IBKRNumsTimer = null
      clearTimeout(this.RYTimer)
      this.RYTimer = null
      clearTimeout(this.FutuTimer)
      this.FutuTimer = null
      clearTimeout(this.countdown3s)
      this.countdown3s = null
      clearTimeout(this.successTimer)
      this.successTimer = null
      clearInterval(this.timerUBS)
      this.timerUBS = null
    }
  },
  beforeUnmount() {
    this.clearAllTimer()
  }
}
</script>
<style lang="scss" scoped>
.contentBody {
  .stepsbox {
    // margin-top: 4px;
    height: 70px;
    display: flex;
    align-items: center;
    justify-content: space-around;
    padding: 0 20px;

    ::v-deep .el-steps {
      width: 100%;

      .el-step__main {
        .is-success {
          color: #868a8f;
        }
      }

      .is-success {
        .el-step__line {
          background: #f38700 !important;
        }
      }

      .is-process {
        .el-step__icon-inner {
          font-weight: 600 !important;
        }
      }
    }
  }

  .isComponent {
    // height: 154px;
    @include fc_h_v;

    .isIcon {
      width: 16px;
      height: 16px;
      line-height: 16px;
      text-align: center;
      margin-right: 6px;

      svg {
        width: 16px;
        height: 16px;
        line-height: 16px;
      }

      // border-radius: 50%;
    }
  }

  isCom {
    @include fc_h_v;

    .isIcon {
      width: 16px;
      height: 16px;
      line-height: 16px;
      text-align: center;
      margin-right: 6px;

      svg {
        width: 16px;
        height: 16px;
        line-height: 16px;
      }

      // border-radius: 50%;
    }
  }

  .step0 {
    display: flex;
    justify-content: center;
    padding-top: 52px;

    .linking {
      @include fz14_v;
      @include fc_h_v;
    }
  }

  .el_input_change {
    ::v-deep .el-input__inner {
      font-size: 14px !important;

      &::-webkit-input-placeholder {
        font-size: 14px !important;
      }
    }
  }

  .step1 {
    &.IBKR .tops .forms,
    &.futu-class .tops .forms {
      width: 400px;
      margin: 0 auto;

      & > span {
        display: inline-block;
        // width: 68px;
        vertical-align: middle;
      }

      .isbody {
        display: flex;
        width: 320px;

        .el-input {
          flex: 1;
          width: 280px;
        }

        .v-btn-text {
          width: 60px;
          text-align: center;
          line-height: 32px;
        }
      }

      .form-text {
        width: 100%;
        text-align: center;
      }
    }

    &.futu-class .tops .forms .isbody {
      width: 340px;
    }

    .tops {
      display: flex;
      margin-top: 20px;

      // height: 102px;
      .forms {
        display: flex;
        align-items: center;
        width: 298px;

        &.formErr {
          ::v-deep .el-input__inner {
            @include bderr_lb_v;
          }
        }

        .isbody {
          position: relative;

          .errorText {
            position: absolute;
            bottom: calc(-18px);
            @include fz12_v;
            color: $derr_lb_v;
          }

          .v-btn-text {
            @include fz12_v;
          }
        }

        & > span {
          display: inline-block;
          width: 108px;
          text-align: right;
        }

        & > div {
          width: calc(100% - 114px);
        }
      }

      .imgbox {
        width: 186px;
        display: flex;
        align-items: center;

        img {
          width: 80px;
          height: 32px;
          margin-right: 8px;
        }

        span {
          @include fz12_v;
          @include fc_m_v;
          cursor: pointer;
        }
      }

      .futu-forms-1 {
        align-items: flex-start;
        // padding-top: 52px;
      }
    }

    .bottom {
      text-align: right;
      padding-top: 22px;
      padding-right: 20px;
      height: 54px;
    }
  }

  .step2 {
    .cell-box {
      display: flex;
      justify-content: center;
      align-items: center;
      padding-top: 20px;

      & > span {
        @include fc_h_v;
        @include fz14_v;
      }
    }

    .error {
      color: #ef5345;
      @include fz12_v;
      padding-left: 131px;
      margin-top: 2px;
    }
  }

  .step3 {
    // position: relative;
    height: 88px;

    .progressbox {
      width: 100%;
      text-align: center;
      margin-top: 38px;
      // padding-top: 50px;
      @include fc_h_v;

      & > span {
        @include fc_m_v;
      }
    }

    .progressBar {
      width: 100%;
      position: absolute;
      bottom: -1px;

      ::v-deep .el-progress-bar__outer {
        @include bg_dia_ck_v;
      }
    }
  }

  .ibk {
    .cell-box {
      display: flex;
      justify-content: center;
      align-items: center;
      padding-top: 16px;

      & > span {
        @include fc_h_v;
        @include fz14_v;
      }
    }

    .error {
      color: #ef5345;
      @include fz12_v;
      padding-left: 60px;
      margin-top: 4px;
    }
  }

  .psdError {
    // height: 210px;
    padding-top: 60px;

    .top {
      display: flex;
      justify-content: center;
      align-items: center;
      margin-bottom: 16px;

      .isIcon {
        // background: linear-gradient(143deg, #fc8a92 0%, #f57780 100%);
        @include fc_fff_v;
      }

      p {
        @include fz16_v;
        line-height: 22px;
        @include fc_h_v;
      }
    }

    .middle {
      @include fz12_v;
      line-height: 20px;

      p {
        text-align: center;
        @include fc_c_v;
        height: 20px;
      }
    }

    .bottom {
      margin-top: 40px;
      text-align: right;
      padding-right: 20px;
    }
  }

  .successLink {
    // height: 210px;
    padding-top: 60px;

    & > div {
      display: flex;
      justify-content: center;
      align-items: center;
    }

    .isIcon {
      // background: linear-gradient(143deg, #83d594 0%, #6ecb80 100%);
      @include fc_fff_v;
    }

    p {
      @include fz16_v;
      @include fc_h_v;
    }
  }

  .QRcode {
    // height:180px;
    margin-top: 22px;
    // margin-bottom: 38px;
    text-align: center;

    p {
      font-size: 14px;
      @include fc_a_v;
    }

    img {
      width: 159px;
      height: 157px;
      margin-top: 20px;
      margin-bottom: 20px;
    }
  }

  .second-time {
    width: auto !important;
    @include fc_m_v;
  }
}

.choose-method {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.pd_tp {
  padding-top: 50px !important;
}

::v-deep .el-dialog {
  min-height: 254px;
  position: relative;

  .el-dialog__header {
    padding: 20px 20px 16px 20px;

    .el-dialog__title {
      font-weight: 600;
      line-height: 22px;
    }

    .el-dialog__headerbtn {
      right: 20px;
    }
  }
}

.bottom1 {
  margin-top: 16px;
  text-align: right;
  padding-right: 20px;
  margin-bottom: 20px;
}

.noBtm {
  margin-top: 0;
}

.no_pd {
  padding: 0 !important;
  // ::v-deep .el-steps{
  //   >div{
  //      &:nth-child(1){
  //       padding-right: 20px;
  //       .el-step__line{
  //         right: calc(-50% + -20px);
  //       }
  //      }
  //      &:nth-child(2){
  //        .el-step__line{
  //         right: calc(-50% + -4px);
  //       }
  //      }
  //      &:nth-child(3){
  //       padding-left: 25px;
  //      }
  //   }
  // }
}

.hasClick {
  color: $wfc_a_v !important;
}

.pd_lt_56 {
  padding-left: 56px;
}

.get_vlid_info_satus {
  padding-left: 12.5px;
  font-size: 12px;
  color: #868a8f !important;
  cursor: pointer;
  user-select: none;
}

.active_class {
  color: #f38700 !important;
}

.mg_bm_12 {
  margin-bottom: 12px;
}
</style>
