Introduction

Goal: Play around with Solidity and get something to work (not production ready)

## Setup

// TODO: links and short text on different setup methods and frameworks (truffle, embark, populus)

* docker for testRPC
* install solc
* ./build.sh script, compile & one-line-ify
* web3.js

## The Contract

// TODO: formulate

Winner Takes all Crowdfunding
* Create contract with target date in the future and minimum entry fee
* People can enter their projects (with an initial money transfer of X ether, so there's no spam)
* Also, a deadline for project proposals
* Now, other people can "vote" with money for the projects
    * transaction to the contract, with amount and address/name of the receiving project
    * When the target date is here, ALL the money, goes to the project with the highest votes (most money contributed)
        * and contract is closed thereafter

Maybe not very practical, but an interesting example :)

## Implementation

// TODO: go over contract step-by-step
// TODO: explain tradeOffs, redundancies (can't return structs / mappings, arrays weird to return)

## Testing

// TODO: some web3 calls, how to call it, how the verify, maybe link some testing framework 

I also quickly threw together a simple Web-UI in order to test the application in a nicer way. The code (very ugly and unfinished) for the UI is also on GitHub.

This is what it looks like::

<center>
    <a href="images/simpleui.png" target="_blank"><img src="images/simpleui_thmb.png" /></a>
</center>

## Conclusion

// TODO: Conclusion

#### Resources

* [Full Example on Github](https://github.com/zupzup/solidity-example-crowdfunding)
* [Official Solidity Docs](http://solidity.readthedocs.io/en/develop/index.html)
* [testrpc](https://github.com/ethereumjs/testrpc)
* [browser-solidity](https://ethereum.github.io/browser-solidity/)
* [web3.js](https://github.com/ethereum/web3.js)

