<template>
  <div id="app">
    <div v-if="!isLogging && wallet && backup">
      <b-navbar class="hide-print">
        <template slot="brand">
          <b-navbar-item tag="router-link" :to="{ path: '/' }">
            <img src="/logo.png" />
          </b-navbar-item>
        </template>
        <template slot="start">
          <b-navbar-item href="/#/">Dashboard</b-navbar-item>
          <b-navbar-item href="/#/create">Nuovo</b-navbar-item>
          <b-navbar-item href="/#/archive">Archivio</b-navbar-item>
          <b-navbar-item href="/#/search">Ricerca</b-navbar-item>
          <b-navbar-item ><select style="background-color: rgba(21, 24, 36, 0.85);" v-on:change="changeBlockchain()" class="navbar-item" v-model="selected">
          <option v-for="option in options" v-bind:key="option.value">
            {{ option.text }}
          </option>
        </select></b-navbar-item>
        <div v-if="blockchain==='Algorand'" style="top:1vw; position:relative;">{{address}}</div>
        <div v-if="blockchain==='Scrypta'" style="top:1vw; position:relative;">{{address}}</div>
        </template>

        <template slot="end">
          
          <div v-if="blockchain==='Scrypta'" style="top:1vw; position:relative;">{{balance}} LYRA</div>
          
          <div v-if="blockchain==='Algorand'" style="top:1vw; position:relative;">{{balance/1000000}} ALGO</div>
          <b-navbar-item tag="div">
            <div class="buttons">
              <v-gravatar class="gravatar" :email="address" style="margin-right: 10px; height: 80px;max-height: 37px;float: right;margin-top: -8px;border-radius: 4px;"/>
              
              <a v-on:click="logout" class="button login-btn is-primary">
                <strong>Logout</strong>
              </a>
            </div>
          </b-navbar-item>
        </template>
      </b-navbar>
      <router-view />
      <div class="hide-print">
        <hr />Atmosphere Arc DCMS is Based on the Scrypta Digital Contracts
        <a
          href="https://github.com/scryptachain/scrypta-digital-contracts"
          target="_blank"
        >open-source</a> project by
        <a href="https://scrypta.foundation" target="_blank">Scrypta Foundation</a>.
        <br />
        <br />
      </div>
    </div>

    <div v-if="!isLogging && wallet && !backup">
      <section class="hero">
        <div class="hero-body" style="padding: 0;">
          <div class="container" id="backup" style="margin-top:50px;">
            <div class="card">
              <div style="padding: 50px 20px;">
                <h1 class="title is-1">Fai un backup della tua identità</h1>
                <p>Per prevenire la perdità di dati è fondamentale fare un backup sicuro di questo file.</p><br>
                <b-button v-on:click="downloadBackup" type="is-primary" size="is-large">ESEGUI BACKUP</b-button>
                <a id="downloadid" style="display:none"></a>
              </div>
            </div>
          </div>
        </div>
      </section>
    </div>

    <div v-if="!wallet">
      <section class="hero">
        <div class="hero-body" style="padding: 0;">
          <div class="container" id="create" style="margin-top:50px;">
            <div class="card">
              <div style="padding: 50px 20px;">
                <img src="/logo.png"><br>
                <h1 class="title is-1">Atmosphere Arc DCMS</h1>
                <h2 style="color:white;">Seleziona la Blockchain</h2>
                <b-navbar-item ><select style="background-color: rgba(21, 24, 36, 0.85);" v-on:change="changeBlockchain()" class="navbar-item" v-model="selected">
                  <option v-for="option in options" v-bind:key="option.value">
                    {{ option.text }}
                  </option>
                </select></b-navbar-item>
                <h2 class="subtitle">
                  <br />Hai bisogno un'identità Scrypta per entrare in piattaforma.
                  <br />
                  <br />Usa <a href="https://id.scryptachain.org/" target="_blank">Scrypta ID Extension</a>, <a v-on:click="showCreate">crea un nuovo wallet</a> <br><br><b>oppure</b><br><br>accedi con il pulsante sottostante.
                  <br />
                  <br />
                  <div id="scrypta-login" dapp="Digital Contracts"></div>
                </h2>
              </div>
            </div>
            <br />Atmosphere Arc DCMS is based on 
            <a
              href="https://github.com/scryptachain/scrypta-digital-contracts"
              target="_blank"
            >open-source</a> project by
            <a href="https://scrypta.foundation" target="_blank">Scrypta Foundation</a>.
            <br />
            <br />
          </div>
        </div>
      </section>
    </div>

    <b-loading :is-full-page="true" :active.sync="isLogging" :can-cancel="false"></b-loading>
    <b-modal :active.sync="showCreateModal" has-modal-card trap-focus aria-role="dialog" aria-modal>
      <form action>
        <div class="modal-card" style="width: auto">
          <header class="modal-card-head">
            <p class="modal-card-title">Crea identità</p>
          </header>
          <section class="modal-card-body">
            <b-field label="Inserisci una nuova password">
              <b-input
                type="password"
                v-model="password"
                password-reveal
                placeholder="Inserisci la password"
                required
              ></b-input>
            </b-field>

            <b-field v-if="!wallet" label="Ripeti la password">
              <b-input
                type="password"
                v-model="passwordrepeat"
                password-reveal
                placeholder="Ripeti la password"
                required
              ></b-input>
            </b-field>
          </section>
          <footer v-if="!isCreating && !isUpdating" class="modal-card-foot">
            <button
              v-if="!wallet"
              class="button is-primary"
              style="width:100%"
              v-on:click="createUser"
            >CREA</button>
          </footer>
          <footer v-if="isCreating" class="modal-card-foot">
            <div style="text-align:center">Creo l'identità, si prega di attendere...</div>
          </footer>
        </div>
      </form>
    </b-modal>
  </div>
