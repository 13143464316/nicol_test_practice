<template>
  <div class="bank-account">
    <template v-if="isShow">
      <section v-if="!accountNulls" class="bank-content">
        <div class="bank-account-info" v-if="checkedCitiesObj">

          <div class="info-title clearfix">
            <span class="info-title-t mr10">总资产概览</span>
            <span class="info-title-t1">数据按当日8:00汇率进行换算，仅供参考</span>
            <router-link :to="`/bank-manage/add?bankSource=${isOfflineBank ? 2 : 1}`" v-if="$VeriPer('bankAccountAdd')">+创建账户</router-link>
          </div>

          <div class="info clearfix" v-if="currencyList[currencyId]">
            <div class="info-one">
              <div class="set-currency">

                <span @click.stop="downCurrency" class="set-btn">总资产净值({{currencyList[currencyId].currencyName}})
                  <i class="el-icon-caret-bottom adjkticon" :class="isCurrency ? 'iconActive' : ''"></i>
                </span>
                <div v-show="isCurrency" class="currency-list" ref="currency" @click.stop>
                  <div class="currency-li link" @click="setDialogVisible">+ 添加币种</div>
                  <div v-for="(t,i) in currencyList" :key="i" class="currency-li" :class="{active : i === currencyId}" @click="setCurrencyId(i)">{{t.currencyName}}</div>
                </div>

                <el-dialog title="添加换算币种" :visible.sync="dialogVisible" width="680px" :before-close="handleClose">
                  <div class="dialog-set-currency">
                    <p class="dialog-set-currency-title">常用</p>
                    <el-checkbox-group v-model="checkedCities" class="dialog-set-currency-box" min="1">
                      <el-checkbox v-for="city in checkedCitiesList" :label="city.currencyType" :key="city.currencyType" size="mini" v-show="cities.indexOf(city.currencyName) !== -1">{{city.currencyName}}</el-checkbox>
                    </el-checkbox-group>
                  </div>
                  <span slot="footer">
                    <el-button class="v-reset" @click="dialogVisible = false">取 消</el-button>
                    <el-button class="v-define" @click="setExc">确 定</el-button>
                  </span>
                </el-dialog>

              </div>
              <div class="info-assets"> {{numRemTailNum(accountData.accountMarketValue * currency)}}</div>
            </div>
            <div class="info-two">
              <div>
                <span>
                  今日盈亏
                  <el-popover popper-class="hot-pop" placement="top" max-width="250" trigger="hover">
                    <p>今日盈亏=（所有持仓股票今日市值-昨日市值）+（今<br />
                      日买入成交额-今日卖出成交额-今日费用）<br />
                      <br />
                      今日盈亏率=今日盈亏÷总资产净值<br />
                      若今日为非交易日，则显示的是上一个交易日盈亏数据
                    </p>
                    <svg-icon icon-class="home-info" class="funds-title-icon" slot="reference" />
                  </el-popover>
                </span>
              </div>
              <div class="info-tb">
                <span class="mr8" :class="colorto(accountData.todayProfit)">{{numberFormat((accountData.todayProfit ||
                0) * currency,2,true)}}</span>
                <span :class="colorto(accountData.todayProfitRate)">{{numberFormat(accountData.todayProfitRate * 100 ||
                0,2,true)}}%</span>
              </div>
            </div>
            <div class="info-sp"></div>
            <ul class="info-three">
              <li v-for="(t,i) in infoList" :key="i" :class="t.isAdaptation? 'is-adaptation' : ''">
                <div>
                  {{t.name}}
                  <el-popover popper-class="hot-pop" placement="top" max-width="250" trigger="hover" v-if="t.icon">
                    <p>{{t.icon}}</p>
                    <svg-icon icon-class="home-info" class="funds-title-icon" slot="reference" />
                  </el-popover>
                </div>
                <div :class="{links:accountData[t.key] * currency>0}" v-if="t.url" @click="toHolding(1,t)">
                  {{t.choice === 4 ? numberFormat(accountData[t.key] * currency,2,t.isNeedPositiveOrNegative) :
                  numRemTailNum(accountData[t.key] * currency,t.isNeedPositiveOrNegative)}}</div>
                <span v-else :class="t.isNeedAddColor ? colorto(accountData[t.key] * currency) : ''">{{t.choice === 4 ?
                numberFormat(accountData[t.key] * currency,2,t.isNeedPositiveOrNegative) :
                numRemTailNum(accountData[t.key] * currency,t.isNeedPositiveOrNegative)}}</span>
                <div v-for="it in t.children" class="adaptation-show">
                  <div>
                    {{it.name}}
                    <el-popover popper-class="hot-pop" placement="top" max-width="250" trigger="hover" v-if="it.icon">
                      <p>{{it.icon}}</p>
                      <svg-icon icon-class="home-info" class="funds-title-icon" slot="reference" />
                    </el-popover>
                  </div>
                  <div :class="{links:accountData[it.key] * currency>0}" v-if="it.url" @click="toHolding(1,it)">
                    {{it.choice === 4 ? numberFormat(accountData[it.key] * currency,2,it.isNeedPositiveOrNegative) :
                    numRemTailNum(accountData[it.key] * currency,it.isNeedPositiveOrNegative)}}</div>
                  <span v-else :class="it.isNeedAddColor ? colorto(accountData[it.key] * currency) : ''">{{it.choice ===
                  4 ? numberFormat(accountData[it.key] * currency,2,it.isNeedPositiveOrNegative) :
                  numRemTailNum(accountData[it.key] * currency,it.isNeedPositiveOrNegative)}}</span>
                </div>
              </li>
            </ul>
          </div>
          <SwiperContainer v-if="accountList && accountList.length" :list="accountList" :currencyOption="currencyOption"
            tradeDayText="今日" />
          <div class="account-table">
            <el-table :data="tables" border :span-method="arraySpanMethod" @sort-change="sortChange">
              <div slot="empty">
                <EmptyPage type="table" />
              </div>
              <el-table-column type="index" label="序号" width="66" align="center" fixed="left">
                <template slot-scope="scope">{{scope.row.index}}</template>
              </el-table-column>
              <el-table-column show-overflow-tooltip label="银行/券商" min-width="82" align="left" fixed="left"
                class-name="linkCell">
                <template slot-scope="scope">
                  <router-link :to="`/bank-manage/info?id=${scope.row.tradingAcctId}&bankSource=${isOfflineBank ? 2 : 1}`" class="links">
                    {{scope.row.bankName}}</router-link>
                </template>
              </el-table-column>
              <el-table-column show-overflow-tooltip label="户名" min-width="90" align="left" fixed="left"
                class-name="linkCell">
                <template slot-scope="scope">
                  <router-link :to="`/bank-manage/info?id=${scope.row.tradingAcctId}&bankSource=${isOfflineBank ? 2 : 1}`" class="links">
                    {{scope.row.accountName}}</router-link>
                </template>
              </el-table-column>
              <el-table-column :label="isOfflineBank ? '账户编号' : '账户号码'" min-width="90" align="left" fixed="left"
                class-name="linkCell">
                <template slot-scope="scope">
                  <el-popover placement="top" trigger="hover" :content="scope.row.accountCode"
                    popper-class="bank-el-popover">
                    <router-link :to="`/bank-manage/info?id=${scope.row.tradingAcctId}&bankSource=${isOfflineBank ? 2 : 1}`" slot="reference" class="links">
                      {{toMi(scope.row.accountCode)}}</router-link>
                  </el-popover>
                </template>
              </el-table-column>
              <el-table-column label="账户类别" min-width="96" fixed="left" align="left">
                <template slot-scope="scope">
                  <span v-if="isOfflineBank">--</span>
                  <div v-else>
                    <svg-icon v-if="[0,1].includes(scope.row.isOpen)&&scope.row.accountType!==0"
                    :icon-class="scope.row.isOpen===1?'st_icon_table_fold':'st_icon_table_open'" class="mr8 pointer"
                    @click.stop="openFold(scope.row)"></svg-icon>
                    <span v-if="[0,1].includes(scope.row.isOpen)">{{ scope.row.accountType }}</span>
                    <template v-else>
                      {{dictFun(accTypeOption,scope.row.accountType || 0)}}{{dictFun(currencyOption,scope.row.accountCashType)}}
                    </template>
                  </div>
                </template>
              </el-table-column>
              <el-table-column label="总资产净值" prop="marketValue" min-width="140"
                :width="flexColumnWidth('marketValueText',tables)>140?flexColumnWidth('marketValueText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span>{{numRemTailNum(scope.row.marketValue* currencyFn(scope.row))}}</span>
                </template>
              </el-table-column>
              <el-table-column label="股票市值" prop="stockMarketValue" min-width="140"
                :width="flexColumnWidth('stockMarketValueText',tables)>140?flexColumnWidth('stockMarketValueText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span :class="{links:scope.row.stockMarketValue>0}"
                    @click="toHolding(2,scope.row)">{{numberFormat(scope.row.stockMarketValue*
                    currencyFn(scope.row),2)}}</span>
                </template>
              </el-table-column>
              <el-table-column label="市值盈亏" prop="marketProfit" min-width="140"
                :width="flexColumnWidth('marketProfitText',tables)>140?flexColumnWidth('marketProfitText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span :class="colorto(scope.row.marketProfit)">{{numberFormat(scope.row.marketProfit*
                  currencyFn(scope.row),2,true)}}</span>
                </template>
              </el-table-column>
              <el-table-column label="系统现金" prop="amount" min-width="140"
                :width="flexColumnWidth('amountText',tables)>140?flexColumnWidth('amountText',tables):140" align="right"
                sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <el-popover popper-class="hot-pop" placement="top" width="220" trigger="hover">
                    <div class="amount-hover">
                      <p>可用资金</p>
                      <p>{{numRemTailNum(scope.row.availableAmount * currencyFn(scope.row))}}</p>
                    </div>
                    <div class="amount-hover">
                      <p>交易占用资金</p>
                      <p>{{numRemTailNum(scope.row.frozenAmount * currencyFn(scope.row))}}</p>
                    </div>
                    <p slot="reference">{{numRemTailNum(scope.row.amount * currencyFn(scope.row))}}</p>
                  </el-popover>
                </template>
              </el-table-column>
              <el-table-column label="银行现金" prop="bankAmount" min-width="140"
                :width="flexColumnWidth('bankAmountText',tables)>140?flexColumnWidth('bankAmountText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span>{{numRemTailNum(scope.row.bankAmount * currencyFn(scope.row))}}</span>
                </template>
              </el-table-column>
              <el-table-column label="初始成本" prop="initialCost" min-width="140"
                :width="flexColumnWidth('initialCostText',tables)>140?flexColumnWidth('initialCostText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span v-if="isOfflineBank">--</span> 
                  <span v-else>{{numRemTailNum(scope.row.initialCost * currencyFn(scope.row))}}</span> 
                </template>
              </el-table-column>
              <el-table-column label="盈亏余额" prop="profitBalance" min-width="140"
                :width="flexColumnWidth('profitBalanceText',tables)>140?flexColumnWidth('profitBalanceText',tables):140"
                align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  <span v-if="isOfflineBank">--</span>
                  <span v-else :class="colorto(scope.row.profitBalance)">{{numberFormat(scope.row.profitBalance*
                  currencyFn(scope.row),2,true)}}</span>
                </template>
              </el-table-column>
              <el-table-column label="费用金额" prop="feeAmount" min-width="100" align="right" sortable="custom"
                :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  {{numberFormat(scope.row.feeAmount* currencyFn(scope.row),2,false)}}
                </template>
              </el-table-column>
              <el-table-column label="债券" prop="debenture" min-width="100" align="right" sortable="custom"
                :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  {{numberFormat(scope.row.debenture* currencyFn(scope.row),2)}}
                </template>
              </el-table-column>
              <el-table-column label="基金" prop="fund" min-width="100" align="right" sortable="custom"
                :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  {{numberFormat(scope.row.fund* currencyFn(scope.row),2)}}
                </template>
              </el-table-column>
              <el-table-column label="定存" prop="fixedDeposit" min-width="100" align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  {{numberFormat(scope.row.fixedDeposit* currencyFn(scope.row),2)}}
                </template>
              </el-table-column>
              <el-table-column label="保单" prop="guarantee" min-width="100" align="right" sortable="custom" :sort-orders="['descending', 'ascending', null]">
                <template slot-scope="scope">
                  {{numberFormat(scope.row.guarantee* currencyFn(scope.row),2)}}
                </template>
              </el-table-column>
              <el-table-column label="机器手状态" min-width="110" align="left">
                <template  slot-scope="scope">
                  <span v-if="isOfflineBank">--</span>
                  <div v-else>
                    <span v-if="scope.row.robotFirst === 1" class="err">未连接</span>
                    <span v-else-if="scope.row.robotState === true">已连接</span>
                    <span v-else-if="scope.row.robotState === false" class="err">已断开</span>
                    <span >
                    <!--@click="robotHandOpen(scope.row)" scope.row.robotState === 0 &&  :value="scope.row.robotState"-->
                    <el-switch  @change="val => robotHandOpen(val,scope.row)" v-if=" $VeriPer('connectRobot')"
                      :value="scope.row.robotState"  class="ml6"></el-switch>
                  </span>
                  </div>
                </template>
              </el-table-column>
              <el-table-column label="账户状态" min-width="100" align="left">
                <template slot="header">
                  账户状态
                  <el-popover popper-class="hot-pop" placement="top" width="250" trigger="hover">
                    <p>
                      交易启用表示该账户可用于证券交易；交易停止表示该账户仅不可在投资管理系统内进行交易，已经挂单的交易会继续等待成交，不影响银行账户使用
                    </p>
                    <svg-icon icon-class="home-info" class="funds-title-icon" slot="reference" />
                  </el-popover>
                </template>
                <div v-if="isOfflineBank" slot-scope="scope">
                  <span>--</span>
                </div>
                <template v-else slot-scope="scope">
                  <span :class="{err : scope.row.accountState}">{{scope.row.accountState === null ? '--' :
                  scope.row.accountState === 0 ? '交易启用' : '交易停止'}}</span>
                </template>
              </el-table-column>
              <el-table-column v-if="!isOfflineBank" label="操作" width="100" align="left" fixed="right" style="padding: 0 10px;">
                <template slot-scope="scope">
                  <div class="tabBtn">
                    <span class="svg-btn"
                      v-if="scope.row.accountState !== null&&scope.row.accountType!==0&&$VeriPer('transactionState')">
                      <el-popover placement="top" trigger="hover" popper-class="bank-el-popover">
                        <p>
                          {{scope.row.accountState === 1 ? '启用交易' : '停止交易'}}
                        </p>
                        <svg-icon :icon-class="scope.row.accountState === 1 ? 'st_icon_start' : 'vv_list_deactivate'"
                          class="bt-hover" slot="reference" @click="setStop(scope.row)" />
                      </el-popover>
                    </span>
                    <span class="svg-btn" v-if="scope.row.robotFirst !== 1">
                      <el-popover placement="top" trigger="hover" popper-class="bank-el-popover">
                        <p>手动爬取</p>
                        <svg-icon class="crawling bt-hover" icon-class="list_crawling_setup" slot="reference"
                          @click="crawleHandle(scope.row.tradingAcctId, scope.row.bankCode)"></svg-icon>
                      </el-popover>
                    </span>
                  </div>
                </template>
              </el-table-column>
            </el-table>
            <Paging ref="page" @pageSize="setPageSize" @Current="currentPage" :isHiden="!total" />
          </div>
        </div>
        <RobotHand :visibleStatus="robotHandStatus"  :bankData="robotHandData" @upDataConnect="upDataConnectInfo" @upDateRobot="init(true)" />
      </section>
      <section v-else>
        <AccountEmpty :is-offline-bank="isOfflineBank"/>
      </section>
    </template>
  </div>
