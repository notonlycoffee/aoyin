<template>
  <div class="order-done-buyer-wrapper" style="visibility:hidden;">
    <van-nav-bar
      :title="headerObj.title"
      left-arrow
      :fixed="headerObj.fixed"
      @click-left="back"
    />
    <div class="content nav-top">
      <van-cell-group title="订单状态">
        <van-cell class="status" :title="orderStatusText.status" :value="orderInfo.m_name|formatPhone" size="large" />
        <van-cell class="tips" :title="orderStatusText.tips" value="买家" size="large" />
      </van-cell-group>
      <van-cell-group title="交易信息">
        <van-cell title="下单时间" :value="orderInfo.created_date" />
        <van-cell @click="copy" :data-clipboard-text="orderInfo.order_no" class="copy" title="订单编号" :value="orderInfo.order_no">
          <van-icon
            slot="right-icon"
            name="ali iconfont iconfuzhi_o-copy"
            style="line-height:inherit;"
          />
        </van-cell>
        <van-cell title="数量" :value="orderInfo.trade_num+' '+orderInfo.symbol" />
        <van-cell title="单价" :value="'￥'+orderInfo.price" />
        <van-cell v-if="orderInfo.pay_usdtrate" title="汇率" :value="orderInfo.pay_usdtrate" />
        <van-cell title="总交易额" :value="'￥'+orderInfo.total_price" />
      </van-cell-group>
      <van-cell-group title="支付信息">
        <!--银行卡-->
        <div v-if="accountInfo.payment_type==3">
          <van-cell v-if="accountInfo.name" title="收款人" :value="accountInfo.name" />
          <van-cell v-if="accountInfo.bank" title="银行卡信息" :value="accountInfo.bank" />
          <van-cell v-if="accountInfo.payment_no" title="银行卡卡号" :value="accountInfo.payment_no" />
        </div>
        <!--微信/支付宝/MPAY-->
        <div v-else>
          <van-cell title="收款方式">
            <template slot="default">
              <span class="custom-title">{{paymentDictionary[accountInfo.payment_type]}}</span>
              <div class="icon-pay">
                <img :src="paymentIconSrc[accountInfo.payment_type]">
              </div>
            </template>
          </van-cell>
          <van-cell v-if="accountInfo.name" @click="copy" :data-clipboard-text="accountInfo.name" class="copy" title="收款人" :value="accountInfo.name">
            <van-icon
              slot="right-icon"
              name="ali iconfont iconfuzhi_o-copy"
              style="line-height:inherit;"
            />
          </van-cell>
          <van-cell @click="copy" :data-clipboard-text="accountInfo.payment_no" class="copy" :title="paymentDictionary[accountInfo.payment_type]+'账号'" :value="accountInfo.payment_no">
            <van-icon
              slot="right-icon"
              name="ali iconfont iconfuzhi_o-copy"
              style="line-height:inherit;"
            />
          </van-cell>
          <van-cell v-if="accountInfo.payment_url" @click="showQrimg" title="收款二维码">
            <template slot="default">
              <div class="icon-pay">
                <img :src="'data:image/png;base64,'+accountInfo.payment_url">
              </div>
            </template>
          </van-cell>
        </div>
        <van-cell @click="copy" :data-clipboard-text="orderInfo.amount_pwd" class="copy" title="付款备注信息" :value="orderInfo.amount_pwd">
          <van-icon
            slot="right-icon"
            name="ali iconfont iconfuzhi_o-copy"
            style="line-height:inherit;"
          />
        </van-cell>
        <p class="cell-text">如您已向卖家转账付款，请务必点击下方"我已付款"按钮,否则有可能造成资金损失。</p>
      </van-cell-group>
      <div class="btn-area">
        <div @click="submit(3,'确定收款并放币')" class="custom-block-btn">
                           确认收款并放币
          <i class="custom-block-btn-bg"><img src="~@/assets/image/img/block-btn-bg@2x.png"></i>
        </div>
      </div>
      <van-cell-group title="支付凭证图片">
        <div v-if="orderInfo.pay_img" class="pay-img-box">
          <img :src="orderInfo.pay_img" >
        </div>
        <p v-else class="cell-text">未上传</p>
      </van-cell-group>
    </div>
    <van-popup v-model="showQrMasker">
      <div class="img-box"><img :src="'data:image/png;base64,'+accountInfo.payment_url"></div>
    </van-popup>
  </div>
</template>

<script>
import ClipboardJS from 'clipboard';
import { NavBar, Cell, CellGroup, Toast, Icon, Popup, Uploader, Dialog } from 'vant';
import * as utils from '@/utils/utils';

