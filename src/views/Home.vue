<template>
  <div class="wallet">
    <div type="hidden" id="workstorage"></div>
      <div id="login" class="page" :class="{ active: open===false }">
        <div class="title rpc">RPC Server : <span>{{ $store.state.app.node.address }}</span></div>
        <div id="inputs">
          <div v-if="error !== null" class="error">{{ error }}</div>
          <div class="labeltabs">
          <label @click="setSeed" class="df" :class="{ active: seedtab === true}" for="seed">
            Seed
            </label>
          <label @click="setPrivate" class="df" :class="{ active: seedtab !== true}" for="key">
            Private Key
            </label>
            <a v-if="seedtab === true" class="morebutton mla" href="" @click.prevent="showadvanced = !showadvanced"><i data-fa-transform="grow-20" class="fal fa-ellipsis-h"></i></a>

            </div>
          <div v-if="seedtab === true" class="login">
            <input v-model="seed" :type="logintype" placeholder="Seed" id="seed" name="seed">
            <span class="eye" @click="togglevisibility">
              <span :class="{ active: logintype === 'password'}"><i class="far fa-eye"></i></span>
              <span :class="{ active: logintype === 'text'}"><i class="far fa-eye-slash"></i></span>
            </span>
          </div>
            <div v-if="seedtab === false" class="login">
              <input v-model="key" :type="logintype" placeholder="Private key" id="key" name="key">
              <span class="eye" @click="togglevisibility">
                <span :class="{ active: logintype === 'password'}"><i class="far fa-eye"></i></span>
                <span :class="{ active: logintype === 'text'}"><i class="far fa-eye-slash"></i></span>
              </span>
            </div>

          <div v-if="showadvanced === true">
            <label for="seedindex">Seed Index</label>
            <div class="login">
              <input v-model="seedindex" :type="logintype" id="seedindex" name="seedindex">
            </div>
          </div>
          <button @click="openWallet" class="openwallet btn" type="button">Open Wallet</button>
          <scan-qr @scanned="scanDone"></scan-qr>
          <scan-nfc v-if="nfcsup !== false" @scanned="scanDone"></scan-nfc>
        </div>
        <div id="buttons">
          <router-link class="genwallet" :to="$store.getters['app/nodeLink'] + 'generate'">Generate Wallet</router-link>
        </div>
      </div>
      <div id="wallet" class="page" :class="{active: open === true}">
        <div id="powstatus">
            <div class="status busy" :class="{active: ready === false}">Calculating Work <i class="fas fa-spinner fa-spin"></i></div>
            <div class="status ready" :class="{active: ready === true}">Ready <i class="fas fa-check"></i></div>
        </div>
        <div class="inner">
          <div class="block">
            <div class="headingtitle top">
              <span id="closewallet" class="" @click="logout"><i class="fal fa-sign-out fa-flip-horizontal"></i></span>
              <span>Wallet</span>
              <span class="refresh rotate" @click="refresh" :class="{ down: isActive }"><i class="fal fa-sync"></i></span>
              <div class="lastrefresh">Last Refresh: {{ lastrefresh.toLocaleTimeString() }}</div>
            </div>
            <simplebar class="block pending">
              <div id="output">
                <div class="balance">
                  <div @click="copyToClipboard(address)" :class="{ active: balanceextra }" class="raw">{{ address }}</div>
                  <div class="value" v-html="abbreviateNumber(balance)"></div>
                  <div class="raw" :class="{ active: balanceextra }">{{ balance }}</div>
                  <a class="balanceextra" href="" @click.prevent="balanceextra = !balanceextra"><i data-fa-transform="grow-20" class="fal fa-ellipsis-h"></i></a>
                </div>
              </div>
              <div class="headingtitle showmobile">History</div>
              <transaction
                v-for="(transaction, index) in pending"
                :key="index"
                :index="index"
                :transaction="transaction"
                v-on:blockdetails="blockdetails = $event"
                type="pending"
                @receive="refreshDetails"
              ></transaction>

            </simplebar>

          </div>
          <simplebar class="block history">
            <div class="headingtitle hidemobile">History</div>

            <transaction
              v-for="(transaction, index) in history"
              :key="index"
              :index="index"
              :transaction="transaction"
              v-on:blockdetails="blockdetails = $event"
              :type="transaction.type"
            ></transaction>
          </simplebar>
        </div>
        <div id="walletmenu" class="menu">
          <div class="bg"></div>
          <div class="content">
            <div class="tab" data-tab="#receive" @click="receive = true">
              <span class="menuicon"><i data-fa-transform="grow-8" class="fal fa-wallet"></i></span>
              <span>Receive</span>
            </div>
            <div class="tab" data-tab="#send" @click="send = true">
              <span class="menuicon"><i data-fa-transform="grow-8" class="fal fa-coins"></i></span>
              <span>Send</span>
            </div>
            <div class="tab" data-tab="#settings" @click="settings = true">
              <span class="menuicon"><i data-fa-transform="grow-8" class="fal fa-cog"></i></span>
              <span>Settings</span>
            </div>
          </div>
        </div>
      </div>
      <div id="send" class="page" :class="{active: send !== false}">
        <a class="close" v-if="closebutton === true" @click="send = false"><i class="fal fa-times"></i></a>
        <send :open="send" @close="send = false"></send>
      </div>
      <div id="settings" class="page" :class="{active: settings !== false}">
        <a class="close" v-if="closebutton === true" @click="settings = false"><i class="fal fa-times"></i></a>
        <settings
          :open="settings"
          :representative="representative"
          @change="repChange"
        ></settings>
      </div>
      <div id="receive" class="page" :class="{active: receive !== false}">
        <a class="close" v-if="closebutton === true" @click="receive = false"><i class="fal fa-times"></i></a>
        <receive
          :address="address"
        ></receive>
      </div>
      <div id="blockdetails" class="page" :class="{active: blockdetails !== null}">
        <a class="close" v-if="closebutton === true" @click="blockdetails = null"><i class="fal fa-times"></i></a>
        <block-state :details="blockdetails"></block-state>
      </div>

  </div>
