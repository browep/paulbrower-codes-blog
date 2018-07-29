---
title: "Permissionless Blockchain Programming"
date: 2018-07-27T14:56:47-06:00
---

### Leverage the blockchain so your app needs neither permission nor forgiveness.  This is my quick adventure into using NodeJS and the Ethereum blockchain to write an unstoppable app.

_TLDR; I built an app that takes the Gitcoin Bounties and creates an RSS Feed using NodeJS, Web3JS, AWS S3, and AWS Lambda_

Anyone who has deployed an app to the Google Play Store or the Apple App Store or built something using the Twitter API or anything that requires permission to do can tell you that the rug can be pulled out from under you at any moment.  You can wake up to an email about your app "violating" some obscure Terms of Service clause or hit some limit or maybe things just stop working and you have to just figure it out.  

One of the interesting things about programming with a distributed ledger system, the Ethereum blockchain in this case, is that if you can talk the protocol then you can use the ecosystem.  That's it.  No Terms of Service need to be accepted, no API keys to revoke, you need only the documentation to get started.  In the case of Ethereum, you need only work within the confines of the public smart contract interface.  If the code works, it will work forever.  _It's a nice feeling._

#### The project is pretty straightforward:  Take the [Gitcoin bounties](https://gitcoin.co/explorer) and turn them into an RSS feed.  

If you are not familiar with Gitcoin, they are a community where you can get paid in cryptocurrencies to complete work on open-source repositories, among other great things.  Check them out, the community is excellent.  Gitcoin does not currently have an RSS feed and I really enjoy both the Gitcoin community and consuming things via RSS so it seemed like a useful goal to help at least 1 person (me).  

How the data is structured:

* [The smart contract](https://etherscan.io/address/0x2af47a65da8cd66729b4209c22017d6a5c2d2400#code) contains the bounties in an array of strings
* The `getBountyData(uint256)` method takes an array index and returns a string.  You can play around with it at [https://etherscan.io/address/0x2af47a65da8cd66729b4209c22017d6a5c2d2400#readContract](https://etherscan.io/address/0x2af47a65da8cd66729b4209c22017d6a5c2d2400#readContract) .  Try using `860` as the `_bountyId`
* That string is a hash that [IPFS](https://ipfs.io/) uses as key to retrieve a file.
* That file contains JSON which has the bounty data (bounty amount, link to github issue, requirements, etc)


A solution would include:

* querying the smart contract for the latest bounties
* getting the location for the bounty data (hash)
* retrieving the file for that hash
* concatting all those files
* creating an RSS feed
* uploading it to a place that can serve it

Other considerations:

* Lots of these data sources are distributed but might not be reliable, an ipfs node may take a while to get your data because it needs to request it, your ethereum node can go down, etc.  So caching data that is immutable is a good idea.
* RSS can be served statically and updated when needed so we don't need a dynamic solution to serve it.
* We don't need live updates for the smart contract, polling is fine.

My approach:

* Use easy to swap infrastructure services where possible to find the most reliable:  ( [Infura](https://infura.io/) to read the smart contract and a [IPFS Gateway](https://ipfs.github.io/public-gateway-checker/) to get the files ).  They can be replaced with locally running nodes if need be.
* Let AWS do the CDN and caching work ( S3 to serve the RSS file and to save the JSON files locally )
* Let AWS Lambda run the code periodically

A quick diagram will help us:

![Gitcoin RSS Sequence Diagram](/gitcoin_wsd.png)

The more interesting part of the app is how to interact with a smart contract using Web3JS, let's have a look:

{{< highlight javascript  >}}

// load the contract data so we can use its methods
const abiArray = require('../contract-abi.json');

const Web3 = require('web3');

const contract = (nconf) => {
    const ethNodeAddress = nconf.get('eth_address');

    // set the connection info
    const web3 = new Web3(
        new Web3.providers.HttpProvider(ethNodeAddress));
    // load the contract and make an object out of its
    // interface
    const BountyContract = web3.eth.contract(abiArray);
    // link the contract to a current deployment
    const contractInstance = BountyContract.at(
        nconf.get('contract_address'));

    // call a method on the contract with no args
    const getNumBounties = (cb) => {
        contractInstance.getNumBounties(cb);
    }

    // return a promise for a contract method that
    // takes an arg and has returned data
    const getBountyData = (bountyId) => {
        return new Promise((resolve, reject) => {
            contractInstance.getBountyData(bountyId,
             (error, bountyData) => {
                if (error) {
                    reject(error);
                } else {
                    resolve([bountyId, bountyData]);
                }
            });
        });
    };

    return {
        getBountyData: getBountyData,
        getNumBounties: getNumBounties
    }
}

{{< / highlight >}}

from [https://github.com/browep/gitcoin-rss/blob/master/lib/contract.js](https://github.com/browep/gitcoin-rss/blob/master/lib/contract.js)


Pretty straightforward.  You can get the ABI ( Application Binary Interface ) for lots of contracts via etherscan.io, ours is at [https://etherscan.io/address/0x2af47a65da8cd66729b4209c22017d6a5c2d2400#code](https://etherscan.io/address/0x2af47a65da8cd66729b4209c22017d6a5c2d2400#code).  The JSON in `contract-abi.json` is simply copy pasted from there.  The ABI is an interface description of how to interact with the contract.

That's about it.  You can get the feed at: [https://s3-us-west-2.amazonaws.com/gitcoin-rss/feed.rss](https://s3-us-west-2.amazonaws.com/gitcoin-rss/feed.rss)  and find the full project at [https://github.com/browep/gitcoin-rss](https://github.com/browep/gitcoin-rss) Which has the full implementation.

### Takeaways

* I learned a lot about NodeJS as this was my first project using it.  It's pretty easy to pick up as there are lots of familiar paradigms that other newer languages or frameworks use.
* Javascript being weakly typed allows for pretty rapid programming but you pay for it with harder to understand errors.  I might check out TypeScript as a compromise.
* AWS Lambda cut out about 30% of the work of the project by not having to create a server and will cut out future maintenance.
* With permissionless blockchain development there are fewer gate keepers to data and functionality.



