<template>
  <div>
    <div class="member custom-scroll">
      <div class="inner">
        <div class="user-profile">
          <user-profile :key="profileKey"/>

          <el-row class="user-opt" :gutter="20">
            <el-col :span="12">
              <el-button type="primary" @click="showBindEmailDialog = true">绑定邮箱</el-button>
            </el-col>
            <el-col :span="12">
              <el-button type="primary" @click="showBindMobileDialog = true">绑定手机</el-button>
            </el-col>
            <el-col :span="12">
              <el-button type="primary" @click="showThirdLoginDialog = true">第三方登录</el-button>
            </el-col>
            <el-col :span="12">
              <el-button type="primary" @click="showPasswordDialog = true">修改密码</el-button>
            </el-col>
            <el-col :span="24">
              <el-button type="primary" @click="showRedeemVerifyDialog = true">卡密兑换
              </el-button>
            </el-col>

            <el-col :span="24" style="padding-top: 30px" v-if="isLogin">
              <el-button type="danger" round @click="logout">退出登录</el-button>
            </el-col>
          </el-row>
        </div>

        <div class="product-box">
          <div class="info" v-if="orderPayInfoText !== ''">
            <el-alert type="success" show-icon :closable="false" effect="dark">
              <strong>说明:</strong> {{ vipInfoText }}
            </el-alert>
          </div>

          <ItemList :items="list" v-if="list.length > 0" :gap="15" :width="240">
            <template #default="scope">
              <div class="product-item">
                <div class="image-container">
                  <el-image :src="vipImg" fit="cover"/>
                </div>
                <div class="product-title">
                  <span class="name">{{ scope.item.name }}</span>
                </div>
                <div class="product-info">
                  <div class="info-line">
                    <span class="label">商品原价：</span>
                    <span class="price">￥{{ scope.item.price }}</span>
                  </div>
                  <div class="info-line">
                    <span class="label">促销立减：</span>
                    <span class="price">￥{{ scope.item.discount }}</span>
                  </div>
                  <div class="info-line">
                    <span class="label">有效期：</span>
                    <span class="expire" v-if="scope.item.days > 0">{{ scope.item.days }}天</span>
                    <span class="expire" v-else>长期有效</span>
                  </div>

                  <div class="info-line">
                    <span class="label">算力值：</span>
                    <span class="power" v-if="scope.item.power > 0">{{ scope.item.power }}</span>
                    <span class="power" v-else>{{ vipMonthPower }}</span>
                  </div>

                  <div class="pay-way">
                    <el-button type="primary" @click="alipay(scope.item)" size="small" v-if="payWays['alipay']">
                      <i class="iconfont icon-alipay"></i> 支付宝
                    </el-button>
                    <el-button type="success" @click="huPiPay(scope.item)" size="small" v-if="payWays['hupi']">
                      <span v-if="payWays['hupi']['name'] === 'wechat'"><i
                          class="iconfont icon-wechat-pay"></i> 微信</span>
                      <span v-else><i class="iconfont icon-alipay"></i> 支付宝</span>
                    </el-button>

                    <el-button type="success" @click="PayJs(scope.item)" size="small" v-if="payWays['payjs']">
                      <span><i class="iconfont icon-wechat-pay"></i> 微信</span>
                    </el-button>
                    <el-button type="success" @click="wechatPay(scope.item)" size="small" v-if="payWays['wechat']">
                      <i class="iconfont icon-wechat-pay"></i> 微信
                    </el-button>
                  </div>
                </div>
              </div>
            </template>
          </ItemList>

          <h2 class="headline">消费账单</h2>

          <div class="user-order">
            <user-order v-if="isLogin"/>
          </div>
        </div>
      </div>

      <password-dialog v-if="isLogin" :show="showPasswordDialog" @hide="showPasswordDialog = false"
                       @logout="logout"/>

      <bind-mobile v-if="isLogin" :show="showBindMobileDialog" @hide="showBindMobileDialog = false"/>

      <bind-email v-if="isLogin" :show="showBindEmailDialog" @hide="showBindEmailDialog = false"/>
      <third-login v-if="isLogin" :show="showThirdLoginDialog" @hide="showThirdLoginDialog = false"/>

      <redeem-verify v-if="isLogin" :show="showRedeemVerifyDialog" @hide="redeemCallback"/>

      <el-dialog
          v-model="showPayDialog"
          :close-on-click-modal="false"
          :show-close="true"
          :width="400"
          @close="closeOrder"
          :title="'实付金额：￥'+amount">
        <div class="pay-container">
          <div class="count-down">
            <count-down :second="orderTimeout" @timeout="refreshPayCode" ref="countDownRef"/>
          </div>

          <div class="pay-qrcode" v-loading="loading">
            <el-image :src="qrcode"/>
          </div>

          <div class="tip success" v-if="text !== ''">
            <el-icon>
              <SuccessFilled/>
            </el-icon>
            <span class="text">{{ text }}</span>
          </div>
          <div class="tip" v-else>
            <el-icon>
              <InfoFilled/>
            </el-icon>
            <span class="text">请打开手机{{ payName }}扫码支付</span>
          </div>
        </div>

      </el-dialog>
    </div>

  </div>