</template>

<script>
// @ is an alias to /src
import Transaction from '@/components/Transaction.vue'
import Send from '@/views/Send.vue'
import Receive from '@/views/Receive.vue'
import Settings from '@/views/Settings.vue'
import BlockState from '@/components/BlockState.vue'
import { serverMixin } from '../mixins/serverMixin.js'
import * as webglpow from '../mixins/webgl-pow.js'
import * as NanoCurrency from 'nanocurrency'
import Worker from 'worker-loader!./../mixins/pow.js'
import simplebar from 'simplebar-vue';
import 'simplebar/dist/simplebar.min.css';
import ScanQr from '../components/ScanQr.vue'
import ScanNfc from '../components/ScanNfc.vue'

const hardwareConcurrency = window.navigator.hardwareConcurrency || 2
const workerCount = Math.max(hardwareConcurrency - 1, 1)
let workerList = []

function initialState (){
  return {
    seed: null,
    seedindex: null,
    key: null,
    open: false,
    details: {},
    error: null,
    balance: 0,
    receive: false,
    genwallet: false,
    send: false,
    settings: false,
    representative: '',
    blockdetails: null,
    address: null,
    logintype: 'password',
    walletdata: null,
    isActive: false,
    balanceextra: false,
    terminate: false,
    showadvanced: false,
    seedtab: true,
    pendingpoll: null,
    lastrefresh: new Date(),
    closebutton: true,
    nfcsup: false
  }
}

