<template lang='pug'>
  div
    voting(
      :voting="voting"
    )
</template>

<script>
import Vue from "vue/dist/vue.esm";
import votingContract from "../voting-contract";
import store from "../store";

const ttrOption = {
  closeButton: true,
  positionClass: "toast-bottom-full-width",
  timeOut: 0,
  extendedTimeOut: 0,
  tapToDismiss: false
};

export default {
  props: ["web3", "web3Helper", "votingContract"],
  data() {
    return {
      store,
      voting: {},
      submitting: false
    };
  },
  created() {
    const self = this;

    const Contract = web3.eth.contract(votingContract.ABI);
    const route = self.$router.currentRoute;
    const address = route.params.address;

    web3Helper.getNetwork((err, net) => {
      if (err) throw err;

      $.ajax({
        url: `votings/${address}.json`,
        data: { network: net },
        success(data) {
          const voting = data.voting;

          if (!voting) {
            return self.$router.push("/not-found");
          }

          voting.balance = null;
          voting.minimumFund = null;
          voting.endBlock = null;
          voting.options = JSON.parse(voting.options);
          voting.optionVotes = [];
          voting.optionApproves = [];
          voting.optionFunds = [];
          voting.currentUserCreator =
            web3.eth.defaultAccount &&
            web3.eth.defaultAccount.toLowerCase() ===
              voting.creator.toLowerCase();

          const contract = Contract.at(voting.address);

          (function(voting, contract) {
            web3.eth.getBalance(voting.address, (err, data) => {
              if (err) throw err;

              voting.balance = parseInt(data);
            });

            contract.minimumFund.call((err, data) => {
              if (err) throw err;

              voting.minimumFund = parseInt(data);
            });

            contract.endBlock.call((err, data) => {
              if (err) throw err;

              voting.endBlock = parseInt(data);
            });

            for (const key in voting.options) {
              (function(key) {
                // Set default value
                voting.optionFunds.splice(key, 1, 0);

                contract.approves.call(key, (err, data) => {
                  if (err) throw err;

                  voting.optionApproves.splice(key, 1, parseInt(data));
                });

                contract.funds.call(key, (err, data) => {
                  if (err) throw err;

                  voting.optionFunds.splice(key, 1, parseInt(data));
                });
              })(key);
            }
          })(voting, contract); // save params inside closure

          self.voting = voting;
          self.getOptionVotes();
        },
        error() {}
      });
    });
  },
  methods: {
    getOptionVotes() {
      const self = this;

      const Contract = web3.eth.contract(votingContract.ABI);
      const contract = Contract.at(self.voting.address);

      // for each voting options
      for (const index in self.voting.options) {
        (function(index) {
          // get votes of option
          contract.getVotes.call(index, 0, 1000, (err, data) => {
            const votes = [];
            let isCurrentUser = false;

            votes.currentUserFund = 0;
            votes.currentUserCancel = false;
            votes.validCount = 0;

            // for each votes
            for (let i = 0; i < data[0].length; i++) {
              const value = {};

              value.voter = data[0][i];
              value.fund = parseInt(data[1][i]);
              value.isCancel = data[2][i];

              isCurrentUser =
                web3.eth.defaultAccount &&
                web3.eth.defaultAccount.toLowerCase() ===
                  value.voter.toLowerCase();

              if (isCurrentUser) {
                votes.currentUserFund = value.fund;
                votes.currentUserCancel = value.isCancel;
              }

              if (value.fund && !value.isCancel) {
                votes.validCount++;
              }
              
              votes.push(value);
            }

            Vue.set(self.voting.optionVotes, index, votes);
          });
        })(index);
      }
    }
  }
};
</script>
