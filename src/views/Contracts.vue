<template>
  <div class="home">
    <div class="main text-left">
      <div class="container">
        <h1>Dashboard</h1>
        <hr>
        <div v-if="contracts.length > 0 && blockchain=='Scrypta'">
          <div v-for="contract in contracts" v-bind:key="contract.data.hash">
            <a :href="'/#/contract/' + contract.address">
              <div style="border:1px solid #ccc; text-align:left; position:relative; color:#000; border-radius:5px; margin-top:20px; font-size:12px; padding:15px; color:white;">
                <v-gravatar :email="contract.address" style="float:left; height:55px; margin-right:10px;" />
                <strong v-if="contract.data.title">{{ contract.data.title }}</strong>
                <strong v-if="!contract.data.title">Nessun titolo</strong>
                <br><strong>Indirizzo:</strong> {{ contract.address }}
                <br><strong>Partecipanti:</strong> {{ contract.data.subjects.length }}
                <b-icon
                    style="position:absolute; top:27px; right:30px"
                    icon="arrow-right"
                    size="is-medium">
                </b-icon>
              </div>
            </a>
          </div>
        </div>
        <div v-if="contracts.length > 0 && blockchain=='Algorand'">
          <div v-for="contract in contracts" v-bind:key="contract.hash">
            <a :href="'/#/contract/' + contract.address">
              <div style="border:1px solid #ccc; text-align:left; position:relative; color:#000; border-radius:5px; margin-top:20px; font-size:12px; padding:15px; color:white;">
                <v-gravatar :email="contract.address" style="float:left; height:55px; margin-right:10px;" />
                <strong v-if="contract.title">{{ contract.title }}</strong>
                <strong v-if="!contract.title">Nessun titolo</strong>
                <br><strong>Indirizzo:</strong> {{ contract.address }}
                <br><strong>Partecipanti:</strong> {{ contract.subjects.length }}
                <b-icon
                    style="position:absolute; top:27px; right:30px"
                    icon="arrow-right"
                    size="is-medium">
                </b-icon>
              </div>
            </a>
          </div>
          
        </div>
        <div v-if="isLoading === true">
          Cerco i tuoi contratti in blockchain...
        </div>
        <div v-if="contracts.length === 0 && isLoading === false">
          Non hai ancora creato nessun contratto digitale!<br>
          Grazie al digital contract è possibile realizzare contratti registrati all'interno della blockchain di Scrypta. Questi contratti sono firmati da indirizzi di persone fisiche o giuridiche riconosciute dalle parti e la cui identità può essere verificata grazie a <a href="https://scrypta.id" target="_blank">https://scrypta.id</a>.<br><br>
          <a href="/#/create">
          <b-button type="is-primary">CREA IL TUO PRIMO CONTRATTO</b-button>
          </a>
        </div>
      </div>
    </div>
    <b-loading :is-full-page="true" :active.sync="isLoading" :can-cancel="true"></b-loading>
  </div>
</template>

<script>
  const ScryptaCore = require('@scrypta/core')
  const LZUTF8 = require('lzutf8')
  const algosdk = require('algosdk')
  const request=require('request')

  export default {
    name: 'Home',
    data(){ 
      return {
        scrypta: new ScryptaCore(true),
        address: '',
        blockchain: '',
        wallet: '',
        isLoading: true,
        algorand: new algosdk.Algod({'x-api-key':"aFxfrQVOwd8VNFWCNftNy26jWNVenx7k8VMkOS9J"}, "https://testnet-algorand.api.purestake.io/ps1", ""),
        contracts: []
      }
    },
    async mounted() {
      const app = this
      app.blockchain=localStorage.blockchain
      if(localStorage.blockchain==="Scrypta"){
        app.wallet = await app.scrypta.returnDefaultIdentity()
        let SIDS = app.wallet.split(':')
        app.address = SIDS[0]
        app.scrypta.staticnodes = true
        app.scrypta.testnet = true
        let identity = await app.scrypta.returnIdentity(app.address)
        app.wallet = identity
        let contracts = await app.scrypta.post('/read', { protocol: 'DC://' })
        for(let k in contracts.data){
          let contract = contracts.data[k]
          let subjects = []
          for(let j in contract.data.subjects){
            subjects.push(contract.data.subjects[j].address)
          }
          if(contract.data.v !== undefined && contract.data.v === 1){
            if(contract.data.creator === app.address || subjects.indexOf(app.address) !== -1){
              if(contract.data.title !== undefined && contract.data.title !== ''){
                contract.data.title = LZUTF8.decompress(contract.data.title, { inputEncoding: 'Base64' });
              }
              app.contracts.push(contract)
            }
          }
        }
        app.isLogging = false
        app.isLoading = false
      }else if(localStorage.blockchain==="Algorand"){
        app.contracts=[]
        app.wallet=JSON.parse(localStorage.algoWallet)
        app.address=app.wallet.addr
        let transactions=await app.algorand.transactionByAddress(app.address)
        for(let i=0;i<transactions.transactions.length; i++){
          if(transactions.transactions[i].from!=transactions.transactions[i].payment.to){
            let contract=transactions.transactions[i].payment.to
            let contractTrans=await app.algorand.transactionByAddress(contract)
            console.log(contractTrans)
            for(let j=0;j<contractTrans.transactions.length;j++){
              if(contractTrans.transactions[j].note!=undefined && contractTrans.transactions[j].from==contractTrans.transactions[j].payment.to){
                console.log(algosdk.decodeObj(contractTrans.transactions[j].note))
                request(' https://ipfs.io/ipfs/'+algosdk.decodeObj(contractTrans.transactions[j].note), function (error, response, body) {
                  let contr=JSON.parse(body)
                  console.log(contr)
                  app.contracts.push(contr)
                });
              }
            }
          }
        }
        app.isLogging = false
        app.isLoading = false
      }
    }
  }
</script>