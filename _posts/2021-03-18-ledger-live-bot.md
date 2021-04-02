---
layout: post
title: Ledger Live Bot
author: Gaëtan Renaudeau
summary: Introducing a testing framework created last year to automate blockchain testing during recent Ledger Live coin integration scaling.
featured-img: ledgerlive
categories: Tech
---

# Ledger Live Bot

Last year has been a great scaling period for Ledger Live coins and features. We grew from 3 to 9 families of coins. We scaled to new kind of features like [Secure Swap](https://blog.ledger.com/secure-swap/), Staging _(Tezos delegation, Tron votes, Cosmos validation, Algorand staking and very recently, Polkadot)_

- it's not longer scalable to only do manual test. we need to automate.
- context of blockchain. immutable. costly.
  - testnet? => we still need to test end2end.
- quick live schema.
- what do we test here.

## What is Ledger Live Bot?

developed a framework that allows to automate transaction testing on all Ledger Live coins and all features in a very end-to-end way.

**test implicitly a lot of things with a very simple spec.**

=> SCREENSHOT ON A COMMIT TEST REPORT

### Philosophy

- Stateless: My state is on the blockchain.
- Configless: I only need a seed and a coinapps folder.
- Autonomous: I simply restore on existing seed accounts and continue from existing blockchain state.
- Generative: Bot is sending funds to siblings account to create new accounts and rotate funds. The only costs is the network fees.
- Data driven: My engine is simple and I do actions based on data specs that drives my capabilities.
- End to End: I rely on the complete "Ledger stack": live-common which is the same logic behind Ledger Live (derives same accounts, use same transaction logic,..) and Speculos which is the Ledger devices simulator!
- Realistic: I am very close to what is the flow of Ledger Live's users and what they would do with the device, I even press device buttons.
- Completeness: I can technically do everything a user can do on Ledger Live with their account (send but also any feature of Ledger Live like delegation, freeze, rewards,...) but I'm faster at this job 🤖
- Automated: I can run in Github Actions and comments on the Pull Requests and Commits.

## Speculos, a Ledger Hardware wallet simulator

```js
function deviceActionAcceptBitcoin({
  transport,
  event,
}: {
  transport: Transport<*> & { button: (string) => void },
  event: { type: string, text: string },
}) {
  if (event.text.startsWith("Accept")) {
    transport.button("LRlr");
  } else if (
    event.text.startsWith("Review") ||
    event.text.startsWith("Amount") ||
    event.text.startsWith("Address") ||
    event.text.startsWith("Confirm") ||
    event.text.startsWith("Fees")
  ) {
    transport.button("Rr");
  }
}
```

which we reworked later as

```js
const acceptTransaction: DeviceAction<Transaction, *> = deviceActionFlow({
  steps: [
    {
      title: "Amount",
      button: "Rr",
      expectedValue: ({ account, status }) => ...,
    },
    {
      title: "Fees",
      button: "Rr",
      expectedValue: ({ account, status }) => ...,
    },
    {
      title: "Address",
      button: "Rr",
      expectedValue: ...,
    },
    {
      title: "Review",
      button: "Rr",
    },
    {
      title: "Confirm",
      button: "Rr",
    },
    {
      title: "Accept",
      button: "LRlr",
    },
  ],
});
```



## A coin "spec" defines all possibles mutations on the blockchain

```js
const dogecoinSpec: AppSpec<*> = {
  name: "DogeCoin",
  currency: getCryptoCurrencyById("dogecoin"),
  dependency: "Bitcoin",
  appQuery: {
    model: "nanoS",
    appName: "Dogecoin",
    firmware: "1.6.0",
    appVersion: "1.3.x",
  },
  mutations: [
    {
      name: "send max",
      transaction: ({ account, siblings, bridge }) => {
        invariant(account.balance.gt(100000), "balance is too low");
        let t = bridge.createTransaction(account);
        const sibling = pickSiblings(siblings);
        const recipient = sibling.freshAddress;
        t = bridge.updateTransaction(t, { useAllAmount: true, recipient });
        return t;
      },
      deviceAction: deviceActionAcceptBitcoin,
      test: ({
        account,
        accountBeforeTransaction,
        transaction,
        optimisticOperation,
        operation,
      }) => {
        expect(account.balance.toString()).toBe("0");
      },
    },
  ],
};
```

## What's next?

Automating swap?


{% include signatures/gre.html %}
