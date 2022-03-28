<template>
  <div id="page-demo" class="unipass-page">
    <i class="background-logo iconfont icon-logo"></i>
    <div class="head">mix-UniPass Demo</div>
    <div v-if="username">
      <div>
        <br />
        <h3>{{ username }}</h3>
        <br />
      </div>
      <el-button class="transfer" type="primary" @click="logout"
        >logout</el-button
      >
    </div>
    <div v-else>
      <el-button type="primary" class="transfer login" @click="connect"
        >login</el-button
      >
    </div>
    <el-tabs
      v-show="username"
      v-model="activeTab"
      class="body"
      type="border-card"
    >
      <el-tab-pane label="RPG Transaction" name="first">
        <el-form
          ref="form"
          class="body-input"
          label-position="top"
          :model="form"
          @submit.native.prevent
        >
          <el-form-item label="Your Address:" prop="address">
            <template #label>
              <span>Your Address:</span>
              <i
                v-show="myAddress"
                class="iconfont icon-copy sea-background"
                @click="bindCopy"
              ></i>
            </template>
            <el-input
              v-model="myAddress"
              disabled
              readonly
              type="textarea"
              resize="none"
              :autosize="{ minRows: 1 }"
            />
          </el-form-item>
          <span>
            <b style="color: black">
              <a href="https://robin-faucet.rangersprotocol.com/home">
                Rangers Faucet:
              </a>
            </b>
          </span>
          <el-form-item label="Your Balance:" prop="address">
            <el-input v-model="myBalanceFormat" disabled readonly />
          </el-form-item>
          <el-form-item label="Transfer RPG To:" prop="address">
            <el-input v-model="toAddress" clearable />
          </el-form-item>
          <el-form-item label="Amount:" prop="address">
            <el-input v-model="toAmount" clearable />
          </el-form-item>
        </el-form>

        <br />
        <div>
          <el-button type="primary" class="transfer" @click="sendRPG"
            >send</el-button
          >
        </div>

        <div>{{ txHash }}</div>
      </el-tab-pane>
      <el-tab-pane label="Sign Message" name="second">
        <div>
          <br />
          <h3 class="input">Message:</h3>
          <el-input
            v-model="message"
            type="textarea"
            :autosize="{ minRows: 8, maxRows: 10 }"
            resize="none"
          ></el-input>
          <br />
          <div class="message">
            <el-button type="primary" class="message-button" @click="authorize"
              >authorize</el-button
            >
            <el-button type="primary" class="message-button" @click="verifySig"
              >verify</el-button
            >
          </div>
          <br />
          <div v-if="signature">
            <h3 class="input">Signature:</h3>
            <el-input
              v-model="signature"
              type="textarea"
              :autosize="{ minRows: 8, maxRows: 10 }"
              resize="none"
            ></el-input>
          </div>
        </div>
      </el-tab-pane>
            <el-tab-pane label="nft" name="third">
            <div>
          <br />
          <h3 class="input">nft数量:{{ nftNum }}</h3>
          <div class="message">
            <el-button type="primary" class="message-button" @click="claim"
              >claim</el-button
            >
              <el-button type="success" class="message-button" @click="claim1"
              >claim1</el-button
            >
          </div>
        </div>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { UPAuthMessage, UPAuthResponse } from 'up-core-test'
import { ChainID, UPRangers } from 'up-rangers'
import {mixUnipass} from "mix-unipass";
import {contractConfig} from "@/config/contract";

const DAI_ADDRESS = '0x25c58Aa062Efb4f069bD013De3e3C5797fb40651'

