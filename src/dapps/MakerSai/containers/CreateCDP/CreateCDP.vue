<template>
  <div class="container-maker">
    <dai-confirmation-modal
      ref="daiconfirmation"
      :opencdp="openCdp"
      :txinfo="txInfo"
      :liquidation-price="liquidationPrice"
      :collat-ratio="displayFixedPercent(collatRatio)"
      :liquidation-penalty="displayPercentValue(liquidationPenalty)"
      :min-ratio="displayPercentValue(liquidationRatio)"
      :current-price="displayFixedValue(ethPrice, 2)"
      :collateral="ethQty.toString()"
      :generate="daiQty.toString()"
    />
    <loading-overlay
      v-if="loading"
      :loadingmessage="$t('dappsMaker.creating-message')"
    />
    <div class="manage-container">
      {{ $t('dappsMaker.no-longer-active-notice') }}
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import ethUnit from 'ethjs-unit';
import DaiConfirmationModal from '../../components/DaiConfirmationModal';
import LoadingOverlay from '@/components/LoadingOverlay';
import {
  displayFixedValue,
  displayPercentValue,
  displayFixedPercent
} from '../../helpers';

import BigNumber from 'bignumber.js';
import Arrow from '@/assets/images/etc/single-arrow.svg';

const toBigNumber = num => {
  return new BigNumber(num);
};

const bnOver = (one, two, three) => {
  return toBigNumber(one).times(toBigNumber(two)).div(toBigNumber(three));
};

