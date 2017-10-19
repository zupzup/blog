Working on a project for the last couple of months with the guys of [block42](http://block42.org/), we noticed that in order to use a technology such as Ethereum in a real-world project, every user needs to have an accound with some ether to be able to interact with the blockchain.

This poses a real problem, as either Ethereum nor BitCoin or any other blockchain technology is anywhere near mainstream adoption yet and won't be for the foreseeable future.

Also, even if a user has dabbled in cryptocurrencies, there is no guarantee that the user holds ether or the currency needed on the platform you are using. To tackle this issue, there is a [proposal](https://github.com/ethereum/EIPs/blob/bd136e662fca4154787b44cded8d2a29b993be66/EIPS/abstraction.md) on Ethereum to make it possible for contracts to pay for transactions. Unfortunately, this change will hit, at the earliest, somewhere around next year with the second Metropolis release "Constantinople".

In the meantime, there are several workarounds one can do with different tradeoffs each. This post will show a Proof-of-Concept implementation of one of these workarounds with Go.

## Concept

The idea is pretty simple. A User requests something (e.g.: some kind of Agreement) and, if the Server is okay with it, the Server puts it on the blockchain, inside a smart-contract. Now the Server wants the User to sign the Agreement on the blockchain, to be able to proof afterwards, that both parties agreed.

The User doesn't have any ether, so the Server sends enough ether to the User (very minimal amount for the transaction fee). Of course, the User needs an Ethereum address to be able to receive it and to be able to sign the Agreement. For this purpose, there might be an App or WebApp, which creates an Account for the user and provides the ability to validate the Agreement, as it is saved on the blockchain and to sign it using the received transaction fee.

**Example:**

* User installs App and Ethereum Account (Address) is created for the User
* User sends an *Agreement* and Public Key to the Server
* Server puts the *Agreement* on the smart-contract, only accessible by the Users address
* Server sends a calculated Transaction Fee to the User, which is enough that the User can sign the *Agreement*
* User can now query the *Agreement* on the smart-contract and check if it's OK
* User can, once the Transaction Fee arrived, Sign the *Agreement* on the smart-contract

Now, of course this approach won't work for many use-cases and there need to be some anti-fraud mechanisms in place. Also, it introduces some centralization, which is always frowned upon in blockchain applications. But the approach helps both sides, the User and the Server, to have persistent proof on the blockchain, that they agreed on something at some time.

A Proof of Concept of this can be implemented using a Webserver as the *Server*  and a WebApp as the *Client*, [testrpc](https://github.com/ethereumjs/testrpc) can be used for local testing.

## PoC Implementation 

TBD

```go
```

That's it!

The full code for the example can be found [here](https://github.com/zupzup/eth-prepaid-transaction)

## Conclusion

The outlined concept won't work for a lot of applications, but for simple signing-use-cases, which are a good fit for blockchain platforms, it seems sufficient.

The solution works pretty well and could, I believe, even serve as a basis for a real-world implementation of such a mechanism. Of course there need to be some serious precautions, or even a manual process for actually sending out transactions fees to arbitrary users, but the general concept works.

I'm curious how the blockchain space will deal with this issue in the future and which security and anti-fraud mechanisms will be provided by the platforms. What I am sure of today is that smart contract platforms will need mechanisms like this to shield users from the need to deal with cryptocurrency.

Please don't use this simplistic implementation for anything serious, as this will certainly end in tears, but maybe use it for inspiration or for learning in regards to the possibilities of interacting with the ethereum blockchain using Go. :)

#### Resources

* [block42](http://block42.org/)
* [Full Code Example](https://github.com/zupzup/eth-prepaid-transaction)
* [Previous Post on Solidity](https://zupzup.org/smart-contract-solidity/)
* [Previous Post Ethereum with Go](https://zupzup.org/eth-smart-contracts-go/)
* [go-ethereum](https://github.com/ethereum/go-ethereum)
* [testrpc nonce bug](https://github.com/ethereum/go-ethereum/issues/14909)
* [geth Go Bindings documentation](https://github.com/ethereum/go-ethereum/wiki/Native-DApps:-Go-bindings-to-Ethereum-contracts)
* [testrpc](https://github.com/ethereumjs/testrpc)
* [EIP including self-paying contracts](https://github.com/ethereum/EIPs/blob/bd136e662fca4154787b44cded8d2a29b993be66/EIPS/abstraction.md)
