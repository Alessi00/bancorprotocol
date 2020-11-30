<template>
  <b-modal
    :content-class="darkMode ? 'bg-block-dark' : 'bg-block-light'"
    id="vote-details"
    scrollable
    size="xl"
    centered
    v-model="show"
    @close="onHide"
    @cancel="onHide"
    @hide="onHide"
    @show="update"
  >
    <template slot="modal-header">
      <div class="w-100">
        <b-row>
          <b-col cols="12" class="d-flex justify-content-between">
            <span
              class="font-size-14 font-w600"
              :class="darkMode ? 'text-dark' : 'text-light'"
            >
              <span>Voting Statistics</span>
            </span>
            <font-awesome-icon
              class="cursor font-size-lg"
              :class="darkMode ? 'text-dark' : 'text-light'"
              @click="onHide"
              icon="times"
            />
          </b-col>
        </b-row>
      </div>
    </template>

    <div class="w-100 p-3">
      <span
        class="text-uppercase font-size-12 font-w600"
        :class="darkMode ? 'text-muted-dark' : 'text-muted-light'"
      >
        Proposal Title:
      </span>
      <span
        class="font-size-14 font-w500"
        :class="darkMode ? 'text-dark' : 'proposal-title-light'"
      >
        {{ proposal.name }}
      </span>
    </div>

    <data-table
      :items="getVoters()"
      :fields="fields"
      class="p-0"
      hide-pagination="true"
      per-page="100000000"
    >
      <template #cell(account)="data">
        <a
          :href="getEtherscanUrl(data.item.account)"
          target="_blank"
          rel="noopener"
          class="font-size-14"
          >{{ data.item.account }}
        </a>
      </template>

      <template #cell(weight)="data">
        <div class="text-right">
          {{ getWeight(data.item.votes) }}
          {{ symbol }}
        </div>
      </template>

      <template #cell(voted)="data">
        <div class="text-uppercase" :class="getVoteClass(data.value)">
          {{ data.value === 1 ? "FOR" : "AGAINST" }}
        </div>
      </template>

      <template #cell(percentOfTotal)="data">
        <div class="text-right">{{ data.item.percentOfTotal }}%</div>
      </template>
    </data-table>

    <template slot="modal-footer">
      <div class="text-center w-100">
        <main-button
          @click="onHide"
          label="Close"
          :active="true"
          :block="false"
          style="width: 175px"
        />
      </div>
    </template>
  </b-modal>
</template>

<script lang="ts">
import { vxm } from "@/store/";
import { Component, Vue, Prop, Emit, VModel } from "vue-property-decorator";
import { prettifyNumber } from "@/api/helpers";
import MainButton from "@/components/common/Button.vue";
import {
  Proposal,
  Voter,
  Votes
} from "@/store/modules/governance/ethGovernance";
import BigNumber from "bignumber.js";
import DataTable, { ViewTableField } from "@/components/common/DataTable.vue";
import { shrinkToken } from "@/api/eth/helpers";

@Component({
  components: {
    MainButton,
    DataTable
  }
})
export default class ModalVoteDetails extends Vue {
  @VModel({ type: Boolean }) show!: boolean;
  @Prop() proposal!: Proposal;

  symbol: string = "";
  decimals: number = 0;
  etherscanUrl: string = "";

  fields: ViewTableField[] = [
    {
      id: 1,
      key: "index",
      label: "#",
      sortable: true
    },
    {
      id: 2,
      key: "account",
      label: "User Wallet",
      sortable: true
    },
    {
      id: 3,
      key: "weight",
      label: "Amount",
      sortable: true,
      thClass: "text-right"
    },
    {
      id: 4,
      key: "voted",
      label: "Vote",
      sortable: true
    },
    {
      id: 5,
      key: "percentOfTotal",
      label: "% of Total",
      sortable: true,
      thClass: "text-right"
    }
  ];

  getWeight(votes: Votes): string {
    return this.formatNumber(votes.for !== "0" ? votes.for : votes.against);
  }

  getVoteClass(vote: number) {
    return vote === 1 ? "voted-for" : "voted-against";
  }

  getVoters() {
    return this.proposal.voters.map((v: Voter, index: number) => {
      return {
        index: index + 1,
        ...v,
        percentOfTotal: this.calculatePercentOfTotal(
          v.votes.for !== "0" ? v.votes.for : v.votes.against
        ),
        voted: v.votes.voted === "for" ? 1 : 2,
        weight: v.votes.for !== "0" ? v.votes.for : v.votes.against
      };
    });
  }

  calculatePercentOfTotal(vote: string) {
    const percent = new BigNumber(100)
      .dividedBy(new BigNumber(this.proposal.totalVotes))
      .multipliedBy(
        new BigNumber(vote).dividedBy(new BigNumber(10).pow(this.decimals))
      );

    return percent.toFixed(2);
  }

  formatNumber(num: string) {
    return prettifyNumber(shrinkToken(num, this.decimals));
  }

  getEtherscanUrl(account: string) {
    return `${this.etherscanUrl}address/${account}`;
  }

  get darkMode(): boolean {
    return vxm.general.darkMode;
  }

  @Emit("hide")
  onHide() {
    console.log("fa");
    this.show = false;
  }

  async update() {}

  async mounted() {
    this.symbol = await vxm.ethGovernance.getSymbol();
    this.decimals = await vxm.ethGovernance.getDecimals();
    this.etherscanUrl = await vxm.ethGovernance.getEtherscanUrl();
  }
}
</script>
<style lang="scss">
@import "@/assets/_scss/custom/_variables";

#vote-details .voted-for {
  color: #3ec8c8;
}

#vote-details .voted-against {
  color: #de4a5c;
}

#vote-details .modal-header {
  border-bottom: 1px solid #e6ebf2;
}

#vote-details .proposal-title-light {
  color: #0a2540;
}

#vote-details .modal-body {
  padding-right: 0 !important;
  padding-left: 0 !important;
}

#vote-details a {
  color: #0f59d1;
}
</style>