export default {
  components: {
    'dai-confirmation-modal': DaiConfirmationModal,
    'loading-overlay': LoadingOverlay
  },
  props: {
    tokensWithBalance: {
      type: Array,
      default: function () {
        return [];
      }
    },
    getBalance: {
      type: Function,
      default: function () {}
    },
    highestGas: {
      type: String,
      default: '0'
    },
    ethPrice: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    pethPrice: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    liquidationPenalty: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    stabilityFee: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    liquidationRatio: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    wethToPethRatio: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    pethMin: {
      type: BigNumber,
      default: toBigNumber(0)
    },
    priceService: {
      type: Object,
      default: function () {
        return {};
      }
    },
    cdpService: {
      type: Object,
      default: function () {
        return {};
      }
    },
    proxyService: {
      type: Object,
      default: function () {
        return {};
      }
    },
    buildEmpty: {
      type: Function,
      default: function () {}
    },
    values: {
      type: Object,
      default: function () {
        return {
          maxPethDraw: '',
          maxEthDraw: '',
          maxUsdDraw: '',
          ethCollateral: '',
          pethCollateral: '',
          usdCollateral: '',
          debtValue: '',
          maxDai: '',
          collateralRatio: '',
          cdpId: '',
          stabilityFee: '',
          minEth: '',
          liquidationRatio: '',
          wethToPethRatio: '',
          liquidationPenalty: '',
          targetPrice: '',
          pethPrice: ''
        };
      }
    }
  },
  data() {
    return {
      arrowImage: Arrow,
      daiPrice: 0,
      priceFloor: 0,
      ethQty: 0,
      daiQty: 0,
      txInfo: {},
      loading: false
    };
  },
  computed: {
    ...mapState('main', ['account', 'gasPrice', 'web3', 'network', 'ens']),
    validInputs() {
      if (toBigNumber(this.ethQty).isNaN() || toBigNumber(this.daiQty).isNaN())
        return false;
      if (toBigNumber(this.ethQty).gt(0)) {
        if (toBigNumber(this.ethQty).lte(this.values.minEth)) return false;
        if (toBigNumber(this.maxDaiDraw).lte(toBigNumber(this.daiQty)))
          return false;
        if (toBigNumber(this.collatRatio).lte(1.501)) return false;
        return toBigNumber(ethUnit.toWei(this.ethQty, 'ether').toString()).lte(
          this.account.balance
        );
      }
      return false;
    },
    hasEnoughEth() {
      if (toBigNumber(this.ethQty).isNaN()) return false;
      return toBigNumber(ethUnit.toWei(this.ethQty, 'ether').toString()).lte(
        this.account.balance
      );
    },
    atSetFloor() {
      if (this.priceFloor <= 0) return 0;
      return bnOver(this.ethPrice, this.liquidationRatio, this.priceFloor);
    },
    collatRatio() {
      if (this.daiQty <= 0 || this.ethQty <= 0) return 0;
      return this.calcCollatRatio(this.ethQty, this.daiQty);
    },
    liquidationPrice() {
      if (this.daiQty <= 0 || this.ethQty <= 0) return 0;
      return this.calcLiquidationPrice(this.ethQty, this.daiQty);
    },
    maxDaiDraw() {
      if (this.ethQty <= 0) return 0;
      const bufferVal = this.calcDaiDraw(this.ethQty).times(0.01);
      return toBigNumber(this.calcDaiDraw(this.ethQty)).minus(bufferVal);
    },
    minEthDeposit() {
      if (this.daiQty <= 0) return 0;
      return this.calcMinEthDeposit(this.daiQty);
    },
    risky() {
      const collRatio = this.collatRatio;
      if (toBigNumber(collRatio).gt(0)) {
        return toBigNumber(collRatio).lte(2);
      }
      return false;
    },
    veryRisky() {
      const collRatio = this.collatRatio;
      if (toBigNumber(collRatio).gt(0)) {
        return toBigNumber(collRatio).lte(1.75);
      }
      return false;
    },
    depositInPeth() {
      if (this.ethQty <= 0) return 0;
      return this.toPeth(this.ethQty);
    },
    minEth() {
      if (this.wethToPethRatio) {
        return toBigNumber(this.pethMin).times(this.wethToPethRatio);
      }
      return '--';
    }
  },
  async mounted() {
    this.buildEmptyInstance();
  },
  methods: {
    async buildEmptyInstance() {
      this.makerCDP = await this.buildEmpty();
      this.$forceUpdate();
    },
    displayPercentValue,
    displayFixedValue,
    displayFixedPercent,
    async openCdp() {
      this.loading = true;

      if (this.ethQty <= 0) return 0;
      setTimeout(() => {
        this.loading = false;
      }, 5000);

      // [Note from David to Steve] This should be implemented on TX core.
      // Close DAI confirmation modal
      this.$eventHub.$on('showTxConfirmModal', () => {
        this.$emit('cdpOpened');
        if (this.loading) {
          this.$refs.daiconfirmation.$refs.modal.hide();
          this.loading = false;
        }
      });

      await this.makerCDP.openCdp(this.ethQty, this.daiQty);
    },
    openDaiConfirmation() {
      this.$refs.daiconfirmation.$refs.modal.show();
    },
    toUSD(eth) {
      if (eth === undefined || eth === null) return toBigNumber(0);
      const toUsd = this.ethPrice.times(toBigNumber(eth));
      if (toUsd.lt(0)) {
        return toBigNumber(0);
      }
      return toUsd;
    },

    toPeth(eth) {
      if (!toBigNumber(eth).eq(0)) {
        return toBigNumber(eth).div(this.wethToPethRatio);
      }
      return toBigNumber(0);
    },
    fromPeth(peth) {
      if (!toBigNumber(peth).eq(0)) {
        return toBigNumber(peth).times(this.wethToPethRatio);
      }
      return toBigNumber(0);
    },
    calcMinCollatRatio(priceFloor) {
      return bnOver(this.ethPrice, this.liquidationRatio, priceFloor);
    },
    calcDaiDraw(
      ethQty,
      ethPrice = this.ethPrice,
      liquidationRatio = this.liquidationRatio
    ) {
      if (ethQty <= 0) return 0;
      return bnOver(ethPrice, toBigNumber(ethQty), liquidationRatio);
    },

    calcMinEthDeposit(
      daiQty,
      ethPrice = this.ethPrice,
      liquidationRatio = this.liquidationRatio
    ) {
      if (daiQty <= 0) return 0;
      return bnOver(liquidationRatio, daiQty, ethPrice);
    },

    calcCollatRatio(ethQty, daiQty) {
      if (ethQty <= 0 || daiQty <= 0) return 0;
      return bnOver(this.ethPrice, ethQty, daiQty);
    },

    calcLiquidationPrice(ethQty, daiQty) {
      if (ethQty <= 0 || daiQty <= 0) return 0;
      const getInt = parseInt(this.ethPrice);
      for (let i = getInt; i > 0; i--) {
        const atValue = bnOver(i, ethQty, daiQty).lte(this.liquidationRatio);
        if (atValue) {
          return i;
        }
      }
      for (let i = 100; i > 0; i--) {
        const atValue = bnOver(i / 100, ethQty, daiQty).lte(
          this.liquidationRatio
        );
        if (atValue) {
          return i / 100;
        }
      }
      return 0;
    }
  }
};
</script>

<style lang="scss" scoped>
@import 'CreateCDP';
</style>
