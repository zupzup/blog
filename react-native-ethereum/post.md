In the Blockchain world, a light-client is an application, which runs it's own p2p node and is therefore connected to the whole network without any centralized intermediaries. This is, at least from a decentralized mindset perspective, a desirable property. From a purely technical perspective however, this comes with several problems.

One problem is, that the data-structure underlying these blockchain systems is usually very large (GBs - TBs), so it's simply not feasible to download it on a mobile device or sometimes even on a desktop. Because of this issue, the concept of light clients was developed. Such a light client, in the case of [ethereum](https://github.com/ethereum/wiki/wiki/Light-client-protocol) only downloads block-headers and verifies a lot less.

With such a light client it's possible to be a part of the whole p2p network and directly interact with the blockchain by deploying contracts, sending transactions, querying balances etc.

This concept has been working quite well for BitCoin in the past and is one of the things which successful blockchain-platforms seem to need at some point. In the ethereum space however, this is, at the time of this blog post, an experimental feature.

When I researched the subject quickly, I didn't find many projects using a light client for mobile yet. Most prominently there is [status](https://status.io/), who wrote their own wrapper of [go-ethereum](https://github.com/ethereum/go-ethereum).

I also found [walleth](https://github.com/walleth/walleth), is a fairly new project and a Kotlin-based light client using the cross-compiled Android package of go-ethereum.

This cross-compiled package is what I will also use in this blog post, as it seems to be the suggested and most standard way of approaching this problem at the moment. 

I will use react-native in this example, because I'm familiar with react and want to see if wrapping native modules is really [as simple as they say it is](https://facebook.github.io/react-native/docs/native-modules-android.html).

As usual in the ethereum world, there is little to no updated documentation available, adoption of the mobile wrapper seems to be very limited - which means very few examples and testing will be awkward because of the whole p2p thing, so it's probably going to be abit of a bumpy ride.

Let's get started!

## Setup

For setting up the react-native application, I usually simply use [create-react-native-app](https://github.com/react-community/create-react-native-app) and `eject` immediately. This gives us a working, debuggable application which can be started using `npm run android`.

## Code Example 

First, RN wrapper to be able to communicate

Second, get Geth to run

Third, communicate from RN

TBD

```javascript
```

## Problems

TBD

## Conclusion

TBD

#### Resources

* [Full Example on Github](https://github.com/zupzup/react-native-ethereum)
* [light client protocol](https://github.com/ethereum/wiki/wiki/Light-client-protocol)
* [react-native android module wrapping](https://facebook.github.io/react-native/docs/native-modules-android.html)
* [go-ethereum](https://github.com/ethereum/go-ethereum)
* [walleth](https://github.com/walleth/walleth)
* [status-go](https://github.com/status-im/status-go)
* [geth mobile docs](https://github.com/ethereum/go-ethereum/wiki/Mobile:-Introduction)