</template>

<script setup>
import {onMounted, ref} from "vue"
import {ElMessage} from "element-plus";
import {httpGet, httpPost} from "@/utils/http";
import ItemList from "@/components/ItemList.vue";
import {InfoFilled, SuccessFilled} from "@element-plus/icons-vue";
import {checkSession, getSystemInfo} from "@/store/cache";
import UserProfile from "@/components/UserProfile.vue";
import PasswordDialog from "@/components/PasswordDialog.vue";
import BindMobile from "@/components/BindMobile.vue";
import RedeemVerify from "@/components/RedeemVerify.vue";
import {useRouter} from "vue-router";
import {removeUserToken} from "@/store/session";
import UserOrder from "@/components/UserOrder.vue";
import CountDown from "@/components/CountDown.vue";
import {useSharedStore} from "@/store/sharedata";
import BindEmail from "@/components/BindEmail.vue";
import ThirdLogin from "@/components/ThirdLogin.vue";

const list = ref([])
const showPayDialog = ref(false)
const vipImg = ref("/images/vip.png")
const enableReward = ref(false) // 是否启用众筹功能
const rewardImg = ref('/images/reward.png')
const qrcode = ref("")
const showPasswordDialog = ref(false)
const showBindMobileDialog = ref(false)
const showBindEmailDialog = ref(false)
const showRedeemVerifyDialog = ref(false)
const showThirdLoginDialog = ref(false)
const text = ref("")
const user = ref(null)
const isLogin = ref(false)
const router = useRouter()
const curPayProduct = ref(null)
const activeOrderNo = ref("")
const countDownRef = ref(null)
const orderTimeout = ref(1800)
const loading = ref(true)
const orderPayInfoText = ref("")
const vipMonthPower = ref(0)
const powerPrice = ref(0)

const payWays = ref({})
const amount = ref(0)
const payName = ref("支付宝")
const curPay = ref("alipay") // 当前支付方式
const vipInfoText = ref("")
const store = useSharedStore()
const profileKey = ref(0)


onMounted(() => {
  checkSession().then(_user => {
    user.value = _user
    isLogin.value = true
  }).catch(() => {
    store.setShowLoginDialog(true)
  })

  httpGet("/api/product/list").then((res) => {
    list.value = res.data
  }).catch(e => {
    ElMessage.error("获取产品套餐失败：" + e.message)
  })

  getSystemInfo().then(res => {
    rewardImg.value = res.data['reward_img']
    enableReward.value = res.data['enabled_reward']
    orderPayInfoText.value = res.data['order_pay_info_text']
    if (res.data['order_pay_timeout'] > 0) {
      orderTimeout.value = res.data['order_pay_timeout']
    }
    vipMonthPower.value = res.data['vip_month_power']
    powerPrice.value = res.data['power_price']
    vipInfoText.value = res.data['vip_info_text']
  }).catch(e => {
    ElMessage.error("获取系统配置失败：" + e.message)
  })

  httpGet("/api/payment/payWays").then(res => {
    payWays.value = res.data
  }).catch(e => {
    ElMessage.error("获取支付方式失败：" + e.message)
  })
})

