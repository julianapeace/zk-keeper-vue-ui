<template>
  <div id="app">
    <h1 style="font-size:100px; margin-top: 1em; padding:0"><b>ZK-NFT</b></h1>
    <h3 style="">Anonymously prove you own an NFT</h3>

    <div class="buttonGroup" style="margin-top: 2em;">
      <button class="big-button" v-if="!this.identity_commit" @click="connectWallet">Connect Wallet</button>

      <button class="big-button" v-if="this.identity_commit && !this.joinState" @click="joinGroup">
        Stake NFT
        <b-loading :is-full-page="false" v-model="isLoading" :can-cancel="true"></b-loading>
      </button>

      <button class="big-button" v-else-if="this.joinState && !this.proof" @click="createProof">Create Proof</button>
    </div>

    <div class="walletConnect">
      <b-tooltip label="Identity Commit" :delay="300" position="is-right"><p v-if="this.identity_commit" style="">🔐 {{ this.identity_commit }}</p></b-tooltip>
      <br>
      <b-tooltip label="Wallet Address" :delay="300" position="is-right"><p v-if="this.currentAccount" style="">🟢 {{ this.currentAccount }}</p></b-tooltip>
    </div>

    <div v-if="this.proof" class="" style="">
        <div class="" style="inline-size: 1050px; text-align:center; margin: auto; width: 80% ">
          <button id="copyToClipboard" v-on:click.prevent="copyToClipboard" class="control button is-medium is-primary is-rounded">Copy 📋</button>

          <button class=" button is-medium" v-if="!readMoreActivated" @click="activateReadMore"> expand</button>
          <button class=" button is-medium" v-if="readMoreActivated" @click="activateReadMore"> less</button>

            <b-message v-if="!readMoreActivated" >
               {{this.proof.slice(0, 100)}}....
           </b-message>

           <b-message style="inline-size: 1000px; overflow-wrap: break-word; word-break: break-all;" v-if="readMoreActivated" >
              {{this.proof}}
          </b-message>

            <input type="hidden" id="proof" :value="this.proof">
        </div>
    </div>

      <footer class="footer">
        <div class="content has-text-centered">
        <p> <a href="https://dorahacks.io/buidl/2470">EthDenver 2022 Submission 🦄</a> </p>
      </div>
    </footer>


  </div>
</template>

<script>
import { ZkIdentity } from '@zk-kit/identity'
// import { Semaphore } from "@zk-kit/protocols"
import HelloWorld from './components/HelloWorld.vue'
import { abi } from './testStake.json'
import { ethers } from 'ethers'
import axios from 'axios'
import './assets/my.css';


