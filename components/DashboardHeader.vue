<template>
  <section class="hero is-medium">
    <div class="hero-body">
      <div class="container">
        <div
          class="columns is-align-items-center has-text-white has-text-centered"
        >
          <div class="column">
            <h3 class="animate__animated animate__fadeInDown">
              Current Balance
            </h3>
            <span
              v-if="balanceLoaded"
              class="is-size-1 animate__animated animate__fadeInDown"
            >
              ${{
                this.balance
                  .toFixed(10)
                  .replace(/\d(?=(\d{3})+\.)/g, '$&,')
                  .split('.')[0]
              }}
            </span>
            <span
              v-if="balanceLoaded"
              class="is-size-5 has-text-grey-lighter decimals animate__animated animate__fadeInDown"
            >
              .{{
                this.balance
                  .toFixed(10)
                  .replace(/\d(?=(\d{3})+\.)/g, '$&,')
                  .split('.')[1]
              }}
            </span>

            <b-loading :active="!balanceLoaded" :can-cancel="true"></b-loading>
            <h3 class="animate__animated animate__fadeInDown">
              Total Earnings
            </h3>
            <span v-if="balanceLoaded" class="is-size-1">
              ${{
                this.yield
                  .toFixed(10)
                  .replace(/\d(?=(\d{3})+\.)/g, '$&,')
                  .split('.')[0]
              }}
            </span>
            <span
              v-if="balanceLoaded"
              class="is-size-5 has-text-grey-lighter decimals animate__animated animate__fadeInDown"
            >
              .{{
                this.yield
                  .toFixed(10)
                  .replace(/\d(?=(\d{3})+\.)/g, '$&,')
                  .split('.')[1]
              }}
            </span>
            <h6
              class="has-text-grey-lighter mt-2 animate__animated animate__fadeInDown"
            >
              Earnings per Second
            </h6>
            <span
              v-if="balanceLoaded"
              class="is-size-5 has-text-text-white decimals animate__animated animate__fadeInDown"
            >
              ${{
                this.earnedInterest
                  .toFixed(10)
                  .replace(/\d(?=(\d{3})+\.)/g, '$&,')
              }}
            </span>
          </div>
          <div class="pa-6 column is-one-quarter">
            <h3 class="animate__animated animate__fadeInDown">APY</h3>
            <h1 class="is-size-1 animate__animated animate__fadeInDown">
              {{ this.apy.toFixed(2) }}%
            </h1>
            <div
              class="mt-6 is-flex is-justify-content-center is-align-content-center level"
            >
              <b-button
                class="mr-1 level-item animate__animated animate__fadeInUp"
                @click="openDeposit"
                >Deposit</b-button
              >
              <b-button
                class="ml-1 level-item animate__animated animate__fadeInUp"
                @click="openWithdrawl"
                >Withdraw</b-button
              >
            </div>
          </div>
          <div
            class="column animate__animated animate__fadeInDown chart-container is-three-fifth"
          >
            Historical Chart
            <line-chart />
          </div>
        </div>
      </div>
    </div>
    <withdraw :balance="balance" ref="withdrawlmodal" />
    <deposit :balance="balance" ref="depositmodal" />
  </section>
</template>

<script>
import Withdraw from '~/components/Withdraw'
import Deposit from '~/components/Deposit'
import LineChart from '~/components/LineChart'
import fTokenABI from '~/helpers/fToken.json'

export default {
  components: { LineChart, Deposit, Withdraw },
  data() {
    return {
      balance: 0,
      apy: 0,
      yield: 0,
      withdrawlOpen: false,
      earnedInterest: 0,
      balanceLoaded: false,
    }
  },
  async mounted() {
    const accounts = await ethereum.request({ method: 'eth_accounts' })
    //We take the first address in the array of addresses and display it
    this.isLoggedIn = accounts[0]
    this.fDaiInstance = new this.$web3.eth.Contract(
      fTokenABI.abi,
      '0xF80cFBbed73261E3802603aEDF76bDb25530d328'
    )

    const aave = await this.$axios.$get(
      'https://api.zapper.fi/v1/lending-stats/aave?network=ethereum&api_key=96e0cc51-a62e-42ca-acee-910ea7d2a241'
    )
    this.aaveAPY = aave[1].supplyApy.toFixed(2) * 100

    const compound = await this.$axios.$get(
      'https://api.zapper.fi/v1/lending-stats/compound?network=ethereum&api_key=96e0cc51-a62e-42ca-acee-910ea7d2a241'
    )
    this.compoundAPY = compound[1].supplyApy.toFixed(2) * 100
    this.apy = this.aaveAPY > this.compoundAPY ? this.aaveAPY : this.compoundAPY
    console.log(this.apy)

    this.balance = Number(
      (await this.fDaiInstance.methods.balanceOf(this.isLoggedIn).call()) /
        Math.pow(10, 18)
    )
    this.balanceLoaded = true
    ;(this.$nuxt || EventBus || this.$EventBus).$on(
      'addToBalance',
      this.addToBalance
    )
    ;(this.$nuxt || EventBus || this.$EventBus).$on(
      'removeFromBalance',
      this.removeFromBalance
    )
    ;(this.$nuxt || EventBus || this.$EventBus).$on(
      'getBalance',
      this.getBalance
    )
    this.earnedInterest = this.balance * (this.apy / 100 / 365 / 24 / 60 / 60)
    setInterval(() => {
      let balance = Number(this.balance)
      let apy = Number(this.apy)
      this.earnedInterest = balance * (apy / 100 / 365 / 24 / 60 / 60)
      let newBalance = balance + this.earnedInterest
      let newYield = Number(this.yield) + this.earnedInterest
      this.balance = newBalance
      this.yield = newYield
      // this.emitUpdateTransaction([
      //   {
      //     id: (Math.random() * 100).toFixed(0),
      //     date: new Date(),
      //     transaction: {
      //       name: 'Farmtopia Harvest',
      //       type: 'Interest',
      //       amount: this.earnedInterest,
      //     },
      //   },
      // ])
    }, 1000)
  },
  methods: {
    emitUpdateTransaction() {
      ;(this.$nuxt || EventBus || this.$EventBus).$emit('updateTransactions')
    },
    addToBalance(balanceIncrement) {
      this.balance = this.balance + Number(balanceIncrement.amount)
      this.emitUpdateTransaction()
    },
    removeFromBalance(balanceDecrement) {
      this.balance = this.balance - Number(balanceDecrement.amount)
      this.emitUpdateTransaction()
    },
    getBalance() {
      return this.balance
    },
    openWithdrawl() {
      return this.$refs.withdrawlmodal.open()
    },
    openDeposit() {
      return this.$refs.depositmodal.open()
    },
  },
}
</script>
<style scoped>
.chart-container {
  height: 400px;
}

.level-item {
  border-radius: 50px;
}

.decimals {
  margin-left: -13px;
}

.automargin {
  margin-top: auto;
}
</style>
