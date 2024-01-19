*This post was originally posted on the [LogRocket](https://blog.logrocket.com/substrate-blockchain-framework-core-concepts/) blog on 31.12.2021 and was cross-posted here by the author.*

In a previous article, we took a look at how to [implement a basic blockchain application in Rust](https://blog.logrocket.com/how-to-build-a-blockchain-in-rust/). This blockchain app featured an absolute minimal subset of functionality and was far from usable in production. However, it was a good exercise to explore how to implement core blockchain concepts and how they might look in Rust.

The next step is to explore how to build an actual, real-world blockchain. We won’t build it from scratch — that would far exceed the scope of a single article, or even a series of articles. Instead, we’ll leverage the [Substrate blockchain framework](https://substrate.io/) for this task.

The objective of this tutorial is to help you understand what Substrate is and isn’t, the core concepts behind it, and how to build a real-world blockchain applications.

With the immense scope of this framework, we can only hope to scratch the surface, but the goal is to give you all the context and resources you need to take the next step. I hope this guide helps you cope with the overwhelming amount of new ideas you’ll be exposed to when you first start to explore the world of blockchain development.

Let’s get started!

## What is Substrate?

Substrate can be described as a blockchain framework — specifically, a framework for building customized blockchains. These blockchains can be run entirely autonomously, which means they don’t depend on any external technology to run.

However, [Parity](https://www.parity.io/), the company behind Substrate (cofounded by [Ethereum](https://ethereum.org/en/) cofounder Gavin Wood), also built the [Polkadot](https://polkadot.network/) network. Polkadot is essentially a decentralized, protocol-based blockchain platform for enabling [secure cross-blockchain communication](https://blog.logrocket.com/polkadot-blockchain-app-development-deployment/).

That means Polkadot can be used as a kind of bridge between blockchains, taking care of the communication layer between chains and making it possible to interact between different blockchains (even systems such as Ethereum and Bitcoin). This represents significant progress toward making the vision of Web3 — a decentralized, blockchain-based version of the internet — a reality.

Since they were developed by the same people, Substrate has first-class support for integrating with Polkadot, so any blockchain you create with Substrate can seamlessly be hooked into Polkadot.

Substrate also provides [forkless runtime upgrades](https://docs.substrate.io/v3/runtime/upgrades/) — the ability to upgrade the state transition function of the blockchain without triggering a hard fork.

## What is a blockchain framework?

Now that we have some context, what is a blockchain framework, in simple terms? Essentially, a [blockchain framework](https://blog.logrocket.com/top-blockchain-development-frameworks/) is a (huge) collection of tools and libraries for building a whole, runnable, secure, feature-complete (albeit basic) blockchain.

A blockchain framework takes care of most of the heavy lifting when it comes to:

- [Consensus](https://blog.logrocket.com/guide-blockchain-consensus-protocols/)
- [P2P networking](https://blog.logrocket.com/libp2p-tutorial-build-a-peer-to-peer-app-in-rust/)
- Account management
- Basic blockchain logic (blocks, transactions etc.)
- A client to interact with the blockchain

Of course, with the vast amount of code provided, Substrate also comes with extensive [docs and tutorials](https://docs.substrate.io/v3/getting-started/overview/) to help you to get started.

If you’ve worked with an extensive web framework before, a blockchain framework is — nothing like that at all. The scope, number of moving parts, relevant concepts, and extension points are simply at a different scale. 

A better comparison might be a fully-fledged game engine (think Unity) that provides a basic implementation for everything you’ll definitely need and a lot of what you might need with extension points everywhere for you to customize.

## Is Substrate customizable?

Of course, this means some decisions have been made in terms of architecture, which you won’t be able to easily change. Depending on your use case, you might need more customizablity, which a framework might restrict in some way. That’s the standard tradeoff: a blockchain framework saves you time at the expense of some things you’ll simply have to live with.

However, as we’ll explore in more detail below, Substrate provides some flexibility in this regard, enabling you to choose between more technical freedom and ease of development at several stages.

Substrate’s backend is built in Rust. It (and most of what Parity does in general) is also fully open source. So you can not only use Substrate, but also improve it by contributing back and forking parts of it to customize to your own needs.

## How does Substrate work?

Now that we know what Substrate is, let’s get a high-level overview of the framework, its moving parts, and the extension points we can use to create a custom blockchain.

Since we’re dealing with a decentralized peer-to-peer system here, the basic unit we’re talking about is a node. This is where our blockchain runs.

This node is run inside a client and provides all the basic components the system needs to function, such as p2p-networking, storage of the blockchain, the logic for block processing and consensus, and the ability to interact with the blockchain from the outside.

When we start a new Substrate project, we have three options to choose:

* Substrate Node
* Substrate FRAME
* Substrate Core

Let’s explore each option in more detail.

### Substrate Node

We’ll start at the top with Substrate Node. This is the highest level we can start at; it provides the most prebuilt functionality and the least technical freedom. It’s fully runnable and includes default implementations for all basic components, such as account management, privileged access, consensus. and so forth. We can customize the genesis block (i.e., initial state) of the chain to start out.

Here, we can run the node and get familiar with what Substrate provides out of the box, playing around with the state and interacting with the running blockchain to get familiar. Another way to accomplish the same is to use the [Substrate Playground](https://docs.substrate.io/playground/), where you can check out both the backend and frontend templates to familiarize yourself with them.

Once we’re ready to really build our own blockchain, however, we’d be better off going one level lower and using FRAME.

### Substrate FRAME

FRAME (Framework for Runtime Aggregation of Modularized Entities) is a framework for building a Substrate runtime from existing libraries and with a high degree of freedom to determine our blockchain’s logic. 

We’re essentially starting from Substrate’s prebuilt [node template](https://github.com/substrate-developer-hub/substrate-node-template) and can add so-called pallets — Substrate’s name for library modules — to customize and extend our chain. We’re also able, at this level of abstraction, to fully customize our blockchain’s logic, state and data types. This is certainly where most basic customization projects that aim to be close to Substrate leverage the best of both worlds between ease of development and technical freedom.

From this starting point, we have to invest minimal time in setting up our blockchain and can focus fully on our own customizations. We’ll still have to take some defaults as they come — or, rather, as they can be configured — but if we fundamentally want to do things differently, we can go one step lower still by starting from Core.

### Substrate Core

Substrate Core essentially means we can implement our runtime in any way we want, as long as it targets WebAssembly and adheres to the fundamental laws underlying Substrate’s block creation. Then, we can use this runtime and run it within the Substrate node. 

This approach certainly requires the most work and the highest level of difficulty, but it also comes with the highest degree of technical freedom while still being able to work seamlessly within the Substrate ecosystem.

Speaking of Substrate’s ecosystem, there is a vibrant community of developers using Substrate in their own projects, many of whom contribute back to the ecosystem by sharing their own pallets.

You can find pallets by using a site such as [Substrate Marketplace](https://substratemarketplace.com/) or simply wherever [Rust crates are hosted](https://crates.io/), since Substrate pallets are essentially self-contained Rust libraries you can integrate into your Substrate project and configure as you need. As with any other library, it’s advisable to audit the code first and to be aware of the tradeoffs of relying on external code versus writing your own.

After playing around a bit with a prebuilt node, we should focus on FRAME, learning how to build a custom blockchain by building on top of the Substrate template node. This is also where many of the fantastic tutorials we’ll take a look at later in the article start off.

Next, we’ll go over some of the core concepts tht make up a Substrate blockchain, adding some color to our rough picture of how this all fits together.

## Core concepts of Substrate blockchain development

To get a deeper understanding of Substrate, we’ll explore some core concepts of the framework. 

The journey begins at the runtime, the heart piece of your custom blockchain. Then we’ll look at consensus and storage — where our blockchain lives and how all nodes agree which is the correct one. 

We’ll finish our expedition by touching on extrinsics, transactions and execution, including how to interact with the blockchain, what such an interaction looks like, and how it becomes part of our blockchain state.

### Runtime

The runtime of your Substrate-based application is the heart and soul of the project. It contains a description of what kind of state can be present on the chain as well as the logic, which guides how the blockchain goes from one state to the next. I

In Substrate, the runtime is also called the state transition function. You can imagine the whole blockchain as a state machine. The runtime defines the rules for going from one state to the next. This means the runtime is absolutely central and, in fact, where you’ll spend a majority of your time when implementing your custom Substrate blockchain.

Substrate itself adheres to the principle of trying to be as open as possible when it comes to custom runtimes. There are some interfaces with which any runtime has to be compatible, but beyond that, you have full technical and creative freedom to implement things as you think they should be.

Additionally, as mentioned above, you can use FRAME to compose existing modules (pallets). That includes the 50+ pallets shipped with Substrate and the ones provided by third-party developers to build your runtime.

Some examples for functionality that falls into the category of a runtime include the logic behind building a new block, integrating data from outside the chain, and handling accounts down to actual cryptographic primitives to use.

Learning about runtime development and it’s facets will be central to successfully using Substrate in the long run.

### Consensus and accounts

Because blockchain systems are peer-to-peer and (usually) consist of multiple nodes without any hierarchy between them, there’s the inherent problem of agreeing on the correct state. 

If multiple participants in the network attempt to make changes to the state, the validity and order of these transactions might be different depending on where you are in the network (i.e., when you were notified of the changes). This can lead to conflicts between different parts of the network that need to be resolved quickly and safely,so the network can continue to function.

Blockchain systems use consensus engines for exactly this purpose. These engines implement rules that define how state-transition needs to happen (guaranteeing that it’s deterministic) and how to resolve conflicts. This touches on block creation, finality (i.e., knowing when a transaction is completed) and fork choice/conflict resolution between competing states.

Substrate provides a handful of [consensus options](https://docs.substrate.io/v3/advanced/consensus/#consensus-in-substrate) already, but it’s also possible to roll out your own, depending on your exact needs, or to mix, match, and extend existing solutions.

As for accounts — the notion of a participant in the network — Substrate also has us covered. As usual with blockchain systems, an account consists of one or a set of public/private key pairs.

Substrate also provides, besides very basic account handling, the possibility for accounts with different use cases. 

There are the concepts of a stash key and a controller key. A stash key, or key pair, in Substrate defines a stash account where you might keep a large number of funds. Here, the security of the key is very, very important and should be kept offline. You should almost never directly interact with the chain using a stash account.

Controller accounts are used when your use case requies frequent interaction with the chain. They can be designated by stash keys as proxies. These accounts are used for direct interaction and only need enough funds to cover transaction fees. The keys still need to be secured, but since they control fewer funds, the stakes aren’t that high.

Read more about the existing account and session implementation in Substrate in the [official documentation](https://docs.substrate.io/v3/concepts/account-abstractions/). Of course, it’s possible to extend or replace these parts as well.

### Extrinsics and transactions

In a blockchain, there are extrinsics and intrinsics — things that happen outside the chain and things that happen inside the chain.

Extrinsics are necessary for interacting with the state of the blockchain from the outside — for example, to put some new piece of information to the state or to make a change. In fact, a block in a Substrate blockchain contains a header and an array of extrinsics — information from outside the chain.

For example, let’s say we have a custom Substrate blockchain where we collect quotes from philosophic texts. If I want to add a new cool quote I just read, I would use my custom-implemented `add_quote` extrinsic — think of it like a function, which I call with a string — and execute it by creating a signed transaction, sending the necessary transaction fee based on the defined [weight](https://docs.substrate.io/v3/concepts/weight/) of the function.

This signals that I indeed want this piece of information added to the blockchain — the shared state between all nodes. If my transaction is valid and I have enough funds to pay the transaction fees, it will be added to a block (based on some defined ordering and some other rules). If the block is valid and accepted, it is added to the blockchain and executed, adding my new quote to the distributed ledger.

Fortunately, Substrate implements the mechanism for successfully executing this dance so we can simply rely on it for our custom extensions, making changes where we need them.

Extrinsics, transactions, and the execution model are very important to the inner workings of Substrate and blockchains in general. You can dig in further in the [official docs](https://docs.substrate.io/v3/concepts/account-abstractions/).

This was just a small selection of the core concepts within Substrate, but it should give you a rough idea of what’s what and where to look in case you’re trying to figure out whether some functionality is already present or you’ll have to completely roll it out on your own. 

At this point, for most of the standard blockchain-related tech, there’s a good chance Substrate has one or more implementations or a pallet to suit your needs. At some point, you’ll have to build on top of these abstractions. 

In those isntances, instead of blindly using the existing modules and algorithms, you may want to dive a bit deeper to really understand what you’re relying on. This still demands a lot less effort than it takes to understand and implement everything from scratch yourself — especially if you want to do so correctly.

## Documentation and further reading

At this point, we’ve covered what Substrate is, how it’s structured, and the core concepts behind the system. Where can you go from here to actually play around with Substrate, or even start implementing your own real blockchain project?

Fortunately, Substrate has great tutorials, even for absolute beginners, covering every step from [installation](https://docs.substrate.io/v3/getting-started/installation/), to your [first running Substrate blockchain](https://docs.substrate.io/tutorials/v3/create-your-first-substrate-chain/), to a [full, customized blockchain example](https://docs.substrate.io/tutorials/v3/kitties/pt1/).

To fill in the gap, which you might still have after going through these tutorials, there are also some helpful [how-to guides](https://docs.substrate.io/how-to-guides/v3/) for specific topics.

Another great way to learn is to look at the many [open source projects](https://substrate.io/ecosystem/projects/) people are building on top of Substrate.

Have fun diving deeper and learning!

## Conclusion

In this tutorial, we took a look at the Substrate blockchain framework, where it comes from, how it’s structured, and how to dive into it.

Substrate has been in development for a while and has matured nicely. The amount of work required to create a robust framework for building blockchains — to to mention enabling beginners to learn and achieve their goals — is staggering, and the team at Parity has done a fantastic job so far.

There’s obviously still a lot to do and a project such as Substrate is never finished or perfect. But I’m very much looking forward to seeing many cool blockchain-related projects pop up in the near and not-so-near future, many of which will likely be based on, or at least inspired by, Substrate.

