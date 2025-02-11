<template>
  <div class="page d-flex">
    <div class="migrate">
      <h3>Upgrade to V2</h3>
      <div>
        <div class="stats d-flex flex-justify-between mt-4">
          <div>
            <MigrationStats
              :pool="poolV1"
              :is-v1="true"
              :details="poolStatDetails"
              :loading="loading"
              @toggle-details="handleToggleStats"
            />
          </div>
          <div class="arrow">→</div>
          <div>
            <MigrationStats
              :pool="poolV2"
              :is-v1="false"
              :details="poolStatDetails"
              :loading="loading"
              @toggle-details="handleToggleStats"
            />
            <MigrationWallet
              class="mt-2"
              v-if="leftoverAssets.length > 0"
              :assets="leftoverAssets"
              :loading="loading"
            />
          </div>
        </div>
        <div class="mt-4 d-flex flex-justify-center">
          <ButtonMigrate
            v-if="isUnlocked"
            :disabled="isDisabled"
            :pending="pendingTx"
            @click="migratePool"
            class="button-primary"
          >
            Migrate {{ _num(liquidity, 'usd') }}
          </ButtonMigrate>
          <ButtonMigrate
            v-else
            :disabled="isDisabled"
            :pending="pendingTx"
            @click="unlockPool"
            class="button-primary"
          >
            Unlock
          </ButtonMigrate>
        </div>
        <div v-if="loading" class="mt-5 impact-label-loading">
          <div class="bg-gray rounded-1 anim-pulse" />
        </div>
        <div
          v-else-if="priceImpact > MIN_PRICE_IMPACT"
          class="mt-5 d-flex impact-label"
          @click="toggleAdvancedOptions"
        >
          Price impact:
          {{ isFullMigration ? _num(priceImpact, 'percent') : '0%' }}
          <div class="ml-2">
            <IconChevronUp v-if="advancedOptions" />
            <IconChevronDown v-else />
          </div>
        </div>
        <div v-if="advancedOptions">
          <div class="options mt-3">
            <div
              class="option d-flex flex-items-center p-3"
              @click="setMigrationType(true)"
            >
              <div class="option-input" :class="{ active: isFullMigration }">
                <div class="option-input-circle" v-if="isFullMigration"></div>
              </div>
              <div class="ml-3">
                <div class="option-type">
                  Migrate fully, no tokens left behind
                </div>
                <div class="option-impact">
                  {{ _num(priceImpact, 'percent') }} price impact
                </div>
              </div>
            </div>
            <div
              class="option d-flex flex-items-center p-3"
              @click="setMigrationType(false)"
            >
              <div class="option-input" :class="{ active: !isFullMigration }">
                <div class="option-input-circle" v-if="!isFullMigration"></div>
              </div>
              <div class="ml-3">
                <div class="option-type">
                  Migrate with minimal price impact
                </div>
                <div class="option-impact">
                  0% price impact
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="overlay" v-if="tx">
        <h3>Successful migration</h3>
        <div>
          <div class="icon-wrapper">
            <IconCheckCircle class="icon" />
          </div>
          <div>Your liquidity has been migrated to V2</div>
          <div class="overlay-buttons">
            <div class="overlay-button">
              <a :href="_etherscanLink(tx, 'tx')" target="_blank">
                Receipt
              </a>
            </div>
            <div class="overlay-button">
              <a href="https://app.balancer.fi/" target="_blank">
                My V2 liquidity
              </a>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapActions } from 'vuex';
import { getAddress } from '@ethersproject/address';
import Pool from '@/_balancer/pool';
import PoolV2 from '@/_balancer/pool2';
import { bnum, scale } from '@/helpers/utils';
import { getPoolLiquidity } from '@/helpers/price';
import {
  getNewPool,
  calculateMinAmount,
  calculatePriceImpact,
  getLeftoverAssets
} from '@/helpers/migration';
import config from '@/config';

const MAX_PRICE_IMPACT = 0.01;
const MIN_PRICE_IMPACT = 0.001;