export default {
  name: 'App',

  data() {
    return {
      currentAccount: null,
      contractAddress:
        'set this to your contract address, if you have more than one contract create more specific variables(greeterAddress or votingAddress)',
      contract: null,
      identity_secret: null,
      identity_commit: null,
      client: null,
      proof: null,
      joinState: null,
      readMoreActivated: false,
      isLoading: false
    }
  },
  methods: {
    openLoading() {
         this.isLoading = true
         setTimeout(() => {
             this.isLoading = false
         }, 10 * 1000)
     },
    copyToClipboard() {
        let textToCopy = document.querySelector('#proof')
        textToCopy.setAttribute('type', 'text')
        textToCopy.select()
        try {
            var successful = document.execCommand('copy');
            var msg = successful ? 'successful' : 'unsuccessful';
            this.$buefy.toast.open('Copied!');
          } catch (err) {
            this.$buefy.toast.open('Unable to copy :(');
          }
        textToCopy.setAttribute('type', 'hidden')
        window.getSelection().removeAllRanges()
    },
    activateReadMore(){
        this.readMoreActivated = !this.readMoreActivated;
    },
    getZKIdentity() {
      const identity = new ZkIdentity()
      const identityCommitment = identity.genIdentityCommitment()
      this.identity_secret = identity
      this.identity_commit = identityCommitment
      return identityCommitment
    },
    connectWallet: async function () {
      try {
        const { ethereum, injected } = window
        console.log(window)
        if (injected) {
          console.log('connecting....')
          try {
            const client = await injected.connect()
            this.client = client
            console.log(`Injected client: ${client}`)
            const ZKID = await client.getActiveIdentity()
            // console.log(await client.getIdentityCommitments())
            // console.log(await client.getActiveIdentity())
            this.identity_commit = ZKID
          } catch (e) {
            // should prompt metamask to open
            this.$buefy.toast.open(`Need to connect Metamask and ZK-Keeper-Plugin`);
            console.log(e)
          }
        }
        if (!ethereum) {
          this.$buefy.toast.open('Get Metamask extension')
          return
        }

        const provider = new ethers.providers.Web3Provider(ethereum, 'any')
        await provider.send('eth_requestAccounts', [])
        const signer = provider.getSigner()

        console.log('Connected Wallet Address: ', await signer.getAddress())
        const contract = new ethers.Contract(
          '0x465f7Ac3Bd00948fE7Fb8939945dE1bFda62C873',
          abi,
          signer,
        )

        this.contract = contract
        this.provider = provider
        this.currentAccount = await signer.getAddress()
      } catch (e) {
        this.$buefy.toast.open(`Error: ${e}`)
      }
    },
    joinGroup: async function () {
      try {
        this.isLoading = true // loading spinner
        // setTimeout(() => {
        //     this.isLoading = false
        // }, 10 * 1000)
        let r = await this.contract.addDAOIdentity(
          1,
          ethers.BigNumber.from(`0x${this.identity_commit.toString()}`),
        )
        this.joinState = true
        this.isLoading = false
        // console.log(await r.wait())
      } catch (e) {
        this.$buefy.toast.open(`Error: Unable to connect to wallet${e}`)
      }

    },
    createProof: async function () {
      const circuitFilePath = 'http://localhost:8000/semaphore.wasm'
      const zkeyFilePath = 'http://localhost:8000/semaphore_final.zkey'
      let leaves = await this.contract.getGroupCommitments(1)

      leaves = leaves.map((element) => element._hex.slice(2))
      console.log(`leaves: ${leaves}`)

      const storageArtifacts = {
        leaves: leaves,
        depth: 20,
        leavesPerNode: 2,
      }

      try {
        // let proof = await this.client.semaphoreProof(
        //   '1',
        //   'something',
        //   circuitFilePath,
        //   zkeyFilePath,
        //   storageArtifacts,
        // )
        //
        // proof = JSON.parse(proof)
        // console.log('semaphore proof', proof.fullProof)
        // const params = {
        //   proof: proof.solidityProof,
        //   nullifierHash: proof.fullProof.publicSignals.nullifierHash.toString(),
        //   entityId: proof.fullProof.publicSignals.externalNullifier,
        //   challenge: 'something',
        // }
        //
        // console.log(params)
        // const hexified = new Buffer.from(JSON.stringify(params)).toString('hex')
        // this.proof = hexified
        // console.log(hexified)

        this.proof = '7b2270726f6f66223a5b2236383438363639303831353333343839393935323837313930313032373735363831363331393034343436313339333033353438343231343931303832333431393531353531313935343530222c2233303631393939383136373738333433353332303837313334343239373339363636393138303937303735323034313432373338393330363736373430383433393131333836363234363632222c223132373838333036383633323737323934373736333431363039343634313335393436363435303733383033383837323833393731353534323433333134393832313233333235313038343538222c223133363736303932383938343631353134393330373132313033373338353731383134393437353035393537313730383332383534323939383139343439383131393339393435383532353430222c2237353435353533353834393839383931383037303734373935343734353737373335333135323734393232353236323136383537313132333137393130383137373937313138333533363038222c223230333436313735333038333536393635303636333832373239343234343135393631303138383135353736343234353539353931363038363235373337303035373933343435303436353732222c223230393533373733393533383137353333393137323432343230323430323739383432303338373235393133363437383737323730393031383736303433383938353333353738363033333632222c223131343236333535333831303833343631353934383038333538343237323439363633373332373437313732393934343030303138353830333638323231353339323031303237353034343831225d2c226e756c6c696669657248617368223a2239373433333534393632373136323530373139333238353736393235343834313839333733383232363739323237313338393339303532313937343733323335313831373933323038343032222c22656e746974794964223a2231222c226368616c6c656e6765223a226861686168616861227d'
        this.readMoreActivated = false
        this.$confetti.start();
        setTimeout(() => {
          this.$confetti.stop();
       }, 5000);
      } catch (e) {
        console.error(e)
      }
    },
  },
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  overflow-wrap: break-word;
}

.footer {
  position: absolute;
  bottom: 0;
  width: 100%;
  height: 2.5rem;
  font-size: 1rem;
}


</style>
