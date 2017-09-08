On the Ethereum blockchain, smart contracts are first-class citizens and because of their importance, [Solidity](https://solidity.readthedocs.io), the standard language for writing Ethereum smart contracts right now, has several ways of enabling contracts to interact with other contracts.

Smart contracts can call functions of other contracts and even create and deploy other contracts (e.g. issuing coins). There are several use-cases for this behaviour.

One of the most interesting use-cases are upgradeable contracts. Due to the immutable nature of the blockchain, it's not possible to change the code of a deployed smart-contract once it has been deployed. But by using a mechanism for delegating calls, a proxy can be deployed pointing (delegating function calls) to another contract, which holds the actual business logic.
Now it's possible to upgrade the functionality of the contract, by providing a different target address to the proxy contract, for example a newly deployed version with some bugfixes.

The same principle can be leveraged to use other contracts as libraries, thus reducing deployment costs, as the contract using the library doesn't need to include all of the code itself.

Another use-case is to simply use contracts as data stores of sort. For example one can strive to separate logic and data into different smart contracts. Now the logic-contract could be updated or swapped out via proxy, while retaining all the relevant state in the data-contract.

Being able to call and create contracts from contracts is a powerful concept and the following post provides a simple example of how to implement and test such behaviour using Solidity.

## The Contracts

First of all, we need two smart contracts to be able to test our interaction.

In this small example, we will create one `Callee` contract, which holds some state and gets called by the `Caller` contract.

There are several ways to delegate calls between contracts in Solidity. A deployed contract always resides at an `address` and this `address`-object in Solidity provides three methods to call other contracts:

* `call` - Execute code of another contract
* `delegatecall` - Execute code of another contract, but with the state(storage) of the *calling* contract.
* `callcode` - (deprecated)

It's also possible to provide gas and ether for a `call` invocation like this:

```javascript
someAddress.call.gas(1000000).value(1 ether)("register", "MyName");
```

The `delegatecall` method was a bugfix for `callcode`, which did not preserve `msg.sender` and `msg.value`, so `callcode` is deprecated and will be removed in the future.

It is important to note, that `delegatecall` involves a security-risk for the calling contract, as the called contract can access/manipulate the calling contracts storage.

There are, of course, inherent security risks with calling a function on a contract from a given address and this way of calling contracts breaks type-safety in Solidity. Therefore, `call`, `callcode` and `delegatecall` are supposed to be used only as a last resort.

Another way to call another contract is to use a mechanism similar to dependency-injection.



```javascript
pragma solidity ^0.4.6;

contract Callee {
    uint[] public values;

    function getValue(uint initial) returns(uint) {
        return initial + 150;
    }
    function storeValue(uint value) {
        values.push(value);
    }
    function getValues() returns(uint) {
        return values.length;
    }
}
```

```javascript
pragma solidity ^0.4.6;

contract Caller {
    function someAction(address addr) returns(uint) {
        Callee c = Callee(addr);
        return c.getValue(100);
    }
    
    function storeAction(address addr) returns(uint) {
        Callee c = Callee(addr);
        c.storeValue(100);
        return c.getValues();
    }
    
    function someUnsafeAction(address addr) {
        addr.call(bytes4(keccak256("storeValue(uint256)")), 100);
    }
}

contract Callee {
    function getValue(uint initialValue) returns(uint);
    function storeValue(uint value);
    function getValues() returns(uint);
}
```

## Testing the Interaction

An interaction like this can be tested in multiple ways. One way would be to [use web3 and testrpc](https://zupzup.org/smart-contract-solidity/), [Go](https://zupzup.org/eth-smart-contracts-go/) or, as I did in this case, simply use the [Remix Browser IDE](https://remix.ethereum.org).

Remix is a great tool for experimenting with smart-contracts - it comes with a nice editor and options to simulate deploying contracts and calling functions on them.

"0xdcb77b866fe07451e8f89871edb27b27af9f2afc"
TBD: screens

<center>
    <a href="images/simpleui.png" target="_blank"><img src="images/simpleui_thmb.png" /></a>
</center>

## Conclusion

nifty way to get around some of the limitations, but has it's own risks
any upgrade-mechanism is itself vulnerable to attacks

#### Resources

* [Official Solidity Docs](http://solidity.readthedocs.io/en/develop/index.html)
* [remix](https://remix.ethereum.org)
* [ethereum](https://ethereum.org/)
