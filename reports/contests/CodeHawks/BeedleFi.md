# What is BeedleFi?

Oracle free peer to peer perpetual lending protocol.

[Link to the contest](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| [M-01](#M-01)     | Possible loss of ownership | [Link](https://github.com/Cyfrin/2023-07-beedle/issues/101)  |

## <a id='M-01'></a>M-01. Possible loss of ownership            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Ownable.sol#L19C4-L22C6

## Summary

Possible loss of ownership.

See [here](https://solodit.xyz/issues/possible-loss-of-ownership-halborn-nftfi-bundlesairdrop-pdf) a reference for this exact issue.

## Vulnerability Details

When transferring the ownership of the protocol, no checks are performed
on whether the new address is valid and active.

```solidity
19:    function transferOwnership(address _owner) public virtual onlyOwner {
20:       owner = _owner;
21:       emit OwnershipTransferred(msg.sender, _owner);
22:    }
```
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Ownable.sol#L19C4-L22C6

## Impact

In case there is a mistake
when transferring the ownership, the whole protocol is locked out of its
permissioned functionalities.

## Tools Used

Manual review

## Recommendations

The transfer of ownership process should be divided into two separate transactions. The first transaction involves calling the `requestTransferOwnership` function to propose a new owner for the protocol. The second transaction requires the new owner to accept the proposal by calling the `acceptsTransferOwnership` function. This approach ensures a secure and controlled transfer of ownership for the protocol.