</template>

<script>
import { Loading } from 'element-ui';
import Paging from '@/components/paging/paging';
import EmptyPage from "@/components/empty/empty-page";
import RobotHand from '@/views/bank-account/components/robotHand/index.vue';
import AccountEmpty from '@/views/bank-account/bank-manage/components/accountEmpty';
import { abridge, colorto, numberFormat, dictFun, isTradeDay, sortOfKeys, flexColumnWidth, } from "@/utils/public";
import { getDictList } from "@/api/public";
import { robotDisConnect,getQuery, setEnable, setExchangeRate, getExchangeRateTotal, getExchangeRateList, bankCapital, bankCapitalOpen, checkRobotIsOnline, manualGetCash } from "@/api/bank-account/index";
import SwiperContainer from "@/views/bank-account/components/swiper-container.vue"
import { numRemTail } from '@/utils/validate';
import ret from 'bluebird/js/release/util';
// import { errorCodes } from '@/utils/robot-errorCode';
let requestLoding = null;
export default {
  components: {
    Paging,
    RobotHand,
    EmptyPage,
    AccountEmpty, SwiperContainer
  },
  props:{
    isOfflineBank:{
      type: Boolean,
      default: false
    }
  },
  data() {

    return {
      accw: 0,//页面宽度
      dialogVisible: false,//设置币种
      checkedCities: [],//多选币种数组
      checkedCitiesList: [],//多选币种可选数组
      checkedCitiesObj: null,//多选币种对象
      isConnect: false,// 是否链接
      flag: false,
      isdisConnect:false,// 点击连接机器手的时候需要清空
      cities: [//币种可选列表
        '港元HKD',
        '美元USD',
        '人民币CNY',
        '新加坡元SGD',
        '英镑GBP',
        '欧元EUR',
        '澳元AUD',
        '加元CAD',
        '韩元KRW',
        '日元JPY',
        '澳门元MOP',
        // '埃及镑EGP',
        // '阿根廷比索ARS',
      ],
      accTypeList: {
        0: '港币',
        1: '美元',
        2: '人民币',
        3: '新币',
      },
      uploadShow: false,// 上传弹框显示
      robotHandStatus: 0,//机器手启动
      robotHandData: {},//下发的数据
      showSetCurrency: false,
      isCurrency: false,
      mySwiper: null,
      currencyId: 0,
      currencyList: [],
      infoList: [
        {
          name: '股票市值',
          icon: '',
          key: 'stockMarketValue',
          url: '/market-holding',
          isNeedPositiveOrNegative:false,
          choice:4,
          children: [
            {
              name: '费用金额',
              icon: '',
              key: 'feeAmount',
              url: '',
              isNeedPositiveOrNegative:false,
              choice:4
            }
          ]
        },
        {
          name: '市值盈亏',
          icon: '市值盈亏=∑(现价-持仓均价不含费用)×持有股数 ',
          key: 'marketProfit',
          url: '',
          isNeedPositiveOrNegative:true,
          choice:4,
          isNeedAddColor:true,
          children: [{
            name: '债券',
            icon: '',
            key: 'debenture',
            url: '',
            isNeedPositiveOrNegative:false,
            choice:4
          }]
        },
        {
          name: '系统现金',
          icon: '',
          key: 'amount',
          url: '',
          isNeedPositiveOrNegative: false,
          choice:5,
          children: [{
            name: '基金',
            icon: '',
            key: 'fund',
            url: '',
            isNeedPositiveOrNegative:false,
            choice:4
          }]
        },
        {
          name: '银行现金',
          icon: '',
          key: 'bankAmount',
          url: '',
          isNeedPositiveOrNegative: false,
          choice: 5,
          children: [{
            name: '定存',
            icon: '',
            key: 'fixedDeposit',
            url: '',
            isNeedPositiveOrNegative:false,
            choice:4
          }]
        },
        {
          name: '初始成本',
          icon: '初始成本=初始化成本+转入资金-转出资金+/-其他',
          key: 'initialCost',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:5,
          children: [{
            name: '保单',
            icon: '',
            key: 'guarantee',
            url: '',
            isNeedPositiveOrNegative:false,
            choice:4
          }]
        },
        {
          name: '盈亏余额',
          icon: '盈亏余额=总资产净值-初始成本',
          key: 'profitBalance',
          url: '',
          isNeedPositiveOrNegative:true,
          choice:4,
          isNeedAddColor:true
        },
        {
          name: '费用金额',
          icon: '',
          key: 'feeAmount',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:4,
          isAdaptation: true
        },
        {
          name: '债券',
          icon: '',
          key: 'debenture',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:4,
          isAdaptation: true
        },
        {
          name: '基金',
          icon: '',
          key: 'fund',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:4,
          isAdaptation: true
        },
        {
          name: '定存',
          icon: '',
          key: 'fixedDeposit',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:4,
          isAdaptation: true
        },
        {
          name: '保单',
          icon: '',
          key: 'guarantee',
          url: '',
          isNeedPositiveOrNegative:false,
          choice:4,
          isAdaptation: true
        },
      ],
      accountData: {},
      accountList: [],
      swiperOptions: {
        pagination: {
          el: '.swiper-pagination'
        },
      },
      tables: [],
      searchData: {
        currentPage: 1,
        pageSize: 10,
        sort: undefined,
        sortKey: undefined
      },
      total: 0,
      si: null,
      timestamp: new Date().getTime(),
      currencyOption: [],//币种
      accTypeOption: [],//账户类型
      accountNulls: false,
      isShow: false,
      openIdList: [],
      newTables: [],//缓存最新列表
      isUpdateIndex: 0,//当前便利获取数据的下标
      openLen: 0,//新数据应该展开的数量
    }
  },
  computed: {
    isTradeDayText() {
      return this.isTradeDay() ? "当日" : "今日"
    },
    currency() {
      return this.currencyList[this.currencyId].basicHKD || 1
    }
  },
  methods: {
    colorto,
    abridge,
    numberFormat,
    dictFun, isTradeDay, flexColumnWidth, numRemTail,
    // 向下截断，保留两位小数
    numRemTailNum(v,isNeedPositiveOrNegative = false) {
      let num = this.numRemTail(v, 2)
      return this.numberFormat(Number(num), 2,isNeedPositiveOrNegative)
    },
    currencyFn(row) {
      // 只有汇总的会进行币种计算
      return [0, 1].includes(row.isOpen) ? this.currency : 1
    },
    initDict() {
      // 账户类型
      getDictList("acc_type").then(res => {
        this.accTypeOption = res.data;
      })
      // 币种
      getDictList("currency_type").then(res => {
        this.currencyOption = res.data;
      })
      getExchangeRateTotal().then(data => {
        this.checkedCitiesObj = {}
        data.data.forEach(t => {
          this.checkedCitiesObj[t.currencyType] = t
        })
        getExchangeRateList().then(data => {
          this.currencyList = null
          this.currencyList = data.data
          this.init(true)
        })

      })
    },
    setExc() {
      setExchangeRate(this.checkedCities).then(data => {
        getExchangeRateList().then(data => {
          this.currencyList = null
          this.currencyList = data.data

          this.dialogVisible = false
          this.currencyId = 0
        })
      })
    },
    setDialogVisible() {
      getExchangeRateTotal().then(data => {
        this.checkedCities = null
        this.checkedCities = []
        this.currencyList.forEach(t => {
          this.checkedCities.push(t.currencyType)
        })
        this.checkedCitiesList = null
        this.checkedCitiesList = data.data
        this.dialogVisible = true
      })
    },
    handleClose() {
      this.dialogVisible = false
    },
    toMi(s, mi = true) {
      if (s.length < 4) {
        if (this.isOfflineBank && s.length === 0) { 
          return '--'
        }
        let s1 = '***'
        for (let i = 0; i < 4 - s.length; i++) {
          s1 += '0'
        }
        s1 = mi ? s1 + s : s1
        return s1
      } else {
        return mi ? '***' + s.split('').slice(s.length - 4, s.length).join('') : s.split('').slice(s.length - 4, s.length).join('')
      }
    },
    //打开
    setUploadShow(row) {
      this.uploadShow = true
    },
    //切换账户状态
    setStop(row) {
      // row.accountState 0启用  1禁用
      if (row.accountState == 1) {
        this.changeEnable(row.tradingAcctCashId, 0)
      } else {
        let text = ""
        // 银行账户-汇总
        if ([0, 1].includes(row.isOpen)) {
          text = `停用${this.toMi(row.accountCode, false)}（${row.bankName}）账户，将无法用此账户挂单，已经挂单的交易将继续等待成交`
        } else {
          // 资金账户-展开
          text = `停用${row.accountCashType === null ? '' : this.dictFun(this.accTypeOption, row.accountType || 0) + this.dictFun(this.currencyOption, row.accountCashType)}账户${this.toMi(row.accountCode, false)}（${row.bankName}），将无法用此账户挂单，已经挂单的交易将继续等待成交`
        }
        this.$confirm(text, `停用交易账户`, {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
          cancelButtonClass: "v-reset",
          confirmButtonClass: "v-define"
        }).then(() => {
          this.changeEnable(row.tradingAcctCashId, 1)
        })
      }
    },
    changeEnable(id, status) {
      // status 0启用 1停用
      setEnable({
        ids: id,
        status: status,
      }).then(data => {
        if (status == 0 && data.data.status === 0) {
          this.$message.success('账户启用成功');
        } else if (status == 1 && data.data.status === 0) {
          this.$message.success('账户停用成功');
        }
        this.isUpdateIndex = 0;
        this.getTableData(true)
      })
    },
    robotHandOpen(val, topdata) {
      if (!val) {
        // this.flag = true
        this.$confirm(`断开机器手将导致银行的放单无法及时回单，且无法及时爬取流水和结单，请确定是否要断开机器手?`, '机器手断开', {
          confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'info',
            closeOnClickModal:false,
            cancelButtonClass: "v-reset",
            confirmButtonClass: "v-define"
         }).then(() => {
           let param = {
             tradingAccountId: topdata.tradingAcctId
          }
           robotDisConnect(param).then(res => {
             topdata.robotState = false
             const {data} = res
             if (data && data.status == 0) {
              //TODO: 成功不需要提示
              //  this.$message.success(`${data.message}`)
             } else {
               this.$message.warning(`${data.message}`)
             }
             this.getTableData(true)
              // this.flag = false
            })
           
         }).catch(() => { 
          // this.flag = false
          })
          return
      }
      
      this.robotHandStatus = this.robotHandStatus + 1;
      //传给机器手的数据详情
      this.robotHandData = {
        tradingAccountId: topdata.tradingAcctId,
        bankType: topdata.bankCode,
        phoneNumber: topdata.phoneNumber
      }
        this.isdisConnect = true
      console.log('传过去的参数',this.robotHandData);
    },
    //列表数据
    getTableData(isNoModal) {
      if (!isNoModal) {
        this.total = 0
        this.searchData.currentPage = 1
      }
      bankCapital(Object.assign({bankSource: this.isOfflineBank ? 2 : 1},this.searchData), isNoModal).then(res => {
        let data = res.data
        this.newTables = data.records.map((item, i) => {
          return Object.assign(item, {
            index: i + 1,
            isOpen: 0,
            openList: [],
            stockMarketValueText: this.numberFormat(item.stockMarketValue * this.currencyFn(item), 2),
            marketValueText: this.numberFormat(item.marketValue * this.currencyFn(item), 2),
            marketProfitText: this.numberFormat(item.marketProfit * this.currencyFn(item), 2),
            amountText: this.numberFormat(item.amount * this.currencyFn(item), 2),
            profitBalanceText: this.numberFormat(item.profitBalance * this.currencyFn(item), 2),
            initialCostText: this.numberFormat(item.initialCost * this.currencyFn(item), 2),
            robotState:item.robotState?true:false
          })
        })
        this.searchData.pageSize = data.size
        this.total = data.total
        this.searchData.currentPage = data.current
        if (data.total) {
          this.accountNulls = false;
        } else {
          this.accountNulls = true;
        }
        this.isShow = true;
        this.$nextTick(() => {
          if (this.$refs.page) this.$refs.page.initLoading(this.searchData.pageSize, this.total, this.searchData.currentPage);
        });
        if (requestLoding) requestLoding.close()
        if (!this.openIdList.length) {
          this.tables = [...this.newTables]
        }
        this.openLen = 0;
        this.newTables.find(val => {
          this.openIdList.find(v => {
            if (val.tradingAcctId === v) {
              this.openLen++
            }
          })
        })
        this.isUpdateIndex = this.openIdList.length - this.openLen
        this.newTables.find((item, index) => {
          this.openIdList.find(val => {
            if (item.tradingAcctId == val) {
              this.openFold(item, isNoModal)
            }
          })
        })
      })
        .finally(() => {
          if (this.si) {
            clearTimeout(this.si)
            this.si = null
          }
          // if (this.isdisConnect) {
          //    clearTimeout(this.si)
          //   return;
          // }
          if (this.$route.path === '/bank-manage') {
            this.si = setTimeout(() => {
              this.init(isNoModal)
            }, 5000)
          }
        })
    },
    arraySpanMethod({ row, column, rowIndex, columnIndex }) {
      let indexArr = [0, 1, 2, 3, 17];
      let namelen = row.openList && row.openList.length;
      if (indexArr.includes(columnIndex)) {
        if (row.isOpen === 0) {
          return [1, 1];
        } else if (row.isOpen === 1) {
          return [Number(namelen + 1), 1];
        } else {
          return [0, 0];
        }
      }
    },
    openFold(v, bool) {
      // isOpen  0未展开  1  展开  
      // 展开
      if (v.isOpen === 0) {
        if (!this.openIdList.includes(v.tradingAcctId)) {
          this.openIdList.push(v.tradingAcctId);
        }
        bankCapitalOpen(v.tradingAcctId, bool).then(res => {
          res.data.forEach(t => {t.bankCode = v.bankCode})
          // 点击
          if (!bool) {
            let index = this.tables.findIndex(val => v.tradingAcctId === val.tradingAcctId);
            this.tables[index].openList = res.data;
            this.tables.splice(index + 1, 0, ...this.tables[index].openList);
          } else {
            // 自动刷新
            this.isUpdateIndex++;
            let i = this.newTables.findIndex(val => v.tradingAcctId === val.tradingAcctId);
            this.newTables[i].openList = res.data;
            this.newTables.splice(i + 1, 0, ...this.newTables[i].openList);
            this.newTables[i].isOpen = 1;
            if (this.isUpdateIndex == this.openIdList.length) {
              this.isUpdateIndex = 0
              this.tables = JSON.parse(JSON.stringify(this.newTables));
            }
          }
          v.isOpen = 1
        })
      } else {
        // 收起
        this.openIdList.find((val, i) => {
          if (val === v.tradingAcctId) {
            this.openIdList.splice(i, 1)
          }
        })
        let index = this.tables.findIndex(val => v.tradingAcctId === val.tradingAcctId);
        let i = this.newTables.findIndex(val => v.tradingAcctId === val.tradingAcctId);
        this.tables.splice(index + 1, this.tables[index].openList.length);
        this.tables[index].openList = [];
        v.isOpen = 0;
        this.newTables.splice(i + 1, this.newTables[i].openList.length);
        this.newTables[i].openList = [];
        this.newTables[i].isOpen = 0;
      }
    },
    getData() {
      this.timestamp = new Date().getTime()
      //资产卡片
      getQuery({bankSource: this.isOfflineBank ? 2 : 1}).then(data => {
        this.accountData = { ...data.data }
        this.accountList = null
        const accountList = (data.data && data.data.accounts) || []
        // 线下银行只显示港元账户
        if (this.isOfflineBank) { 
          this.accountList = accountList.filter(item => { 
            return item.currencyType === 0
          })
          return
        }
        this.accountList = accountList
      })
    },

    upDataConnectInfo() {
      this.isdisConnect = false;
      this.init(true);
    },

    init(isNoModal) {
       if (this.isdisConnect) {
         clearTimeout(this.si)
         this.si = null
            return;
          }
      this.isUpdateIndex = isNoModal ? 0 : this.isUpdateIndex;
      this.getTableData(isNoModal);
      this.getData();
    },
    setSwiper(type) {
      if (type) {
        this.mySwiper.slidePrev()
      } else {
        this.mySwiper.slideNext()
      }
    },
    downCurrency() {
      let _this = this
      if (!this.isCurrency) {
        document.body.onclick = () => {
          _this.isCurrency = false
          document.body.onclick = () => { }
        }
      }
      this.isCurrency = !this.isCurrency
    },
    setCurrencyId(i) {
      this.currencyId = i
      this.downCurrency()
    },
    setPageSize(v, p) {
      this.openIdList = [];
      this.searchData.currentPage = 1
      this.searchData.pageSize = v
      this.getTableData(true)
    },
    currentPage(p) {
      this.openIdList = [];
      this.searchData.currentPage = p
      this.getTableData(true)
    },
    sortChange(c) {
      this.openIdList = [];
      sortOfKeys(this, c);
      this.getTableData(true);
    },
    toHolding(type, rows) {
      // type 1概览  2列表
      if ((type == 1 && !this.accountData[rows.key] * this.currency) || (type == 2 && !rows.stockMarketValue)) return
      let params;
      if (type == 1) {
        params = {}
      } else {
        params = {
          tradingAcctId: rows.tradingAcctId,
        }
      }
      this.$router.push({
        name: 'market-holding',
        params: params
      })
    },
    // 手动爬取
    crawleHandle(id, bankCode) {
      checkRobotIsOnline(id).then(res => {
        if (res.data) {
          this.$confirm(`机器手将在空闲时爬取银行流水和银行现金，爬取完成后将消息通知您。`, '爬取流水', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'info',
            cancelButtonClass: "v-reset",
            confirmButtonClass: "v-define"
          }).then((res) => {
            manualGetCash(id, bankCode).then(res => {
              this.$message.success('提交成功，爬取完成后将消息通知您')
            })
          }).catch(() => {})
        } else {
          this.$message.error('机器手已断开，请连接机器手')
        }
      })
    },
    render() { 
      if (this.$VeriPer('bankAccountQuery')) {
        requestLoding = Loading.service({
          text: '加载中...',
          spinner: 'el-icon-loading',
          background: 'rgba(0, 0, 0, 0.5)',
        });
        this.initDict();
      } else {
        clearTimeout(this.si);
        this.si = null;
      }
    },
    destory() { 
      clearTimeout(this.si);
      this.si = null;
    }
  },
  mounted() {
    this.render()
  },
  deactivated() {
    this.destory()
  },
  beforeDestroy() {
    this.destory()
  },
}
</script>

