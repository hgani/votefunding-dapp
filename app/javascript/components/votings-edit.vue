<template lang='pug'>
  div
    h1.mb-3 Edit Voting
    .card
      .card-header
        h4 
          router-link(:to="`/votings/${voting.address}`" class="d-block") {{voting.label}}    
      .card-body
        p
          strong Address:
          i
            a(:href='voting.addressUrl') {{voting.address}}
          br
          strong Creator:
          i
            a(:href='voting.creatorUrl') {{voting.creator}}
          br
          strong Tx (deployed):
          i
            a(:href='voting.txUrl') {{voting.tx_hash}}
        voting-form(      
          :web3="web3" 
          :web3-helper="web3Helper" 
          :voting-contract="votingContract" 
          :csrf-token="csrfToken"
          :read-only-voting="voting"
          :submitting="submitting"
          :edit-mode="true"
          @submit="submit"
        )
</template>

<script>
export default {
  props: ["web3", "web3Helper", "votingContract", "csrfToken"],
  data() {
    return {
      voting: { options: [] },
      submitting: false
    };
  },
  created() {
    const self = this;

    const { web3, web3Helper, votingContract } = self;
    const Contract = web3.eth.contract(votingContract.ABI);
    const route = self.$router.currentRoute;

    const address = route.params.address;

    $.ajax({
      url: `votings/${address}.json`,
      success(data) {
        const voting = data.voting;

        if (!voting) {
          return self.$router.push("/not-found");
        }

        voting.options = JSON.parse(voting.options);
        voting.addressUrl = `${web3Helper.viewAddressPath}/${voting.address}`;
        voting.creatorUrl = `${web3Helper.viewAddressPath}/${voting.creator}`;
        voting.txUrl = `${web3Helper.viewTxPath}/${voting.tx_hash}`;
        voting.currentUserCreator =
          web3.eth.defaultAccount &&
          web3.eth.defaultAccount.toLowerCase() ===
            voting.creator.toLowerCase();

        self.voting = voting;
      },
      error() {}
    });
  },
  methods: {
    submit(voting) {
      const self = this;

      const { web3, web3Helper, votingContract, csrfToken } = self;

      const ttrOption = {
        closeButton: true,
        positionClass: "toast-bottom-full-width",
        timeOut: 0,
        extendedTimeOut: 0,
        tapToDismiss: false
      };

      const creator = web3.eth.defaultAccount;

      if (!self.web3Helper.metamaskLogin()) {
        return alert("Please login to MetaMask");
      }

      if (!voting.label) return ttr.error("Please enter valid label");
      for (const item of voting.options) {
        if (!item) return ttr.error("Please enter valid option");
      }

      self.submitting = true;
      $.ajax({
        url: `/votings/${voting.address}.json`,
        method: "post",
        data: {
          _method: "patch",
          label: voting.label,
          description: voting.description,
          options: JSON.stringify(voting.options),
          authenticity_token: csrfToken
        },
        success() {
          self.submitting = false;
          ttr.success("Your contract has been saved", null, ttrOption);
        },
        error() {
          self.submitting = false;
          ttr.error("Error when saving contract", null, ttrOption);
        }
      });
    }
  }
};
</script>