</template>

<script>
let ScryptaCore = require("@scrypta/core");
const algosdk = require('algosdk');

export default {
  data() {
    return {
      scrypta: new ScryptaCore(true),
      algorand: new algosdk.Algod({'x-api-key':"aFxfrQVOwd8VNFWCNftNy26jWNVenx7k8VMkOS9J"}, "https://testnet-algorand.api.purestake.io/ps1", ""),
      address: "",
      wallet: "",
      balance: 0,
      isLogging: true,
      file: [],
      isCreating: false,
      backup: false,
      isUpdating: false,
      showCreateModal: false,
      blockchain: "",
      password: "",
      selected: 'Scrypta',
      options: [
        { text: 'Scrypta', value: 'A' },
        { text: 'Algorand', value: 'B' }
      ],
      passwordrepeat: ""
    };
  },
  async mounted() {
    const app = this;
    if(!localStorage.blockchain)
        localStorage.blockchain="Scrypta"
    app.blockchain=localStorage.blockchain
    if(localStorage.blockchain==="Scrypta"){
      app.scrypta.staticnodes = true
      app.scrypta.testnet=true
      app.wallet = await app.scrypta.importBrowserSID()
      app.wallet = await app.scrypta.returnDefaultIdentity()
      
      if (app.wallet.length > 0) {
        let SIDS = app.wallet.split(":")
        app.address = SIDS[0]
        app.balance = await app.scrypta.get('/balance/' + app.address)
        app.balance=app.balance.balance
        let identity = await app.scrypta.returnIdentity(app.address);
        app.wallet = identity;
        let check_backup = localStorage.getItem('sid_backup')
        if(check_backup !== null && check_backup !== undefined && check_backup === app.address){
          app.backup = true
        }
        app.isLogging = false;
      } else {
        app.isLogging = false;
      }
    }else if(localStorage.blockchain==="Algorand"){
        if(localStorage.algoWallet!=null && localStorage.algoWallet!=undefined && localStorage.algoWallet!=""){
          app.wallet=JSON.parse(localStorage.algoWallet)
          app.address=app.wallet.addr
        }
        if(app.wallet !== null && app.wallet !== undefined && app.wallet!=""){
          let balance=await app.algorand.accountInformation(app.address)
          app.balance=balance.amount
          app.backup = true
        }
        app.isLogging = false;
    }
  },
  methods: {
    loadWalletFromFile() {
      if(localStorage.blockchain==="Scrypta"){
        const app = this;
        app.scrypta.testnet=true
        const file = app.file;
        const reader = new FileReader();
        reader.onload = function() {
          var dataKey = reader.result;

          app.$buefy.dialog.prompt({
            message: `Enter wallet password`,
            inputAttrs: {
              type: "password"
            },
            trapFocus: true,
            onConfirm: async password => {
              let key = await app.scrypta.readKey(password, dataKey);
              if (key !== false) {
                app.scrypta.importPrivateKey(key.prv, password);
                localStorage.setItem("SID", dataKey)
                let exp = dataKey.split(':')
                localStorage.setItem("sid_backup", exp[0])
                location.reload();
              } else {
                app.$buefy.toast.open({
                  message: "Wrong password!",
                  type: "is-danger"
                });
              }
            }
          });
        };
        reader.readAsText(file);
      }else if(localStorage.blockchain==="Algorand"){
        const app = this;
        app.scrypta.testnet=true
        const file = app.file;
        const reader = new FileReader();
        reader.onload = function() {
          var dataKey = reader.result;
          app.$buefy.dialog.prompt({
            message: `Enter wallet password`,
            inputAttrs: {
              type: "password"
            },
            trapFocus: true,
            onConfirm: async password => {
              if (password === localStorage.password) {
                let passphrase=algosdk.secretKeyToMnemonic(dataKey);
                let account = algosdk.mnemonicToSecretKey(passphrase);
                localStorage.wallet=account;
                location.reload();
              } else {
                app.$buefy.toast.open({
                  message: "Wrong password!",
                  type: "is-danger"
                });
              }
            }
          });
        };
        reader.readAsText(file);
      }
    },
    changeBlockchain(){
        const app = this;
        localStorage.setItem("SID", "");
        localStorage.blockchain=""
        localStorage.algoWallet=""
        localStorage.password=""
        localStorage.blockchain=app.selected;
        app.blockchain=app.selected;
        window.location.reload();
    },
    showCreate() {
      const app = this;
      app.showCreateModal = true;
    },
    logout() {
      localStorage.setItem("SID", "");
      localStorage.blockchain=""
      localStorage.algoWallet=""
      localStorage.password=""
      location.reload();
    },
    async createUser() {
      const app = this;
      if(localStorage.blockchain==="Scrypta"){
        if (app.password !== "") {
          if (app.passwordrepeat === app.password) {
            app.isCreating = true;
            setTimeout(async function() {
              let id = await app.scrypta.createAddress(app.password, true);
              let identity = await app.scrypta.returnIdentity(id.pub);
              app.address = id.pub;
              app.wallet = identity;
              localStorage.setItem("SID", id.walletstore);
              app.showCreateModal = false;
              app.password = "";
              app.passwordrepeat = "";
              let tx = await app.scrypta.post("/init", {
                address: id.pub,
                airdrop: true
              })
              console.log(tx)
              if (tx.airdrop_tx === false) {
                app.$buefy.toast.open({
                  message: "Sorry, airdrop was not successful!",
                  type: "is-danger"
                });
              }
              app.isCreating = false;
            }, 500);
          } else {
            app.$buefy.toast.open({
              message: "Passwords doesn't matches.",
              type: "is-danger"
            });
          }
        } else {
          app.$buefy.toast.open({
            message: "Write a password first!",
            type: "is-danger"
          });
        }
      }else if(localStorage.blockchain==="Algorand"){
        if (app.password !== "") {
          if (app.passwordrepeat === app.password) {
            app.isCreating = true;
            setTimeout(async function() {
              let id = await algosdk.generateAccount();
              console.log(id);
              app.address = id.addr;
              app.wallet = id;
              localStorage.algoWallet=JSON.stringify(id);
              localStorage.password=app.password;
              app.showCreateModal = false;
              app.password = "";
              app.passwordrepeat = "";
              app.isCreating = false;
            }, 500);
          } else {
            app.$buefy.toast.open({
              message: "Passwords doesn't matches.",
              type: "is-danger"
            });
          }
        } else {
          app.$buefy.toast.open({
            message: "Write a password first!",
            type: "is-danger"
          });
        }
      }
    },
    downloadBackup(){
      const app = this
      if(localStorage.blockchain==="Scrypta"){
        app.$buefy.dialog.prompt({
          message: `Enter wallet password`,
          inputAttrs: {
            type: "password"
          },
          trapFocus: true,
          onConfirm: async password => {
            let SID = localStorage.getItem('SID')
            let key = await app.scrypta.readKey(password, SID);
            if (key !== false) {
              var a = document.getElementById("downloadid");
              app.backup = true
              localStorage.setItem('sid_backup', app.address)
              var file = new Blob(
                [SID],
                { type: "sid" }
              );
              a.href = URL.createObjectURL(file);
              a.download = app.address + ".sid";
              var clickEvent = new MouseEvent("click", {
                view: window,
                bubbles: true,
                cancelable: false
              });
              a.dispatchEvent(clickEvent);
            }else{
              app.$buefy.toast.open({
                message: "Wrong password!",
                type: "is-danger"
              });
            }
          }
        })
      }else if(localStorage.blockchain==="Algorand"){
        app.$buefy.dialog.prompt({
          message: `Enter wallet password`,
          inputAttrs: {
            type: "password"
          },
          trapFocus: true,
          onConfirm: async password => {
            let key = JSON.parse(localStorage.algoWallet).sk
            if (password === localStorage.password) {
              var a = document.getElementById("downloadid");
              app.backup = true
              localStorage.setItem('sid_backup', app.address)
              var file = new Blob(
                [JSON.stringify(key)],
                { type: "sid" }
              );
              a.href = URL.createObjectURL(file);
              a.download = app.address + ".sid";
              var clickEvent = new MouseEvent("click", {
                view: window,
                bubbles: true,
                cancelable: false
              });
              a.dispatchEvent(clickEvent);
            }else{
              app.$buefy.toast.open({
                message: "Wrong password!",
                type: "is-danger"
              });
            }
          }
        })
      }
    }
  }
};
</script>

<style>
  .show-print{
    display: none;
  }
  @media print {
    .hide-print{
      display:none!important;
    }
    .show-print{
      display:block!important;
    }
    .no-cut{
        page-break-inside: avoid;
    }
  }
  #app {
    font-family: "Sen";
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
  }
  #nav {
    padding: 30px;
  }

  #nav a {
    font-weight: bold;
    color: #2c3e50;
  }

  #nav a.router-link-exact-active {
     color: #2c3e50;
  }
  .container{
    padding:0 15px
  }
  @media screen and (max-width: 1024px){
    .login-btn{
      width: 100%!important;
    }
    .gravatar{
      display:none;
    }
    .card-footer-item{
      font-size:12px!important;
    }
  }
</style>