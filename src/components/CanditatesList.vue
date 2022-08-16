<template>
  <v-row class="py-3">
    <v-col cols="12">
      <v-card class="text-center" outlined>
        <v-overlay absolute :value="connecting">
          <v-progress-circular color="primary" indeterminate opacity="0.5"></v-progress-circular>
        </v-overlay>

        <v-card-text class="title text-center pb-0">Account</v-card-text>

        <template v-if="connected">
          <v-card-text>
            <p class="text-body-2 mb-0">{{ accounts[0] }}</p>
            <p class="text-h6 pb-0">{{ balance }}</p>

            <v-btn block color="primary" @click="disconnect" depressed>Disconnect</v-btn>

          </v-card-text>
        </template>

        <template v-else>
          <v-card-text>
            Connect to Metamask
          </v-card-text>
          <v-card-text>
            <v-btn block color="primary" @click="connectToMetamask" depressed>Connect to Metamask</v-btn>
          </v-card-text>
        </template>
      </v-card>
    </v-col>

    <template v-if="connected">
      <v-col cols="12">
        <v-card outlined>
          <v-overlay absolute :value="candidatesLoading">
            <v-progress-circular color="primary" indeterminate opacity="0.5"></v-progress-circular>
          </v-overlay>
          <v-card-text class="text-center primary white--text">
            <p class="text-h2">{{ candidatesLength }}</p>
            <p class="mb-0">Candidates</p>
          </v-card-text>

          <v-divider></v-divider>

          <template v-for="(candidate, index) in candidates">
            <v-divider :key="`candidate-divider-${index}`" v-if="index > 0"></v-divider>

            <v-list-item link :key="`candidate-${index}`" two-line @click="confirmVote(index)">
              <v-list-item-avatar>
                <v-avatar>
                  <v-img :src="candidate.avatar" />
                </v-avatar>
              </v-list-item-avatar>

              <v-list-item-content>
                <v-list-item-title>{{ candidate.name }}</v-list-item-title>
                <v-list-item-subtitle>Candidate {{ index + 1 }}</v-list-item-subtitle>
              </v-list-item-content>

              <v-list-item-action-text class="text-body-1">
                {{ candidate.vote_count }} votes
              </v-list-item-action-text>
            </v-list-item>

          </template>
        </v-card>
      </v-col>
    </template>

    <v-dialog width="290" persistent v-model="confirmDialog">
      <v-card>
        <v-card-title>Confirm Vote</v-card-title>
        <v-card-text>Are you sure you want to vote this candidate?</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn text color="primary" @click="vote">Vote</v-btn>
          <v-btn text @click="cancelVote">Cancel</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-snackbar v-model="snackbar">
      {{ errorMessage }}

      <template v-slot:action="{ attrs }">
        <v-btn
          color="red"
          text
          v-bind="attrs"
          @click="snackbar = false"
        >
          Close
        </v-btn>
      </template>
    </v-snackbar>
  </v-row>
</template>

<script>
import contractABI from '@/contracts/ballot.json'

import { ethers, utils } from 'ethers'

export default {
  data () {
    return {
      connecting: false,

      providers: null,
      accounts: [],
      rawBalance: 0,
      contract: null,

      candidatesLoading: false,
      candidatesLength: 0,
      candidates: [],
      selectedCandidate: null,

      confirmDialog: false,

      voting: false,

      errorMessage: null,
      snackbar: false
    }
  },

  computed: {
    connected () {
      return this.accounts.length > 0
    },

    balance () {
      return `${utils.formatEther(this.rawBalance)} ETH`
    },
  },

  methods: {
    connectToMetamask: function () {
      this.connecting = true

      this.provider = new ethers.providers.Web3Provider(window.ethereum)

      this.getAccount()
        .then(() => {
          this.getBalance()
        })
        .finally(() => {
          this.connecting = false
        })

      this.createContractInstance()

      this.candidatesLoading = true
      this.getCandidates()
        .then(() => {
          this.candidatesLoading = false
        })
    },

    disconnect: function () {
      this.accounts = []
      this.balance = 0
      this.provider = null
      this.contract = null
      this.candidates = []
    },

    getAccount: async function () {
      this.accounts = await this.provider.send('eth_requestAccounts', [])
    },

    getBalance: async function () {
      this.rawBalance = await this.provider.getBalance(this.accounts[0])
    },

    createContractInstance: function () {
      this.contract = new ethers.Contract('0xE3950690d623a37c8e09B91CE8e6fd5DedD9351D', contractABI)
      this.contract = this.contract.connect(this.provider)
    },

    getCandidates: async function () {
      this.candidatesLength = await this.contract.getCandidatesLength()
      this.candidates = []

      for (let i = 0; i < this.candidatesLength; i++) {
        let _candidate = await this.contract.candidates(i)

        this.candidates.push({
          name: _candidate.name,
          vote_count: _candidate.voteCount,
          avatar: `https://avatars.dicebear.com/api/miniavs/${_candidate.name}-${i}.svg`
        })
      }
    },

    confirmVote: function (index) {
      this.selectedCandidate = index
      this.confirmDialog = true
    },

    cancelVote: function () {
      this.selectedCandidate = null
      this.confirmDialog = false
    },

    vote: async function () {
      let signer = this.provider.getSigner()
      let contractWithSigner = this.contract.connect(signer)

      this.errorMessage = null

      try {
        let transaction = await contractWithSigner.vote(this.selectedCandidate)
        await transaction.wait()
        this.getCandidates()
      } catch (e) {
        this.errorMessage = e?.data?.message

        this.snackbar = true
      } finally {
        this.cancelVote()
      }
    }
  }
}
</script>