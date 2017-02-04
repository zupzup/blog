I came in contact with Blockchain technology only quite recently (mid 2016) and have been an interested, but rather passive member of the local [Blockchain community](https://blockchainhub.net/graz/). However, when I was asked if I would be interested in organizing a Coding Dojo on a Blockchain topic, I was motivated to dive a little deeper. For this purpose, I wrote my first **Smart Contract** for the **Ethereum Blockchain** using **Solidity**.

In this post, I won't explain what a Blockchain is or what Ethereum and Smart Contracts are, I rather want to share the example I built for the Coding Dojo to give the reader an idea of what building a DApp (decentralized application) looks like purely from the technological ecosystem and the code required to get something running.

Let's get started!

*The whole code for this example can be found [here](https://github.com/zupzup/solidity-example-crowdfunding)*

## The Contract

First of all, I want to outline the application we're going to build. If I think about blockchain technology, I always have things in mind which deal with **money** and with **trust** as well as the words **hype** and **buzzword**, so it seems only natural to build a crazy crowdfunding system for reckless startups! ;) 

I present the **Winner Takes All Crowdfunding Contract**:

* Create contract with
  * Minimum Entry Fee for project proposals (to avoid spam)
  * Project proposal deadline
  * Campaign deadline
* Before the Project proposal deadline, people can enter their projects providing a *name*, a *url* and the *entry fee*
* When the Project proposal deadline is over, people can *vote* with *ether* (money) for the projects they want to support
* When the campaign deadline is over, ALL the pledged money (incl. entry fees) goes to the project with the highest votes (most *ether* received)
  * And the contract is closed thereafter

Maybe a bit crazy and not very practical, but definitely an interesting example :)

## Setup

First up, as with every other software project, we'll need some initial setup to get going. There are already some frameworks (of course there are...) for building DApps, which are mentioned in the resources below. However, in our example we will stay with a very simple setup.

We will use [Solidity](http://solidity.readthedocs.io/en/develop/index.html) as the language for building the contract and [web3.js](https://github.com/ethereum/web3.js) for testing and creating a simple frontend for interacting with the contract.

Because installing dependencies is bothersome, we will also use [Docker](https://www.docker.com/) for running our local blockchain and for building our contract. We will use [testrpc](https://github.com/ethereumjs/testrpc) as our local blockchain, which is convenient, as Smart Contract run on Gas (money) and with testRPC it at least won't cost us any real money and all blocks are mined instantly.

We won't deploy the contract to any *real* blockchain in this example, but you can find lots of documentation in the official docs of Ethereum and Solidity on that topic.

Now, for compiling our smart contract, there are several options. We can use [browser-solidity](https://ethereum.github.io/browser-solidity/) as a Web-IDE and copy/paste the generated `web3` code from there every time we change something. However, I prefer to work locally.

In order to do that, we need `solc` to compile our contract, then `one-lineify` (remove line breaks) the contract and somehow get it into our JavaScript application with `web3`.
For this purpose, I created a [docker container](https://hub.docker.com/r/mzupzup/soliditybuilder/) which mounts the given folder, runs `solc` on the `contract.sol` file in that folder, then `one-lineify`'s the code and writes it into a `contract.js` file in the same folder.

Now, this works for the purpose of this example, but doesn't generalize well (e.g.: multiple contracts), but the frameworks mentioned below all have mechanisms to avoid having to do this manually.

We could also go a different route and automatically paste the binary output of `solc` into the JavaScript file, or really any other way of getting our compiled contract into our application and on the blockchain. But for this very simple example, this simplistic container-based approach will suffice.

To run testRPC locally, execute

```
docker pull harshjv/testrpc
docker run -d -p 8545:8545 harshjv/testrpc
```

To run the solidity build container, execute

```
docker pull mzupzup/soliditybuilder
docker run -v /path/to/this/folder:/sol mzupzup/soliditybuilder
```

Or, if you want automatic file-watching as well (not on Windows), you may also use:

```
docker pull mzupzup/soliditywatcher
docker run -v /path/to/this/folder:/sol mzupzup/soliditywatcher
```

If it all works, we can begin working on our contract!

## Contract Implementation

The core part of this application will be the contract. The full code can be found in `contract.sol` within the [GitHub repo](https://github.com/zupzup/solidity-example-crowdfunding). [Here](http://solidity.readthedocs.io/en/develop/index.html) is a link to the official solidity docs.

I found it to be quite convenient to develop the basis of the contract using [browser-solidity](https://ethereum.github.io/browser-solidity/) and then, after I had an idea what it would look like, move to my local setup and `web3.js` testing (described below).

I'll go over the code step by step and explain my reasoning behind the decisions I made. Keep in mind that `Solidity` is under very heavy development. From the time I wrote this example up until now `Solidity` moved from version `0.4.6` to `0.4.10` with lots of changes, so this example could already be outdated when you read it ;)

Also, with this being my first contract and the whole sphere of Smart Contracts being relatively new, there aren't many best-practices established yet, so I'm sure many things can be done more efficiently / elegantly than I managed to do.

I took the simple approach, that when I ran into a problem and couldn't find an answer within the docs or a quick web search, then I built a workaround. Some of there workarounds might seem weird, but if you consider that these contracts need to run on a decentralized, no-trust system, some of the limitations within the language will become pretty clear.

```javascript
pragma solidity ^0.4.6;
```

The first line of the contract tells the compiler which version the following code is written in. 

```javascript
contract WinnerTakesAll {
```

TBD

```javascript
    uint minimumEntryFee;
    uint public deadlineProjects;
    uint public deadlineCampaign;
    uint public winningFunds;
    address public winningAddress;
```

TBD

```javascript
    struct Project {
        address addr;
        string name;
        string url;
        uint funds;
        bool initialized;
    }
```

TBD

```javascript
    mapping (address => Project) projects;
    address[] public projectAddresses;
    uint public numberOfProjects;
```

TBD

```javascript
    event ProjectSubmitted(address addr, string name, string url, bool initialized);
    event ProjectSupported(address addr, uint amount);
    event PayedOutTo(address addr, uint winningFunds);
```

TBD

```javascript
    function WinnerTakesAll(uint _minimumEntryFee, uint _durationProjects, uint _durationCampaign) public {
        if (_durationCampaign <= _durationProjects) {
            throw;
        }
        minimumEntryFee = _minimumEntryFee;
        deadlineProjects = now + _durationProjects* 1 seconds;
        deadlineCampaign = now + _durationCampaign * 1 seconds;
        winningAddress = msg.sender;
        winningFunds = 0;
    }
```

TBD

```javascript
    function submitProject(string name, string url) payable public returns (bool success) {
        if (msg.value < minimumEntryFee) {
            throw;
        }
        if (now > deadlineProjects) {
            throw;
        }
        if (!projects[msg.sender].initialized) {
            projects[msg.sender] = Project(msg.sender, name, url, 0, true);

            projectAddresses.push(msg.sender);
            numberOfProjects = projectAddresses.length;
            ProjectSubmitted(msg.sender, name, url, projects[msg.sender].initialized);
            return true;
        }
        return false;
    }
```

TBD

```javascript
    function supportProject(address addr) payable public returns (bool success) {
        if (msg.value <= 0) {
            throw;
        }
        if (now > deadlineCampaign || now <= deadlineProjects) {
            throw;
        }
        if (!projects[addr].initialized) {
            throw;
        }
        projects[addr].funds += msg.value;
        if (projects[addr].funds > winningFunds) {
            winningAddress = addr;
            winningFunds = projects[addr].funds;
        }
        ProjectSupported(addr, msg.value);
        return true;
    }
```

TBD

```javascript
    function getProjectInfo(address addr) public constant returns (string name, string url, uint funds) {
        var project = projects[addr];
        if (!project.initialized) {
            throw;
        }
        return (project.name, project.url, project.funds);
    }
```

TBD

```javascript
    function finish() {
        if (now >= deadlineCampaign) {
            PayedOutTo(winningAddress, winningFunds);
            selfdestruct(winningAddress);
        }
    }
}
```

TBD

And that's it! Now how can we test this? 

## Debugging / Testing

Debugging and Testing the Smart Contract are not particularly easy, but it's possible. The frameworks mentioned below all include some way of Unit Testing Smart Contracts, for example simply by starting `testRPC` and writing asynchronous JavaScript tests (e.g.: `chai` / `mocha`) with `web3` and validating that they had some impact on the blockchain.
There are also standalone libraries like [chaithereum](https://github.com/SafeMarket/chaithereum).

I mentioned `Events` above as a mechanic for debugging Solidity contracts and `web3` has a way to listen to these events, which can greatly help when writing complex contract logic.

You can do this using `allEvents`:

```javascript
var events = myContractInstance.allEvents([additionalFilterObject,] function(err, log){
  if (!err)
    console.log(log);
});
```

In general, `web3`, because it is a full API to the blockchain, can be very helpful for:

Querying the contract's public interface (Getters / public variables)

```javascript
crowdfunder.numberOfProjects(function(err, data) {
    // handle error and data
});

crowdfunder.projectAddresses[0];
```


Sending transactions to the contract

```javascript
crowdfunder.submitProject.sendTransaction(projectName, projectURL, {
    from: senderAddress,
    value: entryFee,
    gas: 600000,
}, function(err, data) {
    // handle errors and data
});
```

Checking the balances of different accounts on the blockchain

```javascript
web3.eth.getBalance(web3.eth.accounts[0]);

web3.eth.getBalance(web3.eth.accounts[1]);
```

As well as converting to/from Wei and some other utilities:

```javascript
web3.fromWei(web3.toDecimal(web3.eth.getBalance(web3.eth.accounts[0])), 'ether');

crowdfunder.minimumEntryFee(function(err, data) {
    if (!err && data) {
        console.log(web3.fromWei(web3.toDecimal(data), 'ether') + ' ether'));
    }
});
```

All in all, `web3` is pretty well documented and works as one would expect, providing a rich interface to build and test DApps.

## Frontend Implementation

I also quickly threw together a simple Web-UI in order to test the application in a nicer way. The code (not very pretty and without finishing touches) for the UI is also in the [GitHub repo](https://github.com/zupzup/solidity-example-crowdfunding).

It uses:

* [async.js](https://github.com/caolan/async)
* [web3.js](https://github.com/ethereum/web3.js)
* [foundation](http://foundation.zurb.com/) 

The app connects to `testRPC`, compiles the contract and provides handlers and a UI for:

* creating a contract
* submitting a project proposal
* pledging ether to a project
* finishing the contract

Using mostly `web3.js` and the methods mentioned above in **Debugging / Testing**.

This is what it looks like::

<center>
    <a href="images/simpleui.png" target="_blank"><img src="images/simpleui_thmb.png" /></a>
</center>

## Conclusion

After getting my feet wet with Smart Contracts, Solidity und Blockchain technology in general, from a developer's perspective, I found it to be very similar to *normal* web development.

I mean sure, on the *backend* it's very different. Instead of a `Go` or `NodeJS` app, there is a Smart Contract running on the Ethereum Blockchain and you have some new trade offs (Gas cost of deployment / state-changing operations).

But from the perspective of a frontend developer, it is actually very similar to usual web development. You have some endpoints which you call asynchronously to fetch data and to do transactions, all packaged up in a (hopefully) nicely designed Single Page Application.

Also, because the Ethereum ecosystem seems to favor JavaScript (`web3`, similarities with `Solidity`), you can, as a JavaScript developer, use a lot of things you already know and love.

I'm curious to see how Blockchain technology, Smart Contracts and especially the Ethereum ecosystem will continue to grow and mature. After having taken a deeper dive, I can at least say that from a developer's perspective, it's not as scary or different as I thought it would be. :)

#### Resources

* [Full Example on Github](https://github.com/zupzup/solidity-example-crowdfunding)
* [Official Solidity Docs](http://solidity.readthedocs.io/en/develop/index.html)
* [BlockchainHub Graz](https://blockchainhub.net/graz/)
* [testrpc](https://github.com/ethereumjs/testrpc)
* [browser-solidity](https://ethereum.github.io/browser-solidity/)
* [web3.js](https://github.com/ethereum/web3.js)
* [populus](http://populus.readthedocs.io/en/latest/)
* [truffle](truffle.readthedocs.io)
* [embark](https://github.com/iurimatias/embark-framework)
* [ethereum](https://ethereum.org/)
