
# What is Moonwell?

PoolTogether is a no-loss prize savings protocol that enables you to win by saving.

[Link to the contest](https://code4rena.com/audits/2023-07-moonwell#top)

## Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| [M-01](#M-01)     | Proposals which intend to send native tokens to target addresses can't be executed | [Link](https://github.com/code-423n4/2023-07-moonwell-findings/issues/268)  |


## <a id='M-01'></a>Proposals which intend to send native tokens to target addresses can't be executed 

 # Lines of code
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L237-L239 https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L400-L402

# Vulnerability details
## Impact
* In `TemporalGovernor` contract: any verified action approval (VAA)/proposal can be executed by anyone if it has been queued and passed the time delay.
* But if the proposal is intended to send native tokens to the target address; it will revert since `TemporalGovernor` contract doesn't have any balance, as there's no `receive()` or payable functions to receive the funds that will be sent to the proposal's target address.
* So any proposal with a value (decoded from `vm.payload`) will not be executed since the
  `target.call{value:value}(data)` will revert.

## Proof of Concept
* Code:
  [Line 237-239](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L237-L239)

```solidity
File: src/core/Governance/TemporalGovernor.sol
Line 237-239:
    function executeProposal(bytes memory VAA) public whenNotPaused {
        _executeProposal(VAA, false);
    }
```

[Line 400-402](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L400-L402)

```solidity
File: src/core/Governance/TemporalGovernor.sol
Line 400-402:
        (bool success, bytes memory returnData) = target.call{value: value}(
            data
        );
```

* Foundry PoC:

1. This test is copied from `testExecuteSucceeds` test in `TemporalGovernorExec.t.sol` file,and modified to demonstrate the issue; where a proposal is set to send an EOA receiverAddress a value of 1 ether, but will revert due to lack of funds (follow the comments in the test):

```solidity
 function testProposalWithValueExecutionFails() public {
        address receiverAddress = address(0x2);
        address[] memory targets = new address[](1);
        targets[0] = address(receiverAddress);

        uint256[] memory values = new uint256[](1);
        values[0] = 1 ether; // the proposal has a value of 1 eth to send it to the receiverAddress

        bytes[] memory payloads = new bytes[](1);

        payloads[0] = abi.encodeWithSignature("");

        /// to be unbundled by the temporal governor
        bytes memory payload = abi.encode(
            address(governor),
            targets,
            values,
            payloads
        );

        mockCore.setStorage(
            true,
            trustedChainid,
            governor.addressToBytes(admin),
            "reeeeeee",
            payload
        );
        governor.queueProposal("");

        bytes32 hash = keccak256(abi.encodePacked(""));
        (bool executed, uint248 queueTime) = governor.queuedTransactions(hash);

        assertEq(queueTime, block.timestamp);
        assertFalse(executed);
        //---executing the reposal
        vm.warp(block.timestamp + proposalDelay);

        //check that the balance of receiverAddress is zero before execution:
        assertEq(receiverAddress.balance, 0);

        // executeProposal function will revert due to lack of funds:
        vm.expectRevert();
        governor.executeProposal("");
        assertFalse(executed); // the proposal wasn't executed
        assertEq(receiverAddress.balance, 0); // the receiverAddress hasn't received any funds because the proposal execution has reverted due to lack of funds
    }
```

2. Test result:

```shell
$ forge test --match-test testProposalWithValueExecutionFails -vvv
Running 1 test for test/unit/TemporalGovernor/TemporalGovernorExec.t.sol:TemporalGovernorExecutionUnitTest
[PASS] testProposalWithValueExecutionFails() (gas: 458086)
Test result: ok. 1 passed; 0 failed; finished in 2.05ms
```

## Tools Used
Manual Testing & Foundry.

## Recommended Mitigation Steps
Add `receive()` function to the `TemporalGovernor` contract; so that it can receive funds (native tokens) to be sent with proposals.

## Assessed type
ETH-Transfer
