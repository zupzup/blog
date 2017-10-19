Working on a project for the last couple of months with the guys from [block42](http://block42.org/), we talked about, how to use a technology such as Ethereum in a real-world project, every potential user needs to have an account with some ether to be able to interact with the blockchain and hence with the product.

This poses a real problem, as neither Ethereum nor Bitcoin or any other blockchain technology is anywhere near mainstream adoption yet and won't be for the foreseeable future.

Also, even if a user has dabbled in cryptocurrencies, there is no guarantee that the user holds ether or the currency needed on the current platform. To tackle this issue, there is a [proposal](https://github.com/ethereum/EIPs/blob/bd136e662fca4154787b44cded8d2a29b993be66/EIPS/abstraction.md) on Ethereum to make it possible for contracts to pay for transactions. Unfortunately, this change will hit, at the earliest, somewhere around next year with the second Metropolis release "Constantinople".

In the meantime, there are several workarounds one can do with different trade-offs each. This post will show a Proof-of-Concept implementation of one of these workarounds with Go.

## Concept

The idea is simple. A User requests something (e.g.: some kind of Agreement) and, if the Server is okay with it, the Server puts it on the blockchain inside a smart-contract. Now the Server wants the User to sign the Agreement on the blockchain, to have proof afterwards, that both parties agreed.

The User doesn't have any ether, so the Server sends enough ether to the User (a minimal amount for the transaction fee). Of course, the User needs an Ethereum address to be able to receive it and to be able to sign the Agreement. For this purpose, there might be an App or WebApp, which creates an Account for the user and provides the ability to validate the Agreement saved on the blockchain and to sign it using the received transaction fee.

**Example:**

* User installs App and Ethereum Account (Address) is created for the User
* User sends an *Agreement* and Public Key to the Server
* Server puts the *Agreement* on the smart-contract
* Server sends a calculated Transaction Fee to the User, which is enough that the User can sign the *Agreement*
* User can now query the *Agreement* on the smart-contract and check if it's OK
* User can, once the Transaction Fee arrived, Sign the *Agreement* on the smart-contract

This approach won't work for many use-cases and there need to be some anti-fraud mechanisms in place for it to be usable. Also, it introduces some centralization, which is always frowned upon in blockchain applications. But the approach helps both sides, the User and the Server, to have persistent proof on the blockchain, that they agreed on something at some time without the need for both parties to deal with cryptocurrency.

A Proof of Concept of this can be implemented using a Webserver as the *Server*  and a WebApp as the *Client*, [testrpc](https://github.com/ethereumjs/testrpc) can be used for local testing.

## PoC Implementation 

First, we start a local testrpc instance using Docker:

```bash
docker pull harshjv/testrpc
docker run -d --name=preprpc -p 8545:8545 harshjv/testrpc --account="0xb4087f10eacc3a032a2d550c02ae7a3ff88bc62eb0d9f6c02c9d5ef4d1562862, 1000000000000000000000000" --account="0xd2a99b289915eb11ea50a51247e1cef2c4583ae1d9699a3bb0154c2792bda339,0"
```

This starts testrpc with two accounts, one with funds (Server) and one without any funds (User).

Then, we need the smart contract. For this simple example, there is not much to it:

```javascript
pragma solidity ^0.4.6;

contract Signer {
    address public owner = msg.sender;
    struct Agreement {
        string stringToAgreeOn;
        bool signed;
        bool initialized;
    }
    mapping (address=> Agreement) agreements;

    modifier onlyBy(address _account)
    {
        require(msg.sender == _account);
        _;
    }

    function createAgreement(string _stringToAgreeOn, address customer) payable public onlyBy(owner) returns (bool success) {
        agreements[customer] = Agreement(_stringToAgreeOn, false, true);
        return true;
    }

    function signAgreement() payable public returns (bool success) {
        var agreement = agreements[msg.sender];
        require(agreement.initialized == true);
        require(agreement.signed == false);
        agreement.signed = true;
        return true;
    }

    function getAgreement(address addr) public constant returns(string stringToAgreeOn, bool signed, bool initialized) {
        var agreement = agreements[addr];
        return (agreement.stringToAgreeOn, agreement.signed, agreement.initialized);
    }
}
```

Basically, the contract holds a mapping from `address` to `Agreement`, so there is, at any time, at most one agreement per address. Agreements can only be created by the owner of the contract (`onlyBy`), but anyone can call `getAgreement`, to see agreements for a certain address.

Then there is the `signAgreement` transaction-method. In this method, the User simply sends a transaction and the active agreement, if there is one (`initialized==true`), and if it's not already signed (`signed == false`), is signed. This is the transaction, the User needs the transaction fee for. Not much is happening here, so it will be cheap.

I didn't spend any time optimizing or securing this contract, so don't take it as a template for anything you're building. It's just there to showcase the workflow.

Next up, there is a HTML-based UI with some JavaScript which I won't go into much detail on. The WebApp includes web3 and calls the `getAgreement` and `signAgreement` functions on the contract and fetches the balance for the user:

Signing an Agreement:

```javascript
var signbutton = document.getElementById("signbutton");
signbutton.addEventListener("click", function(event) {
    var tx = {
        from: currentAccount,
        value: 0,
        gas: 100000
    }
    instance.signAgreement.sendTransaction(tx, function(err, res) {
        if (err) {
            return alert(err)
        }
    })
});
```

Refreshing the active Agreement:

```javascript
var refresh = document.getElementById("refresh");
refresh.addEventListener("click", function(event) {
    instance.getAgreement(currentAccount, function(err, results) {
        if (err || !results[2]) {
            agreement.innerText = ""
            signed.innerText = "";
        } else {
            agreement.innerText = results[0];
            signed.innerText = results[1];
        }
    });
});
```

Refreshing the Balance:

```javascript
var refreshbalance = document.getElementById("refreshbalance");
refreshbalance.addEventListener("click", function(event) {
    var newBalance = web3.eth.getBalance(currentAccount);
    var balanceElement = document.getElementById("balance");
    balanceElement.innerText = newBalance.toNumber();
});
```

The UI can also create new Agreements using a REST API on the server:

```javascript
var agreementButton = document.getElementById("agreementButton");
agreementButton.addEventListener("click", function(event) {
    var agreementInput = document.getElementById("agreementInput");
    agreementValue = agreementInput.value
    superagent.post(host + "/agreement").send({
        account: currentAccount,
        agreement: agreementValue,
    }).end(function(err, res) {
        if (err) {
            return alert(res.text)
        }
        console.log(res);
    });
});
```

Alright, with the contract and the simplistic UI out of the way, let's look at the meat of this PoC - the logic for creating agreements on the smart-contract and sending the transaction-fee to the User.

First, we need to convert the smart-contract into a Go-API as described in [my previous post](https://zupzup.org/eth-smart-contracts-go/):

```bash
abigen --sol=signer.sol --pkg=main --out=signer.g
```

Then, some setup is needed to connect to `testrpc` and to setup an account to deploy and interact with the contract:

```go
var key = "b4087f10eacc3a032a2d550c02ae7a3ff88bc62eb0d9f6c02c9d5ef4d1562862" // don't hardcode keys in production y'all!
privKey, err := crypto.HexToECDSA(key)
if err != nil {
    log.Fatalf("Failed to convert private key: %v", err)
}
conn, err := ethclient.Dial("http://localhost:8545")
if err != nil {
    log.Fatalf("Failed to connect to the Ethereum client: %v", err)
}
auth := bind.NewKeyedTransactor(privKey)
if err != nil {
    log.Fatalf("Failed to create Transactor: %v", err)
}
```

In this case, we use a hardcoded private key, which will also be supplied to testrpc to create an account. Then this key is converted to an `ECDSA Private Key` and a `Transactor` is created, which is needed for doing transactions.

We also use `Dial` to open a connection to the local `testrpc` instance. With the blockchain connection and credentials up and running, let's deploy the contract:

```go
addr, _, contract, err := DeploySigner(auth, conn)
if err != nil {
    log.Fatalf("Failed to deploy contract: %v", err)
}
fmt.Println("Contract Deployed to: ", addr.String())
```

Alright, now we can set up the WebServer using [chi](https://github.com/go-chi/chi), with CORS headers set for convenience:

```go
r := chi.NewRouter()
corsOption := cors.New(cors.Options{
    AllowedOrigins:   []string{"*"},
    AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
    AllowCredentials: true,
    MaxAge:           300,
})
r.Use(corsOption.Handler)
r.Use(middleware.Logger)
r.Post("/agreement", createAgreementHandler(contract, auth, conn, privKey))

log.Println("Server started on localhost:8080")
log.Fatal(http.ListenAndServe(":8080", r))
```

The only route we add here is `createAgreementHandler`. You might argue that it would have been perfectly ok to just use the go standard lib for this webserver and you would be totally right. The truth is, I thought this PoC would be more complex when I started out and I over prepared. ;)

The `POST` handler at `/agreement` is the only interaction between the Server and the User, so this is where all the magic happens. Let's go over it step-by-step:

```go
// Agreement is an agreement
type Agreement struct {
	Account   string `json:"account"`
	Agreement string `json:"agreement"`
}

// Bind binds the request parameters
func (a *Agreement) Bind(r *http.Request) error {
	return nil
}

func createAgreementHandler(contract *Signer, auth *bind.TransactOpts, conn *ethclient.Client, privKey *ecdsa.PrivateKey) http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        agreement := &Agreement{}
        if err := render.Bind(r, agreement); err != nil {
            w.WriteHeader(http.StatusBadRequest)
            w.Write([]byte("Invalid Request, Account and Agreement need to be set"))
            return
        }
        if agreement.Account == "" || agreement.Agreement == "" {
            w.WriteHeader(http.StatusBadRequest)
            w.Write([]byte("Account and Agreement need to be set"))
            return
        }
    })
```

First up, we need to handle the request. `chi` has a nice way of doing this with its `render.Bind` function, which tries to bind the payload to a given json-struct. Then we validate that both the Account and Agreement are set and send back an error otherwise.

The next step is to create the Agreement on the smart-contract:

```go
    _, err := contract.CreateAgreement(&bind.TransactOpts{
        From:     auth.From,
        Signer:   auth.Signer,
        GasLimit: big.NewInt(200000),
        Value:    big.NewInt(0),
    }, agreement.Agreement, common.HexToAddress(agreement.Account))
    if err != nil {
        log.Fatalf("Failed to create agreement: %v", err)
    }
    fmt.Println("Agreement created: ", agreement.Agreement)
```

Basically, we just call the generated `CreateAgreement` method of our smart-contract with transaction options. We also need to convert the given Account from hex to an actual Ethereum Address. After this step, the Agreement is persisted on the blockchain.

Now to the issue of sending the transaction-fee to the User:

```go
    gasPrice, err := conn.SuggestGasPrice(context.Background())
    if err != nil {
        log.Fatalf("Failed to get gas price: %v", err)
    }
    signer := types.HomesteadSigner{}
    tx := types.NewTransaction(nonce, common.HexToAddress(agreement.Account), big.NewInt(100000), big.NewInt(21000), gasPrice, nil)
    signed, err := types.SignTx(tx, signer, privKey)
    if err != nil {
        log.Fatalf("Failed to sign transaction: %v", err)
    }
    err = conn.SendTransaction(context.Background(), signed)
    if err != nil {
        log.Fatalf("Failed to send transaction: %v", err)
    }
    fmt.Println("Transaction Fee sent to client!")
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
```

Here we just execute a normal transaction to the converted address of the User. It's a bit involved with the `SuggestGasPrice` call and the `HomesteadSigner`. This would be much easier with web3 and I'm not sure if this is the suggested way to do it with Go, but I couldn't find any other way to do it in the documentation.

Essentially, we create a new transaction, sending `100000` wei to the User. Then we sign the transaction and send it. Afterwards, if everything went well, we return `200 OK`.

That's it!

The full code for the example can be found [here](https://github.com/zupzup/eth-prepaid-transaction)

*Unfortunately, at the time of writing this, there seems to be a small [bug](https://github.com/ethereum/go-ethereum/issues/14909) working with `testrpc` from Go. I didn't dig into it very deeply, but it seems to have something to do with how the nonces are sent between `testrpc` and `go-ethereum`. I simply hard-coded the nonces to start from 0 and count them up manually for this PoC. This is not very pretty and will break awkwardly, when the Go server is restarted, but I didn't want to waste time, so there is some hackish nonce-updating-code in the example*

## Conclusion

The outlined concept won't work for a lot of applications, but for simple signing-use-cases, which are a good fit for blockchain platforms, it seems sufficient.

The solution works well and could, I believe, even serve as a basis for a real-world implementation of such a mechanism. Of course, there need to be some serious precautions, or even a manual process for sending out transaction fees to arbitrary users, but the general concept seems sound.

I'm curious how the blockchain space will deal with this issue in the future and which security and anti-fraud mechanisms will be provided by the platforms. What I am sure of today is that smart contract platforms will need mechanisms like self-paying contracts to shield users from the need to deal with cryptocurrency.

Please don't use this simplistic implementation for anything serious, as this will certainly end in tears, but maybe use it for inspiration or for learning in regards to the possibilities of interacting with the Ethereum blockchain using Go. :)

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
* [chi](https://github.com/go-chi/chi)