export default Vue.extend({
  data() {
    return {
      username: '',
      message: 'TO BE SIGNED MESSAGE abc',
      sig: '',
      nftNum:0,
      signature:'',
      activeTab: 'first',
      myAddress: '',
      myBalance: '0.00',
      toAddress: '0x8291507Afda0BBA820efB6DFA339f09C9465215C',
      toAmount: '0.01',
      txHash: '',
      form: {},
      mixUnipass: new mixUnipass(
        {contracts:contractConfig}
      ),
      // STEP 1: create UPRangers instance
      upRangers: new UPRangers({
        chainID: ChainID.testnet,
        userInfoContract: process.env.RANGERS_UNIPASS_CONTRACT,
        upCoreConfig: {
          domain: process.env.UNIPASS_URL,
          // protocol: 'http',
        },
      }),
    }
  },
  computed: {
    myBalanceFormat(): string {
      return this.myBalance + ' RPG'
    },
  },
  methods: {
    bindCopy() {
      this.$clipboard(this.myAddress)
      this.$message.success('copy succeeded')
    },

    async connect() {
      console.log('connect clicked')
      try {
        // STEP 2: connect unipass
        const account = await this.mixUnipass.connect();
        this.username = account.username
        console.log('account', account)

        this.myAddress = this.mixUnipass.getAddress()
        await this.refreshBalance()
        await this.refreshNftNum();
      } catch (err) {
        this.$message.error(err as string)
        console.log('connect err', err)
      }
    },
    async refreshBalance() {
        this.myBalance = await this.mixUnipass.getBalance(this.myAddress)
    },

    async refreshNftNum(){
       this.nftNum = await this.mixUnipass.nativeCall("nft", "balanceOf", [this.myAddress]);
    },
    logout() {
      console.log('connect clicked')
      this.mixUnipass.disconnect()
      this.username = ''
    },
    async authorize(){
      let resp = await this.mixUnipass.sign(this.message);
       this.sig = resp
       this.signature = JSON.stringify(resp);
    },

    async verifySig() {

     let res =  await this.mixUnipass.verifySig(this.message, this.sig);
     if (res === true) {
          this.$message.success('verify signature success')
        } else {
          this.$message.error('verify signature failed')
        }
    },

    async sendRPG() {
      if (Number(this.myBalance) < Number(this.toAmount)) {
        this.$message.error('balance is not enough')
        return
      }
      try {
       this.txHash = await this.mixUnipass.sendRpg(this.toAddress, this.toAmount);
      this.$message.success(`send RPG success, tx hash = ${this.txHash}`)

        await this.refreshBalance()
      } catch (err) {
        this.$message.error(err as string)
        console.log('err', err)
      }
    },
    //NativeCall
    async claim(){
      const res = await this.mixUnipass.executeNativeCall("nft", "cliam", [], 0);
      this.refreshNftNum();
    },
    //
    async claim1(){
     const res = await this.mixUnipass.executeCall(
  "0xe73B04A625e8eDE58fBf8BA3BA6Aa3f733DbA5d4", //合约地址
  {																							//合约abi
    "inputs": [],
    "name": "cliam",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  [],																					 //方法参数
  "0"																					 //原生代币数量
);
    this.refreshNftNum();
    }
  },
})
</script>

<style lang="stylus">
#page-demo {
  max-width: 480px;
  margin: 0 auto;
  overflow: hidden;
  position: relative;
  background: #F5F5FF;

  > * {
    z-index: 1;
  }

  > .background-logo {
    font-size: 237px;
    position: absolute;
    top: 16px;
    right: -40px;
    color: #5575ff;
    opacity: 0.14;
    z-index: 0;
  }

  .head {
    text-align: left;
    font-family: Helvetica;
    font-style: normal;
    font-weight: bold;
    font-size: 20px;
    line-height: 20px;
    color: black;
  }

  .transfer {
    width: 100%;
    font-size: 20px;
  }

  .login {
    margin-top: 50px;
    font-size: 20px;
  }

  .body {
    border-radius: 24px;
    margin: 30px auto 0px;
    width: 100%;
    background: #FFFFFF;
    padding: 0px 0 21px;
    overflow: hidden;

    .body-input {
      margin-top: -20px;

      .icon-copy {
        cursor: pointer;
      }
    }

    .input {
      text-align: left;
      margin-bottom: 20px;
    }
  }

  .message {
    display: flex;
    justify-content: space-between;

    .message-button {
      margin-top: 30px;
      width: 48%;
      font-size: 20px;
    }
  }
}

.unipass-page {
  padding: 24px;
  padding-top: 29px;
  width: 100%;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  text-align: center;
}
</style>