export default {
  name: 'Home',
  components: {
    Transaction,
    Send,
    Receive,
    Settings,
    BlockState,
    simplebar,
    ScanQr,
    ScanNfc
  },
  mixins: [ serverMixin ],
  data() {
    return initialState();
  },
  watch: {
    open: function (newopen) {
      if(newopen === true && this.key !== null || this.seed !== null) {
        console.log('open')
        this.refreshDetails(true)
      }
    },
    pow: async function (newpow, oldpow) {
      if(this.open === true && newpow !== oldpow && newpow === null) {
        console.log('pow change')
        this.refreshDetails()
      }
    },
    receive: function (state) {
      if (state === true && this.$route.name !== 'POS') {
        const that = this
        let currentpending
        let newpending
        let currentkeys
        let newkeys
        this.pendingpoll = setInterval(async function(){ 
          currentpending = that.pending
          await that.$store.dispatch('app/pending', that.address)
          that.lastrefresh = new Date()
          that.$nextTick(function () {
            newpending = that.pending
            if (JSON.stringify(currentpending) !== JSON.stringify(newpending)) {
              currentkeys = Object.keys(currentpending)
              newkeys = Object.keys(newpending)
              for (const key of newkeys) {
                if(currentkeys.indexOf(key) === -1) {
                  that.$notify({
                    title: 'Funds delivered: ' + NanoCurrency.convert(newpending[key].amount,this.rawconv),
                    text: 'Sent from '+ that.abbreviateAddress(newpending[key].source, false),
                    type: 'success'
                  })
                }
              }
            }
          })
        }, 10000)
      } else {
        clearInterval(this.pendingpoll)
      }
    }
  },
  mounted () {
    if (this.$route.name == 'POS') {
     this.address = this.$route.params.address
     this.receive = true
     this.closebutton = false
    }
    if ("NDEFReader" in window) {
      this.nfcsup = true
    }

    if(this.$store.state.app.node.address === undefined) {
      this.$router.push('node')
    }

  },
  computed: {
    genWalletLink () {
      if('node' in this.$route.params) {
        return this.$route.params.node
      }
      return ''
    },
    loginicon () {
      return (this.logintype === 'password') ? 'fa-eye' : 'fa-eye-slash'
    },
    pow () {
      return this.$store.state.app.pow
    },
    ready () {
      return this.$store.state.app.ready
    },
    history () {
      return this.$store.state.app.history
    },
    pending () {
      return this.$store.state.app.pending
    },
    privatekey () {
      return this.$store.state.app.privatekey
    }
  },
  methods: {
    setSeed () {
      this.seedtab = true
    },
    setPrivate () {
      this.seedtab = false
      this.showadvanced = false
    },
    repChange () {
      this.refreshDetails()
      this.settings = false
    },
    scanDone: function (data) {
      if (data.startsWith('nanokey:')){
        this.seedtab = false
        this.showadvanced = false
        this.key = data.replace('nanokey:','').substr(0, 64)
        this.error = null
      } else if (data.startsWith('nanoseed:')) {
        const seed = data.replace('nanoseed:','').substr(0, 64)
        this.seed = seed
        this.error = null
      } else {
        this.error = 'QR code data does not conform to specification'
        this.seed = data
      }
    },

    /* pasteKey () {
      this.key = this.pasteFromClipboard()
    }, */
    logout () {
      Object.assign(this.$data, initialState());
      this.$store.dispatch('app/resetState')
      for (let workerIndex in workerList) {
        console.log('Terminate: ' + workerIndex)
        workerList[workerIndex].terminate();
      }
    },
    refresh () {
      this.isActive = !this.isActive
      this.refreshDetails()
    },
    async refreshDetails (open = false) {
      let current = null
      if('frontier' in this.details) {
        current = this.details.frontier
      }
      try {
        await this.getDetails(this.privatekey)
        this.lastrefresh = new Date()
        if(current !== this.details.frontier || open === true) {
          this.genWork(this.privatekey, this.details)
        }
        this.$store.dispatch('app/history', this.address)
        this.$store.dispatch('app/pending', this.address)
        if (this.details.balance) {
          this.balance = NanoCurrency.convert(this.details.balance, this.rawconv);
          this.$store.commit('app/balance', this.balance)
        } else {
          this.balance = 0;
        }
        this.representative = this.details.representative;
      } catch(e) {
        this.error = 'Could not connect to RPC'
        this.open = false
      }
    },
    async genWork (key, details){
      this.$store.commit('app/pow', null)
      this.$store.commit('app/ready', false)

      let hash
      if ('frontier' in details){
        console.log('Frontier in details');
        hash = details.frontier
      } else {
        console.log('Frontier NOT in details');
        hash = NanoCurrency.derivePublicKey(key)
      }
      
      const gl = document.createElement('canvas').getContext('webgl2');
      if (gl) {
        console.log('Calculating pow for ' + hash + ' using WebGL this may take some time');
        webglpow.calculate(hash, '0xFFFFFFC0', 2048, 256, (work) => {
            this.$store.commit('app/pow', work)
            this.$store.commit('app/ready', true)
        })
      } else {
        if (window.Worker) {
          console.log('Calculating pow for ' + hash + ' this may take some time');
          const work = () => new Promise(resolve => {
            workerList = []
            for (let i = 0; i < workerCount; i++) {
              const worker = new Worker()
              worker.postMessage({
                blockHash: hash,
                workerIndex: i,
                workerCount: workerCount
              });
              worker.onmessage = (work) => {
                console.log('Work: ' + work.data);
              
                this.$store.commit('app/pow', work.data)
                this.$store.commit('app/ready', true)
                for (let workerIndex in workerList) {
                  console.log('Terminate: ' + workerIndex)
                  workerList[workerIndex].terminate();
                }
                resolve();
              };
              workerList.push(worker);
            }
  
          });
          await work();
        }
        else{
          console.log('Calculating pow for ' + hash + ' (no worker) this may take some time');
          var work = await NanoCurrency.computeWork(hash);
          this.$store.commit('app/pow', work)
          this.$store.commit('app/ready', true)

        }
      }

    },

    togglevisibility () {
      console.log(this.logintype)
      this.logintype = (this.logintype === 'password') ? 'text' : 'password'
    },
    getAddress (key) {
      const publickey = NanoCurrency.derivePublicKey(key)
      return NanoCurrency.deriveAddress(publickey,{useNanoPrefix:true})
    },
    async getDetails (key) {
      this.address = this.getAddress(key)
      this.details = await this.$store.dispatch('app/getDetails', this.address)
    },
    async openWallet () {
      this.error = null
      if(this.key) {
        try {
          const checkKey = NanoCurrency.checkKey(this.key)
          if(checkKey === false) {
            this.error = 'Invalid key'
          } else {
            this.$store.commit('app/privatekey', this.key)
            
            this.open = true
          }

        } catch(e) {
          this.error = e
        }

      } else if (this.seed) {
        try {
          const checkSeed = NanoCurrency.checkSeed(this.seed)
          if(checkSeed === false) {
            this.error = 'Invalid Seed'
          } else {
            if (this.seedindex == null) {
              this.seedindex = 0
            } else {
              this.seedindex = parseInt(this.seedindex)
            }
            this.key = NanoCurrency.deriveSecretKey(this.seed, this.seedindex)
            this.$store.commit('app/privatekey', this.key)
            
            this.open = true
          }

        } catch(e) {
          this.error = e
        }

      }

    }
  }
}
</script>
