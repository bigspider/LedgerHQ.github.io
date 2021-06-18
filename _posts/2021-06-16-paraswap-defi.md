---
layout: post
title: Ledger Live X DeFi
author: Pierre Pollastri
summary: When security meets Decentralized Finance.
featured-img: paraswap-logo
categories: Tech
---

# When security meets Decentralized Finance

The growth of Decentralized Finance (DeFi) has opened a whole new world. A world where anyone has the ability to create their own financial product, to manage their assets, and to bypass any interference from institutions. And this is just the beginning, this is an exponential technology that is just getting started! 

We, at Ledger, truly believe in DeFi, and this is why we wish to allow every developer around the world to be able to create Decentralized Applications (DApps) that use our platform, and integrate them into the Ledger ecosystem. 

Today, we just made this easier. 

In this article, we’d like to explain you how it works, and show you an example with "ParaSwap", a DApp that allows you to swap coins within Ledger Live.

Ledger Live is already able to securely manage ETH and ERC20 tokens on the Ethereum network. However, adding support for DApps built on top of Ethereum remained difficult for developers. That changes today with the release of Ledger Live 2.29. 


<br/>
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/paraswap_in_ledger_live.png">
    <br/><br/>
    <figcaption>A screenshot of paraswap.</figcaption>
  </figure>
</center>  
<br/>

# Ledger Live becomes the most secure DeFi enabled wallet

## DApp Sandboxing & Blockchain access

A DApp is a simple page “protected” by the browser sandbox model. In principle it guarantees isolation between the different tabs. A DApp is usually the first place where a user builds a transaction. Thus, even if we mitigate any risk by verifying the transaction on the Nano S/X, this isolation is very important to protect the user from any untrusted environment interference.

Additionally, DApps access the Ethereum blockchain and wallets through a specific Web3 provider that needs to be made available on the page. To achieve this, most software wallets  need a browser extension to work. Unfortunately, for extensions to work they require full rights to inject any Javascript code inside the page. This means users have to fully trust the extension when browsing normal websites or DApps.