export default {
  name: '',
  components: {
    [NavBar.name]: NavBar,
    [Cell.name]: Cell,
    [CellGroup.name]: CellGroup,
    [Toast.name]: Toast,
    [Icon.name]: Icon,
    [Popup.name]: Popup,
    [Uploader.name]: Uploader,
    [Dialog.Component.name]: Dialog.Component
  },
  data() {
    return {
      headerObj: {
        fixed: true,
        title: '订单详情'
      },
      queryParams: {},
      id: '',
      paymentDictionary: {
        '1': '微信',
        '2': '支付宝',
        '3': '银行卡',
        '4': '农信',
        '5': 'USDT交易',
        '6': 'Mpay交易'
      },
      paymentIconSrc: {
        '1': require('@/assets/image/icon/WeChat.png'),
        '2': require('@/assets/image/icon/alipay.png'),
        '3': require('@/assets/image/icon/pay.png'),
        '4': require('@/assets/image/icon/nongxin.png'),
        '5': require('@/assets/image/icon/usdt.png'),
        '6': require('@/assets/image/icon/logo.png')
      },
      orderInfo: '',
      accountInfo: '',
      showQrMasker: false,
      // 图片
      fileList: [],
      havePayImg: '',
      // 循环刷新
      onceLoading: false,
      seI_autoFn: null, // 定时器
      autoFnIndex: 0 // 定时器执行的次数，可判断执行某些东西
    };
  },
  created() {
    this.queryParams = this.$route.query;
    this.id = this.queryParams.id;
  },
  mounted() {
    this.autoFn();
    this.$el.style.visibility = 'visible';
  },
  watch: {},
  computed: {
    orderStatusText() {
      let obj = {
        status: '等待买家付款',
        tips: `买家将在${this.orderInfo.pay_time}内付款`
      };
      return obj;
    }
  },
  filters: {
    formatPhone: (val) => {
      if (val) {
        return utils.hidePart(val, 3, 4, 4);
      }
    }
  },
  methods: {
    // 实时刷新
    autoFn() {
      let self = this;
      if (self.autoFnIndex < 2) {
        // 你要执行的方法function()
        self.getOrderInfo();
        self.autoFnIndex++;
      } else {
        self.autoFnIndex = 0;
      }
      // 每次清空定时器
      clearTimeout(self.seI_autoFn);
      // 定时器更新时间
      let refushTime = 5 * 1000;
      self.seI_autoFn = setTimeout(function() {
        self.autoFn();
      }, refushTime);
    },
    getOrderInfo() {
      this.$api.order.orderInfo({
        id: this.id
      }).then(res => {
        let result = res.data.result;
        this.orderInfo = result;
        this.accountInfo = this.orderInfo.paymentVO;
      }).catch(error => {
        Toast(this.$t(`lang.system.${error.data.msg}`));
      });
    },
    submit(changeStatus, msg) {
      Dialog.confirm({
        title: '提示',
        message: msg + '?'
      }).then(() => {
        this.$api.order.confirmOrder({
          id: this.id,
          receive_status: changeStatus
        }).then(res => {
          // $evt.trigger('refreshOrder',true);
          Toast('提交成功');
          this.back();
          // $ui.openWinFrame('','~/home/order',{noTitle:true});
        });
      }).catch((error) => {
        console.log(error);
      });
    },
    uploadImg(file) {
      let size = (file.file.size / 1024 / 1024);
      if (size > 2) {
        Toast('请上传 小于2M的图片');
        return;
      }
      this.$api.common.uploadImage({
        file: file.content
      }).then(res => {
        let data = res.data;
        this.havePayImg = this.imgBaseUrl + data.result;
        this.saveImg(this.havePayImg);
      }).catch(error => {
        Toast(this.$t(`lang.system.${error.data.msg}`));
      });
    },
    saveImg(imgUrl) {
      this.$api.common.saveImage({
        id: this.id,
        pay_img: imgUrl
      }).then(res => {
      }).catch(error => {
        Toast(this.$t(`lang.system.${error.data.msg}`));
      });
    },
    showQrimg() {
      this.showQrMasker = true;
    },
    // 复制充值地址
    copy: function() {
      let clipboard = new ClipboardJS('.copy')
        .on('success', e => {
          Toast(e.text + '复制成功');
          e.clearSelection();
          clipboard.destroy();
        });
    },
    back() {
      this.$router.goBack();
    }
  },
  destroyed() {}
};
</script>

<style lang="less" scoped>
.content{
  margin-bottom: 30px;
}
.status.van-cell{
  .van-cell__value{
    color: #fff !important;
  }
  .van-cell__title{
    font-size: 18px;
    font-weight: bold;
    color: @main-color;
  }
}
.tips.van-cell{
  padding: 0 16px 15px;
  color: #fff !important;
  font-size: 12px;
  .van-cell__title{
    flex: 5;
    font-size: 12px;
  }
  .van-cell__value{
    color: #fff !important;
  }
}
.cell-text{
  padding: 20px 16px;
  background: #131F25;
  color: #fff;
  font-size: 12px;
  line-height: 1.5;
}
.icon-pay{
  display: inline-block;
  width: 25px;
  height: 25px;
  margin-left: 5px;
  vertical-align: middle;
}
/*图片上传*/
.img-uploader-box.van-cell-group{
  padding: 10px 16px;
  margin-bottom: 30px;
  background-color: #131F25;
  .tips{
    margin-bottom: 10px;
    font-size: 12px;
    color: #999;
  }
  .van-uploader{
    width: 100%;
  }
}
.img-uploader-box div[class*="id-card"]{
  width: 100%;
  min-height: 220px;
  text-align: center;
  border: 1px solid transparent;
}
.img-uploader-box .img-area .aui-btn{
  width: 40%;
  height: 45px;
  line-height: 45px;
  margin: 25% auto auto;
  color: #096183;
  border: 1px solid;
  border-radius: 4px;
}
.id-card1{
  background: url("~@/assets/image/img/upload_img.png") no-repeat center center;
  background-size: 100%;
}
/*end*/
/*按钮*/
.btn-area{
  margin: 30px auto;
}
/*展示图片*/
.pay-img-box{
  padding: 10px 16px;
}
</style>

<style lang="less">
.order-done-buyer-wrapper{
  background: #060d13;
  /*分组cell*/
  .van-cell-group{
    background-color: transparent;
  }
  .van-cell-group__title{
    padding: 16px;
  }
  .van-cell{
    background-color: #131F25 !important;
    color: @sub-color !important;
    font-size: 15px;
    &::after{
      border-bottom: 1px solid transparent !important;
    }
  }
  /*cell 插槽图标*/
  .van-icon{
    margin-left: 5px;
    font-size: 16px;
    color: @main-color;
  }
  /*上传图片*/
  .img-uploader-box.van-cell-group{
    .van-uploader__input-wrapper{
      width: 100%;
    }
  }
}
</style>
