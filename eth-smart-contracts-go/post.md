Despite [recent troubles](https://blog.ethcore.io/the-multi-sig-hack-a-postmortem/), Ethereum remains the biggest player regarding Smart Contracts within the Blockchain space and this doesn't seem likely to change anytime soon.

In my opinion, the technology itself has great potential and is very interesting from an academic perspective, but as the above mentioned issue and many before that show is that blockchain technology, smart contracts and especially the Ethereum ecosystem with Solidity are very immature and just not ready for prime-time / production use cases.

However, it's a great time to learn and get to know this technology and play around a bit to be prepared when it reaches an acceptable level of maturity for serious applications.

In my [previous post](https://zupzup.org/smart-contract-solidity/) on Solidity, I created a small application with a simple Winner-Takes-All Crowdfunding contract. In this post, we will take the `contract.sol` from my previous post and see if we can deploy and interact with it using Go.

Why Go? Well for one, Go is amazing ;) and the most widely used [Ethereum client](https://github.com/ethereum/go-ethereum) is written in Go, which means there is a nice ecosystem for interacting with Ethereum and smart contracts using Go with nice features such as code-generation and reusable helpers from shared libraries.

In this example, we won't use the real Blockchain as a deployment target, but rather use the `SimulatedBackend` provided by `go-ethereum` so we can safely test and experiment without spending any money.

The `Smart Contract` itself is pretty simple - I won't go into much detail on what it does or how it works, as that has been [covered already](https://zupzup.org/smart-contract-solidity/). Suffice to say, that the contract is deployed with 3 parameters:

* Minimum Entry Fee of a Project
* Deadline for Submitting new Projects
* Deadline for Supporting Projects 

Then, during the first phase, projects can be submitted using a `name` and a `url` with a transaction including at least the `Minimum Entry Fee`. In the second phase, projects can be supported by sending ether to their addresses on the contract.

However, in this post we will focus on:

* Deploying the contract
* Reading data from the contract 
* Interacting with the contract (Transactions) 
* Instantiating the deployed contract via address 

And we will do it all in Go and in under 70 lines of code. ;)

Let's get started!

## Code Example

In order to be able to follow along, you need a few things. First and most importantly, you need the [solc](http://solidity.readthedocs.io/en/develop/installing-solidity.html) Solidity compiler.

Then, just fetch `go-ethereum` and build it:

```bash
go get github.com/ethereum/go-ethereum
cd $GOPATH/src/github.com/ethereum/go-ethereum/
make
make devtools
```

Alright - with `solc` and `geth devtools` in place, we can start by generating a Go-version of the `contract.sol` file, which holds our smart contract:

```bash
abigen --sol=Contract.sol --pkg=main --out=contract.go
```

The generated code looks like [this](https://gist.github.com/zupzup/1d49098ec5f068c09c1e7667758cd4e0).

As you can see, we have methods for deploying and instantiating the contract, as well as a mapping of all public contract methods to Go.

The next step is to deploy the contract to the simulated Backend.

To do this, some setup is required. As mentioned above, we will be using the `SimulatedBackend` as our target blockchain for simplicity, but at the end of this post there will be a short section on how to do this with the `testnet` or even the real Ethereum blockchain.

Using some of `go-ethereum's` dependencies, we can start setting up:

```go
import(
    "fmt"
    "log"
    "math/big"
    "time"

    "github.com/ethereum/go-ethereum/accounts/abi/bind"
    "github.com/ethereum/go-ethereum/accounts/abi/bind/backends"
    "github.com/ethereum/go-ethereum/core"
    "github.com/ethereum/go-ethereum/crypto"
)

func main() {
    key, _ := crypto.GenerateKey()
    auth := bind.NewKeyedTransactor(key)

    alloc := make(core.GenesisAlloc)
    alloc[auth.From] = core.GenesisAccount{Balance: big.NewInt(133700000)}
    sim := backends.NewSimulatedBackend(alloc)
```

We just create a key, make a Genesis Account with a bunch of ether and initiate the simulated backend, which returns a `bind.ContractBackend`.

Now we can deploy the contract using the generated `DeployWinnerTakesAll` method:

```go
addr, _, contract, err := DeployWinnerTakesAll(auth, sim, big.NewInt(10), big.NewInt(time.Now().Add(2*time.Minute).Unix()), big.NewInt(time.Now().Add(5*time.Minute).Unix()))
if err != nil {
    log.Fatalf("could not deploy contract: %v", err)
}
```

We pass in an `auth` object, which represents our identity, the backend `sim` and values for the `Minimum Entry Fee`, `Project Deadline` and `Campaign Deadline` each using a bigInt. The method returns the address the contract will be deployed to as well as a handle for the contract and an error. There is also a transaction object returned, but we won't deal with it here.

Now that the contract is deployed, we should be able to interact with it. For example, we can check if the deadline we sent is correctly set in the contract:

```go
deadlineCampaign, _ := contract.DeadlineCampaign(nil)
fmt.Printf("Pre-mining Campaign Deadline: %s\n", deadlineCampaign)
```

However, if we execute this, we get back `<nil>` for the deadline. That is, because our contract wasn't mined yet. If we were using the real network as a backend, we would have to wait until that happens, but with our simulated backend we can simply do this:

```go
fmt.Println("Mining...")
sim.Commit()

postDeadlineCampaign, _ := contract.DeadlineCampaign(nil)
fmt.Printf("Post-mining Campaign Deadline: %s\n", time.Unix(postDeadlineCampaign.Int64(), 0))
```

And we get back the date we set during deployment:

```bash
Post-mining Campaign Deadline: 2017-07-23 20:37:22 +0200 CEST
```

Nice. So, we can read data which is exposed by the contract. Now we want to interact with it. In this case, the simplest thing would be for us to propose a new project by sending a transaction with a `name` and `url` of a project with at least the `Minimum Entry Fee` as value:

```go
numOfProjects, _ := contract.NumberOfProjects(nil)
fmt.Printf("Number of Projects before: %d\n", numOfProjects)

fmt.Println("Adding new project...")
contract.SubmitProject(&bind.TransactOpts{
    From:     auth.From,
    Signer:   auth.Signer,
    GasLimit: big.NewInt(2381623),
    Value:    big.NewInt(10),
}, "test project", "http://www.example.com")
```

Of course, we need to mine again...

```go
fmt.Println("Mining...")
sim.Commit()

numOfProjects, _ = contract.NumberOfProjects(nil)
fmt.Printf("Number of Projects after: %d\n", numOfProjects)
info, _ := contract.GetProjectInfo(nil, auth.From)
fmt.Printf("Project Info: %v\n", info)
```

...but then we get the following output:

```bash
Number of Projects before: 0
Adding new project...
Mining...
Number of Projects after: 1
Project Info: {test project http://www.example.com 0}
```

Awesome - this means that our project was created. So we are able to deploy a contract, read and write to it.

But what if the contract was already deployed and we just want to interact with it? Fortunately, the generated code includes a `NewWinnerTakesAll` method, which, using just the address of the deployed contract, lets us instantiate the contract:

```go
instContract, err := NewWinnerTakesAll(addr, sim)
if err != nil {
    log.Fatalf("could not instantiate contract: %v", err)
}
numOfProjects, _ = instContract.NumberOfProjects(nil)
fmt.Printf("Number of Projects of instantiated Contract: %d\n", numOfProjects)
```

We get the same return value as for our deployed contract and can interact in the exact same way with this version, which was instantiated by address.

Ok, so we went through all the steps we need to interact meaningfully with a contract, but only on the simulated backend. In order to use the testnet or the real Ethereum blockchain, there are only a few things we would need to adapt:

```go
const key = "your key json"
conn, err := rpc.NewIPCClient("/path/to/your/.ethereum/testnet/geth.ipc")
if err != nil {
    log.Fatalf("could not create ipc client: %v", err)
}
auth, err := bind.NewTransactor(strings.NewReader(key), "your password")
if err != nil {
    log.Fatalf("could not create auth: %v", err)
}
```

This yields the auth object we created ourselves above. Of course, please don't use keys and/or passwords in plaintext in your code, but load them in a secure way. ;)

If the contract is already deployed, we don't need to create the `NewIPCClient`, but can just dial to a node:

```go
conn, err := ethclient.Dial("/path/to/your/.ethereum/testnet/geth.ipc")
if err != nil {
    log.Fatalf("could not connect to remote node: %v", err)
}
```

That's it!

The full code for the example can be found [here](https://github.com/zupzup/smart-contracts-with-go)

## Conclusion

As I stated in the beginning of this post, in my opinion it's too early to rely on Solidity smart contracts for serious applications, but the potential of this and several other blockchain-based approaches to smart contracts is huge, so getting to know the tech around it is certainly worthwhile.

Go lends itself nicely to the task of interacting with Ethereum-based smart contracts, as there is a lot of reusable code from `geth` and even some documentation on how to do get started. This can of course be achieved with any other language (e.g.: using web3), but if Go is what you like it seems like a solid choice. :)

#### Resources

* [Previous Post on Solidity](https://zupzup.org/smart-contract-solidity/)
* [Full Code Example](https://github.com/zupzup/smart-contracts-with-go)
* [go-ethereum](https://github.com/ethereum/go-ethereum)
* [geth Go Bindings documentation](https://github.com/ethereum/go-ethereum/wiki/Native-DApps:-Go-bindings-to-Ethereum-contracts)
* [block42](http://block42.org/)
