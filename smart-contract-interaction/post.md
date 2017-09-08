TBD

## The Contracts

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

## Testing the Interaction

TBD

## Conclusion

TBD

#### Resources

* [Official Solidity Docs](http://solidity.readthedocs.io/en/develop/index.html)
* [testrpc](https://github.com/ethereumjs/testrpc)
* [remix](https://remix.ethereum.org)
* [ethereum](https://ethereum.org/)
