<template>
  <div class="col-xs-12 body-mid py-3">
    <loading-overlay :loading="!!unwrapstatus" :text="unwrapstatus" />
    <modal
      v-if="popupModal"
      @modalconfirm="onModalConfirm"
      @modalclose="onModalClose"
      body="Proceed with MetaMask?"
    ></modal>

    <div class="wrap-container-heading">{{ header }}</div>

    <div class="row mb-2">
      <div class="col-xs-12 col-md-6">
        <p>Smart contract platform</p>
      </div>
      <div class="col-xs-12 col-md-6">
        <select
          :class="{ 'row-input-field': true, invalid: !network }"
          v-model="network"
          :disabled="true"
        >
          <option
            v-for="item in activeNetworks"
            :value="item.key"
            :key="item.key"
          >
            {{ item.description }}
          </option>
        </select>
      </div>
    </div>

    <div class="row mb-2">
      <div class="col-xs-12 col-md-6">
        <p>Peercoin Address</p>
      </div>
      <div class="col-xs-12 col-md-6">
        <input
          type="text"
          :class="{ 'row-input-field': true, invalid: !validPPCAddress }"
          v-model="destinationPPCAddress"
          @keypress="onlyForAddress"
        />
      </div>
    </div>

    <div class="row mb-2">
      <div class="col-xs-12 col-md-6">
        <p>Amount</p>
      </div>
      <div class="col-xs-12 col-md-6">
        <input
          type="text"
          :class="{ 'row-input-field': true, invalid: !validAmount }"
          v-model="amount"
          @keypress="onlyForCurrency"
        />
        <p v-if="minAmountNotExceeded" class="text-danger text-end fs-6">
          Minimum amount is set at {{ minAmount }}
        </p>
      </div>
    </div>

    <div class="row my-3">
      <div class="col-xs-12 col-md-8 offset-md-4">
        <div class="d-grid gap-2 d-md-block text-end">
          <button class="btn btn-outline-primary btn-sm foxy" type="button">
            <span class="btn-label">
              <img
                alt="MetaMask"
                height="25"
                src="../assets/metamask-fox.svg" /></span
            >{{ destinationETHAddress }}
          </button>
        </div>
      </div>
    </div>

    <div class="row mb-2" v-if="!callingunwrap">
      <div class="col-xs-12 mt-3">
        <button
          type="button"
          :class="{ btn: true, 'btn-success': true }"
          @click="unwrap"
          :disabled="!validForm"
        >
          Redeem wrapped Peercoin
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from "web3";
import axios from "axios";

import { getNetworks, getContractAddress } from "@/Endpoints.js";
import Modal from "@/components/Modal.vue";
import ABI from "@/abi/erc20.json";
import BaseWrapper from "@/components/BaseWrapper.vue";
import LoadingOverlay from "@/components/LoadingOverlay.vue";

export default {
  extends: BaseWrapper,

  data() {
    return {
      erc20Address: "",
      unwrapstatus: "",
      callingunwrap: false,
    };
  },

  async mounted() {
    this.unwrapstatus = "";
    this.requestId = this.newId();
    this.networks = getNetworks();

    if (!!this.$store.state.network) {
      this.network = this.$store.state.network;
    }

    if (Array.isArray(this.propsaccounts) && this.propsaccounts.length > 0) {
      this.accounts = this.propsaccounts;
    } else {
      this.accounts = await this.getAccounts();
    }

    if (Array.isArray(this.accounts) && this.accounts.length > 0) {
      this.destinationETHAddress = this.accounts[0];
    }
  },

  computed: {
    header() {
      return "Unwrap Peercoin";
    },

    validForm() {
      return (
        this.validAmount &&
        this.validPPCAddress &&
        !!this.network &&
        !this.minAmountNotExceeded
      );
    },
  },

  methods: {
    async onModalConfirm() {
      if (!this.comfirmedProceedMetaMask) {
        this.popupModal = false;
        this.comfirmedProceedMetaMask = true;
        await this.burnTokens();
      }
    },

    enableMetaMaskConfirmationModal() {
      this.popupModal = true;
    },

    async burnTokens() {
      this.unwrapstatus = "Burning tokens on MetaMask";
      this.accounts = await this.getAccounts();

      if (!this.accounts || this.accounts.length < 1) {
        return;
      }

      const contractAddress = getContractAddress(this.session.network);
      if (!contractAddress) {
        return console.error(
          "No contract address found for network: " + this.session.network
        );
      }

      try {
        if (!this.web3) this.web3 = new Web3(ethereum);

        const contract = new this.web3.eth.Contract(ABI, contractAddress, {
          from: this.session.ERC20Address,
        });

        const decimals = await contract.methods.decimals().call();

        const result = await contract.methods
          .burnTokens(
            this.session.amount * 10 ** decimals,
            this.destinationPPCAddress
          )
          .send();

        this.unwrapstatus = "";
        this.resetSession();
        this.gotoHome("Successfully burned " + this.amount + " WPPC");
      } catch (error) {
        this.unwrapstatus = "";
      }
    },

    async unwrap() {
      try {
        this.callingunwrap = true;

        const response = await axios.post(this.endpoints().unwrap, null, {
          headers: {
            "Cache-Control": "no-cache",
            Pragma: "no-cache",
            Expires: "0",
            network: this.network,
            "Idempotency-Key": this.requestId,
          },
          params: {
            amount: this.amount,
            PPCAddress: this.destinationPPCAddress,
            ERC20Address: this.destinationETHAddress,
          },
        });

        if (
          !!response &&
          !!response &&
          !!response.data &&
          !!response.data.data //&& !!response.data.data._id
        ) {
          const success =
            !!response && !!response.data && !!response.data.message;

          this.eventBus.emit("add-toastr", {
            text: success ? response.data.message : `Unable to unwrap`,
            type: success ? "success" : "error",
          });

          this.session = response.data.data;
          // show popup, then burn tokens:
          this.enableMetaMaskConfirmationModal();
        }
      } catch (err) {
        if (
          !!err.response &&
          !!err.response.data &&
          !!err.response.data.message
        )
          console.warn(err.response.data.message);

        this.eventBus.emit("add-toastr", {
          text: `Unable to unwrap`,
          type: "error",
        });
      }
    },
  },

  components: {
    Modal,
    LoadingOverlay,
  },
};
</script>
