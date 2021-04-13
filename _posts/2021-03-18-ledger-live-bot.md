---
layout: post
title: Ledger Live Bot
author: Gaëtan Renaudeau & Valentin De Almeida
summary: Introducing a testing framework created last year to automate blockchain testing during recent Ledger Live coin integration scaling.
featured-img: ledgerlive
categories: Tech
---

# Ledger Live Bot

Last year has been a great scaling period for the Ledger Live software.
We went from 3 to 9 families of coin supported, shipped features like[Secure Swap](https://blog.ledger.com/secure-swap/), Staking _(Tezos delegation, Tron votes, Cosmos validation, Algorand staking and very recently, Polkadot)_ ...  
As the list of features and new coins are supported in Ledger Live, we quickly realized that our testing flow would not scale. Previously, our QA Team would need to test all the different features for each coin manually. With both increasing in number, the process was becoming longer and more tedious. That is when we decided to tackle this problem with a new approach: __automate end-to-end testing for each family of coin alongside its respective features__!

Let's rewind a bit and see the context here. We are talking about end-to-end testing on different blockchains. Blockchains are _immutable_ by design. Once an operation has been broadcasted, there is no way to come back to a _previous_ state of the blockchain. This mean we would not be able to replay any test case or scenario.  
At some point, we thought about _testnet blockchains_ but it still might not yield the same result as a _mainnet_. 

So, with the context in mind and the expectations of real life conditions, that's when we decided to create the **Ledger Live Bot**.

## What is Ledger Live Bot ?

The Ledger Live Bot is a _framework_ we build internally to allow the automation of transaction testing on all Ledger Live supported coins and feature, in a very end-to-end approach.

**The Ledger Live Bot test implicitly a lot of things with a very simple spec file.**

=> INSERT SCREENSHOT ON A COMMIT TEST REPORT

### Philosophy

When we built the bot, we setup a few principles that would help us make it into the tool we needed.

- **Stateless**: My state _is_ the blockchain.
- **Configless**: I only need a _seed_ and a _coinapps_ folder.
- **Autonomous**: I simply restore my accounts using the seeds and continue from here
- **Generative**: I send funds to sibling accounts to create new accounts and rotate funds. My only costs are network fees.
- **Data Driven**: My engine is simple and I do actions based on data specs that drive my capabilities.
- **End to End**: I rely on the complete "Ledger stack"
  - live-common: the library behind Ledger Live logic (deriving accounts, transaction logic...) (open source)
  - Speculos: the Ledger devices simulator(open source)
  - coinapps: a folder containing the apps needed by Speculos (closed source)
- **Realistic**: I am very close to the flow used by Ledger Live users and what they do with their device. I can even press the devices' buttons.
- **Completeness**: I can _technically_ do anything user can do Ledger Live with their account (send, but also any feature from Leger Live like delegations, freeze, staking...), but I do it faster 🤖
- **Automated**: I can run on Github Actions (runners) and comment on the Pull Requests and Commits.

## How to automate device testing with Speculos, a Ledger Hardwarde wallet simulator

One of our main bottleneck is obviously the use of real, physical devices to go through the all the different flows and transactions.  
To do this, we rely on a technology developed at Ledger and released in 2019: [Speculos](https://speculos.ledger.com/).

Now equipped with a "software" version of our device that can be piloted by an API, we were ready to start working on the bot.

Here are some example of how we build our interactions with Speculos

The first iteration looked something like that
```js
function deviceActionAcceptBitcoin({
  // transport is an ojbect that represent the connection 
  // to a device
  transport,
  
  // events are received from Speculos and give us what is 
  // currently displayed on the device, allowing us to 
  // react to the different states
  event,
}: {
  transport: Transport<*> & { button: (string) => void },
  event: { type: string, text: string },
}) {
  // This is where we react to what is on screen
  if (event.text.startsWith("Accept")) {
    // Using Speculos API to trigger button actions, just
    // like a real user ! Here we press both buttons
    transport.button("LRlr");
  } else if (
    // Same here, we react to certain keywords displayed on
    // the device
    event.text.startsWith("Review") ||
    event.text.startsWith("Amount") ||
    event.text.startsWith("Address") ||
    event.text.startsWith("Confirm") ||
    event.text.startsWith("Fees")
  ) {
    // And we trigger a push on the right button
    transport.button("Rr");
  }
}
```

Although functionning at first, we've reworked it so we could also make assertions on what the device displays.

```js
const acceptTransaction: DeviceAction<Transaction, *> = deviceActionFlow({
  // Array of `steps` which are basically the 
  // different screens displayed on the device
  steps: [
    
    // Here we have a more concise and unified format
    // for all the steps
    {
      // Text displayed on the device
      title: "Amount",
      // Which button to press
      button: "Rr",
      // And here is the interesting part where
      // we can make assertion with the data provided
      // by the device AND the account/status data
      // linked to the transactions
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

## The coin spec, or the backbone of our tests

To make our bot _smart_ in its decision making process, we rely on `specs` files. A **coin spec** defines all possible mutations on the blockchain (supported by Ledger Live) as well as the expectation after the mutation has been broadcasted. This is where things get fun, as we said earlier, there are no way to "replay" scenarios on the blockchain. To compensate for these limitations, we came up with a new way, focusing on the _account state_ before and after the mutations.

Here is an example of a coin spec:

```js
// As a convention we call the specs: [coin]Spec
const dogecoinSpec: AppSpec<*> = {
  // Name of the coin we are testing
  name: "DogeCoin",
  currency: getCryptoCurrencyById("dogecoin"),
  // Dependency is related to the device app (here dogecoin) and if it 
  // requires any other app installed for it to work
  dependency: "Bitcoin",
  // Metadata used to spawn the simulator
  appQuery: {
    model: "nanoS",
    appName: "Dogecoin",
    firmware: "1.6.0",
    appVersion: "1.3.x",
  },
  // This is where the real fun is. The mutations array is a list
  // of possible mutations that could be apply to the account
  // if all the conditions are valid
  mutations: [
    {
      // Name of the mutation
      name: "send max",
      // Transaction is where we will use live-common logic to 
      // create the transaction type we wish to test
      transaction: ({ account, siblings, bridge }) => {
        // invariant are `conditions` that need to be true for the
        // mutation to be considered when running the bot
        invariant(account.balance.gt(100000), "balance is too low");
        // We create a new transaction
        let t = bridge.createTransaction(account);
        // We pick a sibling account (or create one) 
        // so the funds stays on the same seed
        const sibling = pickSiblings(siblings);
        const recipient = sibling.freshAddress;
        // We update the transaction and return it
        t = bridge.updateTransaction(t, { useAllAmount: true, recipient });
        return t;
      },
      // Here we plug a `deviceAction` that will take care of the various
      // screens displayed on the device
      deviceAction: deviceActionAcceptBitcoin,
      // And finally, the assertion
      // Here we get a lot of informations about the previous and current account
      // state as well as transaction and some other data 
      // we can use to make some assertions
      test: ({
        account,
        accountBeforeTransaction,
        transaction,
        optimisticOperation,
        operation,
      }) => {
        // This is where we need to be creative:
        // as scenarios cannot be replayed, we described our tests
        // to be some sort of "meta" assertions. 
        expect(account.balance.toString()).toBe("0");
      },
    },
  ],
};
```

## The Bot Logic

So now that we have an API to control a software version of our device, and the spec files, how does the Bot run.

=> Cronjob on Github Actions
=> Randomly choosed between mutation
=> Execute only one operation per run per account
=> Report on PRs and Commits

## What's next?

Automating swap?


{% include signatures/gre.html %}  

and  

{% include signatures/valpinkman.html %}  