export default {
  data() {
    return {
      MIN_PRICE_IMPACT,
      pool: this.$route.params.id,
      pendingTx: false,
      advancedOptions: false,
      poolStatDetails: false,
      isFullMigration: true,
      allowance: '',
      balance: '',
      poolV1: {},
      poolV2: {},
      priceImpact: 0,
      leftoverAssets: [],
      tx: null
    };
  },
  watch: {
    $route() {
      const pool = this.$route.params.id;
      if (pool !== this.pool) {
        this.pool = pool;
        this.poolV1 = {};
        this.poolV2 = {};
        this.allowance = '';
        this.balance = '';
        this.init();
      }
    },
    'web3.dsProxyAddress': async function(val, prev) {
      console.log('[Migration] ds proxy instance', val);
      if (val !== prev) await this.fetchUserState(true);
    }
  },
  computed: {
    isUnlocked() {
      const allowanceNumber = bnum(this.allowance);
      const balanceNumber = bnum(this.balance);
      return balanceNumber.isZero() || allowanceNumber.gte(balanceNumber);
    },
    isDisabled() {
      if (this.loading) {
        return true;
      }
      if (this.isUnlocked) {
        const balanceNumber = bnum(this.balance);
        return balanceNumber.isZero();
      } else {
        return false;
      }
    },
    liquidity() {
      const balanceNumber = scale(bnum(this.balance), -18);
      const liquidity = balanceNumber
        .div(this.poolV1.totalShares)
        .times(this.poolV1.liquidity);
      return liquidity;
    },
    loading() {
      const poolV1Loading = Object.keys(this.poolV1).length === 0;
      const poolV2Loading = Object.keys(this.poolV2).length === 0;
      const userStateLoading = this.allowance === '' || this.balance === '';
      return poolV1Loading || poolV2Loading || userStateLoading;
    }
  },
  async mounted() {
    this.init();
  },
  methods: {
    ...mapActions([
      'approve',
      'migrateAll',
      'migrateProportionally',
      'getBalances',
      'getAllowances'
    ]),
    async init() {
      await this.fetchPool();
      await this.fetchPoolV2();
      this.priceImpact = calculatePriceImpact(
        this.balance,
        this.poolV1,
        this.poolV2
      );
      if (this.priceImpact > MAX_PRICE_IMPACT) {
        this.isFullMigration = false;
      }
      this.leftoverAssets = getLeftoverAssets(
        this.balance,
        this.poolV1,
        this.poolV2,
        this.isFullMigration
      );
    },
    async fetchUserState(isStateUpdated) {
      console.log('[Migration] fetching user state');
      if (!this.web3.account) {
        console.log('[Migration] can not fetch user state: no account');
        if (isStateUpdated) {
          this.allowance = '0';
          this.balance = '0';
        }
        return;
      }
      if (this.web3.dsProxyAddress === null) {
        console.log('[Migration] can not fetch user state: no ds proxy');
        return;
      }
      if (this.web3.dsProxyAddress === '') {
        console.log(
          '[Migration] can not fetch user state: no ds proxy instance'
        );
        this.allowance = '0';
        this.balance = '0';
        return;
      }
      const data = await Promise.all([
        this.getAllowances([this.pool]),
        this.getBalances([this.pool])
      ]);
      this.allowance = data[0][this.pool][this.web3.dsProxyAddress];
      this.balance = data[1][this.pool];
      console.log('[Migration] fetched user state');
    },
    async fetchPool() {
      console.log('[Migration] fetching v1 pool');
      const pool = new Pool(this.pool);
      this.poolV1 = await pool.getMetadata();
      console.log('[Migration] fetched v1 pool');
      this.poolV1.address = this.pool;
      this.poolV1.liquidity = getPoolLiquidity(this.poolV1, this.price.values);
      await this.fetchUserState(false);
    },
    async fetchPoolV2() {
      console.log('[Migration] fetching v2 pool');
      const address = getNewPool(this.pool);
      const poolV2 = new PoolV2(address);
      this.poolV2 = await poolV2.getMetadata();
      console.log('[Migration] fetched v2 pool');
      const totalWeight = this.poolV2.tokens.reduce(
        (total, token) => total.plus(token.denormWeight),
        bnum(0)
      );
      const poolData = {
        tokens: this.poolV2.tokens.map((token, index) => {
          const weightNumber = bnum(this.poolV2.tokens[index].denormWeight);
          const decimals = this.poolV1.tokens.find(
            t => t.address === token.address.toLowerCase()
          ).decimals;
          return {
            checksum: getAddress(token.address),
            balance: scale(bnum(this.poolV2.tokens[index].balance), -decimals),
            weightPercent: weightNumber
              .div(totalWeight)
              .times(100)
              .toNumber()
          };
        })
      };
      this.poolV2.address = address;
      this.poolV2.liquidity = getPoolLiquidity(poolData, this.price.values);
      const swapFeeNumber = scale(bnum(this.poolV2.swapFee), -18);
      this.poolV2.swapFee = swapFeeNumber.toString();
    },
    async migratePool() {
      const vault = config.addresses.vault;
      const poolIn = this.poolV1.address;
      const poolInAmount = this.balance;
      const tokenOutAmountsMin = this.poolV1.tokens.map(() => '0');
      const poolOut = this.poolV2.address;
      const poolOutAmountMin = calculateMinAmount(
        this.isFullMigration,
        this.balance,
        this.poolV1,
        this.poolV2
      );
      this.pendingTx = true;
      const txParams = {
        vault,
        poolIn,
        poolInAmount,
        tokenOutAmountsMin,
        poolOut,
        poolOutAmountMin
      };
      const txPromise = this.isFullMigration
        ? this.migrateAll(txParams)
        : this.migrateProportionally(txParams);
      console.log('[Migration] sending migration tx');
      const tx = await txPromise;
      console.log('[Migration] sent migration tx');
      if (tx) {
        const txReceipt = await tx.wait();
        console.log('[Migration] confirmed migration tx');
        const txHash = txReceipt.transactionHash.toString();
        this.tx = txHash;
      }
      this.pendingTx = false;
    },
    async unlockPool() {
      console.log('[Migration] sending unlock tx');
      this.pendingTx = true;
      await this.approve(this.pool);
      console.log('[Migration] sent unlock tx');
      const data = await this.getAllowances([this.pool]);
      this.allowance = data[this.pool][this.web3.dsProxyAddress];
      this.pendingTx = false;
    },
    setMigrationType(isFullMigration) {
      this.isFullMigration = isFullMigration;
      this.leftoverAssets = getLeftoverAssets(
        this.balance,
        this.poolV1,
        this.poolV2,
        isFullMigration
      );
    },
    toggleAdvancedOptions() {
      this.advancedOptions = !this.advancedOptions;
    },
    handleToggleStats() {
      this.poolStatDetails = !this.poolStatDetails;
    }
  }
};
</script>