Our solution is to integrate DApps inside an iframe to benefit from the integrated browser sandbox model. In addition, we make the Web3 provider available through an “iframe Web3 provider”. This “iframe Web3 provider” is an [open source project](https://github.com/LedgerHQ/iframe-provider) that enables the routing of Web3 calls to the parent iframe container using postMessage. In turn, this postMessage call is routed either to a standard Ethereum node or to Ledger Live when it requires interaction with the Nano S/X or user accounts.

<br />
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/dapps_browser_architecture.png">
    <br/><br/>
    <figcaption>DApps Browser Architecture.</figcaption>
  </figure>
</center>
<br />

With this implementation, Ledger cannot modify any code inside the iframe without the DApp provider’s consent and the hardware wallet is fully isolated from the DApp code.

As an added benefit, Ledger Live users can at all times and easily change the active account used by the DApp instantly, without leaving the DApp. 

## Don't trust, Verify: the blind signing problem

### The first time a hardware wallet can understand DApps

Ledger upgrades DeFi user security by adding native DApps support to Nano S and Nano X users in Ledger Live. No more extensions, no more browser connection issues, no more blind signing. Usability and security in a single package.

Most DeFi applications are running on top of Ethereum blockchain and use Ethereum smart contracts to perform complex operations. Users send transactions to smart contracts with all the required information, to execute specific functions with a specific set of parameters. Smart contract transactions carry an additional data field containing relevant parameters for the smart contract.

<br />
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/smart_contracts_parameters.png">
    <br/><br/>
    <figcaption>Smart contract parameters</figcaption>
  </figure>
</center>
<br />

Both Ledger Nano S and Nano X are already able to verify standard fields in Ethereum transactions (from, to, value, gas limit, gas price, nonce). However, when it comes to the input data, it was impossible to display the smart contract functionality that was activated by the transaction, as the Nano S/X had no knowledge of the contract and its purpose.

<br />
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/random_contract_call.gif">
    <br/><br/>
    <figcaption>A random contract call, it is impossible to verify what side effects it will have</figcaption>
  </figure>
</center>
<br />

With the introduction of the DApp browser in Ledger Live, we also give the Nano S/X the ability to display and verify the data included in a smart contract transaction, without any additional changes to the DApp. This results in a huge boost in security of the Ethereum ecosystem. We will describe this “plugin” mechanism in an upcoming blog post!

<br />
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/paraswap_contract_call.gif">
    <br/><br/>
    <figcaption>A ParaSwap contract call, we can easily verify swap details</figcaption>
  </figure>
</center>
<br />

# Adding you DApp to Ledger Live

To function properly, decentralized applications first need to be connected to a wallet. The latter will provide fresh information straight from the blockchain, such as user balances or smart contract state, and offer secure transaction signing on request. Since many wallets are compatible with DApps, the community had to come up with a standard in order to have a common interface. This standard is called [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193). To explain it briefly, every wallet has to implement a provider, respecting the interface defined by the standard. The DApp then prompts the user to select one of the available providers, allowing him to select his wallet.

<br />
<center>
  <figure class="image">
    <img src="{{ site.baseurl }}/assets/paraswap/list_dapp_providers.png">
    <br/><br/>
    <figcaption>A list of available providers on a DApp</figcaption>
  </figure>
</center>
<br />

## Installing the iframe-provider

To make your DApp compatible with Ledger Live, you first need to add the iframe-provider library to your project. This package is currently hosted on npm as [@ledgerhq/iframe-provider](https://www.npmjs.com/package/@ledgerhq/iframe-provider). This provider can be added alongside other existing providers, not changing existing features such as integrations working on Nano S/X. 

A lot of popular wallet implementations use Javascript injection to dynamically inject their provider. This approach is very tempting but could also result in the injector to modify the page, and present a security risk. This approach is not possible in our case because DApps running in Ledger Live are sandboxed. The only way to communicate between Ledger Live and the DApp is to use the [PostMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) bidirectional API, allowing secure cross-origin communication

## Allowing your DApp to be embedded

For a browser to allow a page to embed another page’s content when they are hosted on different origins, you need the specific approval of the hosted page’s server. This approval is expressed in the form of response headers.

To allow Ledger Live to embed your DApp, you might need to edit your server’s configuration and edit the Content-Security-Policy header.

- To allow any websites to embed your page:
`Content-Security-Policy: frame-ancestors '*';`

- To allow only Ledger DApp browser to embed your page:
`Content-Security-Policy: frame-ancestors `
`https://platform-apps.ledger.com`

## Adding the “what you see is what you sign” property

As of today, most smart contract interactions on hardware wallets are quite difficult to understand for users:
- When the smart contract is not well supported, the user has no way to verify what he is about to sign. In that case, the hardware wallet secures his seed against malware, but not really his ethereum funds from this threat. 
- When the hardware wallet displays the payload that the user is about to sign with minimal parsing, it makes it difficult to actually verify what he is about to consent to. 

To solve this issue, we’ve developed a simple Ethereum plugin mechanism minimizing the burden for developers to support the smart contract interaction while providing maximum security to the users thanks to the secure display. This mechanism will be explained in detail in a further blogpost.


# Conclusion

With the 2.29 release of Ledger Live, we intend to provide an environment to easily integrate a DApp, with minimal - and standard - modifications. We also want to achieve a better user experience while increasing the security level of users’ interactions with DApps. 

We are very excited by this release and the possibilities it creates to easily integrate the world of DeFi and DApps into Ledger Live. We hope to add many more DApps in the near future, and this is only the beginning of our journey to make Ledger Live more open and easily extensible.

Stay tuned with us as we strive to create a smooth user experience gathering all crypto services in the same platform without any compromise in security!


{% include signatures/pierre.html %}
{% include signatures/hugo.html %}