// refresh payment qrcode
const refreshPayCode = () => {
  if (curPay.value === 'alipay') {
    alipay(curPayProduct.value)
  } else if (curPay.value === 'hupi') {
    huPiPay(curPayProduct.value)
  } else if (curPay.value === 'payjs') {
    PayJs(curPayProduct.value)
  }
}

const genPayQrcode = () => {
  loading.value = true
  text.value = ""
  httpPost("/api/payment/qrcode", {
    pay_way: curPay.value,
    product_id: curPayProduct.value.id,
    user_id: user.value.id
  }).then(res => {
    showPayDialog.value = true
    qrcode.value = res.data['image']
    activeOrderNo.value = res.data['order_no']
    queryOrder(activeOrderNo.value)
    loading.value = false
    // 重置计数器
    if (countDownRef.value) {
      countDownRef.value.resetTimer()
    }
  }).catch(e => {
    ElMessage.error("生成支付订单失败：" + e.message)
  })
}

const alipay = (row) => {
  payName.value = "支付宝"
  curPay.value = "alipay"
  amount.value = (row.price - row.discount).toFixed(2)
  if (!isLogin.value) {
    store.setShowLoginDialog(true)
    return
  }

  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()
}

// 虎皮椒支付
const huPiPay = (row) => {
  payName.value = payWays.value["hupi"]["name"] === "wechat" ? '微信' : '支付宝'
  curPay.value = "hupi"
  amount.value = (row.price - row.discount).toFixed(2)
  if (!isLogin.value) {
    store.setShowLoginDialog(true)
    return
  }

  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()
}

// PayJS 支付
const PayJs = (row) => {
  payName.value = '微信'
  curPay.value = "payjs"
  amount.value = (row.price - row.discount).toFixed(2)
  if (!isLogin.value) {
    store.setShowLoginDialog(true)
    return
  }

  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()
}

const wechatPay = (row) => {
  payName.value = '微信'
  curPay.value = "wechat"
  amount.value = (row.price - row.discount).toFixed(2)
  if (!isLogin.value) {
    store.setShowLoginDialog(true)
    return
  }

  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()
}

const queryOrder = (orderNo) => {
  httpGet("/api/order/query", {order_no: orderNo}).then(res => {
    if (res.data.status === 1) {
      text.value = "扫码成功，请在手机上进行支付！"
      queryOrder(orderNo)
    } else if (res.data.status === 2) {
      text.value = "支付成功，正在刷新页面"
      if (curPay.value === "payjs") {
        setTimeout(() => location.reload(), 3000)
      } else {
        setTimeout(() => location.reload(), 500)
      }
    } else {
      // 如果当前订单没有过期，继续等待订单的下一个状态
      if (activeOrderNo.value === orderNo) {
        queryOrder(orderNo)
      }
    }
  }).catch(e => {
    ElMessage.error("查询支付状态失败：" + e.message)
  })
}

const logout = function () {
  httpGet('/api/user/logout').then(() => {
    removeUserToken();
    router.push('/login');
  }).catch(() => {
    ElMessage.error('注销失败！');
  })
}

const closeOrder = () => {
  activeOrderNo.value = ''
}

const redeemCallback = (success) => {
  showRedeemVerifyDialog.value = false
  if (success) {
    profileKey.value += 1
  }
}

</script>

<style lang="stylus">
@import "@/assets/css/custom-scroll.styl"
@import "@/assets/css/member.styl"
</style>
