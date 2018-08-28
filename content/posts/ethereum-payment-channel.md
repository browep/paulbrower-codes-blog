---
title: Ethereum File Hub
date: 2018-08-25T14:56:47-06:00
---

### Ethereum File Hub is a Proof-of-Concept implementation of a payment channel used to purchase digital files between two parties who do not trust each other and does not require a third party.  It was an entry in the EthDenver 2018 hackathon.

You can read this blog post or watch pretty much the same content via the video:
<div align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/fBBufr0IoqM" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</div>

#### Background

Currently, if you want to buy or sell some digital content on the internet you have to involve quite a few other parties.  Let's take for example, a movie.  If you want to watch the latest Avengers movie, you can go to Netflix, create an account, enter your credit card info, select the movie and watch.  In the interaction you have several third parties that all take their cut.  Netflix resells the movie after buying the rights.  The credit card companies take a transaction fee for the purchase.  Each of these middle-men add layers of trust in the purchase.  You trust Netflix to actually deliver the movie since they are well-known and established.  Netflix trusts Visa/Mastercard to send the money for a similar reason.  

They each get their cut.  Would it be possible to do this without the two middle-men?  Could you buy a digital movie straight from the studio?  What if you had never heard of the studio but you were interested in watching the movie?  You could send payment directly to them via cash in the mail or cryptocurrency but it's possible they may not send you the movie in return.  They could send you the movie but they can't trust you to send the payment.  Since neither party trusts each other, layers of middle-men and distributors must be created to escrow and intermediate.  Not ideal.  Ideal would be where the two parties are happy ( consumer watches the movie, studio gets paid) and no one else extracts value ( distributors, CC clearing houses).

#### Solution

A naive solution could be to send a cryptocurrency payment for part of the whole, say 1% of the total price for 1% of the movie.  This lowers the risk for each party no matter who goes first.  If the studio send 1% of the movie but does not receive the payment then they have only incurred 1% of the cost.  If the consumer send 1% of the payment but does not receive 1% of the movie then they are only out 1% of the potential cost.  Much better.  This could get more granular until each party was comfortable. ( 0.1%, 0.001%, etc.)  Probably as small as a single frame of the movie.  

This could continue for each part of the movie, so 100 payments for 100 chunks of the movie.  This is undesirable for two reasons: One would need to wait for each payment to clear, very slow, and each transaction would incur transaction costs.  Increasing the granularity would increase the transaction costs making it more expensive for both parties.

A better solution includes a "payment channel" (sometimes called a "state channel").  A payment channel allows for a "counter factual" transaction and behavior.  Instead of sending a transaction in cryptocurrency for each movie chunk, a payment channel would be opened and value would be put in escrow.  For Bitcoin this would be a time-lock, for Ethereum this would be in a smart contract.  And then the consumer would send an IOU to the studio for the value of all the chunks sent so far, not to the blockchain, for each chunk.  Once the movie had finished whether by coming to the end or the consumer stopped watching, the IOU of the greatest value would then be redeemed by the studio and the remainder of the escrowed funds would be returned to the consumer.  This allows the server / studio to send data without seeing a transaction on-chain but still know that they have been sent funds (counter-factual).  

Let's take a look at a sequence diagram of this using the Ethereum blockchain:

![Payment Channel Diagram](/payment_channel.png)

Only 2 transactions are ever sent to the Ethereum blockchain ( EVM )

* The initial contract creation with the escrowed money and some metadata
* A transaction of the highest value IOU that the server/studio has

No matter how granular the chunks are, only two transactions need to be paid for, everything else happens "off chain".

Some more information about the "IOU" that is being sent from the client to the server.  The IOU is _not_ a transaction, but it is signed like a transaction.  It's sent as a argument in a transaction so a smart contract can verify who signed it.  The Java example of how this particular IOU is created can be found in [this file](https://github.com/browep/efh/blob/master/src/main/java/com/github/browep/efh/FileHubAdapter.java#L133)