<style scoped>
.page {
  flex-direction: column;
  color: white;
}

.migrate {
  position: relative;
  width: 440px;
  margin: 120px auto;
  padding: 28px 26px;
  display: flex;
  flex-direction: column;
  border: 1px solid #333333;
  border-radius: 25px;
  background: #21222c;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  padding: 28px 26px;
  border-radius: 25px;
  background: #21222c;
  text-align: center;
  display: flex;
  justify-content: space-between;
  flex-direction: column;
}

.icon-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 16px;
}

.icon {
  width: 80px;
  height: 80px;
  color: #10b981;
  background: #064e3b;
  border-radius: 50%;
  padding: 16px;
}

.overlay-buttons {
  display: flex;
  margin-top: 16px;
  justify-content: space-between;
}

.overlay-button {
  background: #282932;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0 10px;
  outline: none;
  height: 44px;
  width: 46%;
  cursor: pointer;
  line-height: 40px;
  font-size: 16px;
  margin: 0;
}

.arrow {
  margin: 8px 0;
  display: flex;
  justify-content: center;
  font-size: 18px;
  color: #90a4ae;
}

.impact-label-loading {
  height: 19.6px;
}

.impact-label-loading > div {
  height: 90%;
  width: 30%;
}

.impact-label {
  color: #90a4ae;
  cursor: pointer;
}

.options {
  border: 1px solid #2e2e2e;
  border-radius: 5px;
}

.option {
  cursor: pointer;
}

.option:not(:first-child) {
  border-top: 1px solid #2e2e2e;
}

.option-input {
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid #616161;
  border-radius: 50%;
}

.option-input.active {
  background: #44d7b6;
  border: none;
}

.option-input-circle {
  width: 8px;
  height: 8px;
  background: #21222c;
  border-radius: 50%;
}

.option-type {
  font-size: 16px;
  font-weight: bold;
}

.option-impact {
  color: #90a4ae;
}

@media (max-width: 768px) {
  .migrate {
    width: 100%;
    padding: 18px;
  }
}
</style>