<style lang="scss">
.bank-el-popover {
  min-width: auto;
  text-align: center;
}

.bank-account {
  .el-dialog__header {
    padding: 20px 20px 10px 20px;
  }
  .el-dialog__footer {
    padding: 16px 20px 20px 20px;
  }
  .el-checkbox__label {
    font-size: 12px;
  }
}
</style>

<style lang="scss" scoped>
@mixin to {
  [data-theme="dark"] & {
    color: #6c96f5;
  }
  [data-theme="white"] & {
    color: #2440b3;
  }
}

@mixin err {
  [data-theme="dark"] & {
    color: #fc424e;
  }
  [data-theme="white"] & {
    color: #ef4b45;
  }
}

@mixin btn {
  [data-theme="dark"] & {
    border-radius: 50%;
    color: rgba(255, 255, 255, 0.8);
    background-color: rgba(11, 13, 21, 0.4);
    border: 2px solid rgba(255, 255, 255, 0.15);
  }
  [data-theme="white"] & {
    border-radius: 50%;
    color: #606070;
    background-color: rgba(0, 0, 0, 0.08);
    border: 2px solid transparent;
  }
}

@mixin gradient($fx) {
  [data-theme="dark"] & {
    background-image: linear-gradient(
      to $fx,
      rgba(37, 38, 51, 1),
      rgba(37, 38, 51, 0)
    );
  }
  [data-theme="white"] & {
    background-image: linear-gradient(
      to $fx,
      rgba(255, 255, 255, 1),
      rgba(255, 255, 255, 0.9),
      rgba(255, 255, 255, 0.5),
      rgba(255, 255, 255, 0.3),
      rgba(255, 255, 255, 0)
    );
  }
}
.bank-account {
  height: calc(100vh - 150px);
  font-size: 12px;
  display: flex;
  flex-direction: column;

  .funds-title-icon {
    @include fc_i_i_v;
  }

  .dialog-set-currency {
    height: 300px;
    margin: 0 20px;
    padding: 12px 16px;
    border: 1px solid;
    font-size: 12px;
    @include l_lb_s_v;

    .dialog-set-currency-box {
      height: 245px;
      display: flex;
      flex-direction: column;
      flex-wrap: wrap;
      align-content: flex-start;

      .el-checkbox {
        height: 24px;
        margin: 0 0 9px 0;
        display: inline-block;
        width: 33.33%;
      }
    }

    .dialog-set-currency-title {
      @include fc_h_v;
      margin-bottom: 12px;
    }
  }
  .bank-content {
    height: calc(100vh - 150px);
    background: transparent;
    margin-top: 0;
    overflow: auto;
  }
  .err {
    @include err;
  }

  .bt-hover {
    font-size: 18px;
    cursor: pointer;
    transition: 0.3s;
    @include fc_m_v;
    vertical-align: text-top;
  }

  .svg-icon:hover {
    @include fc_m_v;
  }

  .account-table {
    padding: 16px 0 12px;
    @include bg_t_v;
  }

  .bank-account-info {
    min-height: 100%;
    padding: 0 12px;
    @include bg_t_v;

    .to {
      @include to;
    }

    .info-title {
      padding: 12px 0;

      .info-title-t {
        float: left;
        font-size: 15px;
        line-height: 28px;
        @include fc_h_v;
      }

      .info-title-t1 {
        float: left;
        line-height: 28px;
        @include fc_a_v;
      }

      a {
        line-height: 26px;
        float: right;
        padding: 0 15px;
        border-radius: 2px;
        border: 1px solid transparent;
        box-sizing: border-box;
        @include bg_m_v;
        @include fc_fff_v;
        @include to_m_lb_v;
      }
    }

    .info {
      display: flex;
      position: relative;
      line-height: 26px;
      margin-bottom: 8px;
      @include fc_h_v;

      .info-one {
        float: left;
        width: 220px;

        .adjkticon {
          @include fc_i_i_v;
        }

        .info-assets {
          font-size: 22px;
        }

        .set-currency {
          .set-btn {
            cursor: pointer;
            position: relative;
          }

          .currency-list {
            z-index: 4;
            line-height: 40px;
            position: absolute;
            border-radius: 4px;
            padding: 8px 0;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
            max-height: 500px;
            overflow-y: auto;
            @include bs_fl_v(0px 4px 10px 0px);
            @include bg_t_v;

            .currency-li {
              padding: 0 12px;
              transition: 0.3s;
              cursor: pointer;

              &:hover {
                @include bg_m_f_v(0.08);
              }
            }

            .active {
              @include fc_m_v;
            }

            .link {
              @include fc_m_v;
            }
          }

          .iconActive {
            transform: rotate(180deg);
            @include fc_m_v;
          }
        }
      }

      .info-two {
        float: left;
        text-align: right;

        .info-tb {
          font-size: 14px;
        }
      }

      .info-sp {
        float: left;
        margin: 8px 6px 8px 16px;
        width: 1px;
        @include lb_s_v;
      }

      .info-three {
        flex: 1;
        display: flex;
        flex-wrap: wrap;
        li {
          display: inline-block;
          min-width: 105px;
          padding: 0 10px;
          list-style: none;
        }
      }
    }
  }
}
.swiper-slide {
  .tit {
    @include fc_a_v;
  }

  .swp-title {
    line-height: 20px;

    .swp-t-l,
    .swp-t-r {
      @include fc_h_v;
      width: 50%;
      float: left;
    }

    .swp-t-l {
      @include fc_h_v;
      .l1,
      .l2 {
        transition: 0.3s;
      }

      .l2 {
        font-size: 18px;
      }
    }

    .swp-t-r {
      text-align: right;
    }
  }

  .swp-u {
    line-height: 20px;
    margin-top: 8px;
    transition: 0.3s;

    .swp-l {
      float: left;
      margin-top: 5px;

      .swp-l-v {
        @include fc_h_v;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      a {
        @include to;
      }
    }
  }
}
.amount-hover{
    display: flex;
    justify-content: space-between;
}
.tabBtn {
  height: 100%;
  display: flex;
  align-items: center;
  .svg-btn {
    >span {
      display: flex;
      align-items: center;
    }
  }
  .crawling {
    font-size: 17px;
    @include fc_m_v;
  }
}
.adaptation-show {
  display: none;
}
@media screen and (max-width: 1900px) {
  .bank-account .bank-account-info .info .info-three li.is-adaptation {
    display: none;
  }
  .adaptation-show {
    display: block;
  }
}

</style>