Let's take a look at the smart contract: [https://github.com/browep/efh/blob/master/src/main/resources/solidity/efh/FileTransfer.sol](https://github.com/browep/efh/blob/master/src/main/resources/solidity/efh/FileTransfer.sol)

{{< highlight javascript >}}

contract filetransfer {

    address client;
    address server;
    uint256 fileHash;
    uint128 expirationBlock;

    function filetransfer(
        address _client, 
        address _server, 
        uint256 _fileHash, 
        uint128 _expirationBlock)
        public
        payable
    {
        client = _client;
        server = _server;
        fileHash = _fileHash;
        expirationBlock = _expirationBlock;

    }

    // the args to this method are the parts of the IOU
    // that is sent from the client to the server
    // including "v" which is the value that this IOU
    // represents
    function isRedeemable(
        bytes32 h, 
        uint8 v, 
        bytes32 r, 
        bytes32 s, 
        uint _value) 
        constant returns (bool) {

        // get the address used to sign the hash
        address recoveredAddr = ecrecover(h, v, r, s);

        // hash the _value to see if it matches what the
        // passed in hash is
        bytes32 proof = sha3(_value);

        return recoveredAddr == client && 
               proof == h              && 
               msg.sender == server;
    }

    function redeem(
        bytes32 h, 
        uint8 v, 
        bytes32 r, 
        bytes32 s, 
        uint value) 
        public returns (bool) {
        
        if (isRedeemable(h, v, r, s, value)) {
            if (value >= this.balance) {
                selfdestruct(server);
            } else {
                if(!server.send(value)) throw;
                selfdestruct(client);
            }
        } else {
            throw;
        }
    }

    function clawback() public {
        if (msg.sender != client) throw;
        if (block.number > expirationBlock) throw;

        selfdestruct(client);
    }
}

{{</ highlight >}}

The contract is rather simple with only 3 methods to call.

`isRedeemable` is a read-only function that incurs no gas cost.  This is used by the server to verify that the pieces of the data sent are redeemable later by checking the value and the signature of the data.

`redeem` uses `isRedeemable` to do the same check and sends funds according to the signed value.

`clawback` is for the case where the server does not execute any IOU and the funds are still locked in the contract.  If after the window has closed (`this.blockNumber > expirationBlock`) then the client reclaim their funds.  

These three functions make up the logic for the whole smart contract.

#### Why this is important

In our case of a studio trying to sell a movie direct to a consumer we have eliminated all third parties.  No distributors, no payment processors, no middle-men at all.  More importantly, it requires no trust.  Neither had to take the risk of "going first" and either losing funds or not getting paid.  It also inherits all the benefits of a blockchain solution: __permissionless__ ( no-one's permission was needed to buy or sell the movie), __borderless__ (the country of origin of the studio or the consumer never came up) and __censorship resistant__ (shutting down such a transaction would be extremely difficult as only an internet connection is needed)

#### Further applications

A movie was used as the example but the use case can be extended to pretty much any medium that satifies the following criteria. 

* Can be consumed in serial
* Parts of a file can be validated without the whole.  

If we see 1% of a movie, we know it's the correct 1% and we can do that without the full movie.

Lot's of media we consume fall into this category: livestreams, music, chat, books, professional advice.  They are all consumed serially and can be verified in chunks.  Some do not.  1% of a Linux ISO is completely useless.  

#### Further reading

More about state channels: [https://github.com/machinomy/awesome-state-channels](https://github.com/machinomy/awesome-state-channels)

The repo for Ethereum File Hub is at [https://github.com/browep/efh](https://github.com/browep/efh)

More info about acting counter-factually [https://medium.com/statechannels/counterfactual-generalized-state-channels-on-ethereum-d38a36d25fc6](https://medium.com/statechannels/counterfactual-generalized-state-channels-on-ethereum-d38a36d25fc6)

