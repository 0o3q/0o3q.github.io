---
layout: post
title: "[Ethernaut] MagicNumber"
tags: blockchain ethernaut foundry assembly bytecode
---

# Analysis

---

The goal is

1.  provide the Ethernaut with a `Solver`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
    */
}
```

We respond to the `whatIsTheMeaningOfLife()` function with the right 32 byte number.

The number **42** is suspected to be the magic value expected by `whatIsTheMeaningOfLife()`.

```nasm
PUSH1 0x2a // Push 42(0x42) onto the stack => 602a
PUSH1 0x00 // Push memory offset 0x00 => 6000
MSTORE // MSTORE(0x00: offset, 0x2a: value) Store 42 at memory[0x00]=> 52
PUSH1 0x20 // Push size (32 bytes) => 6020
PUSH1 0x00 // Push memory offset 0x00 => 6000
RETURN // Return memory[0x00:0x20] => f3
```

The [bytecode](https://www.evm.codes/) `0x602a60005260206000f3`  returns right 32 byte number:

```nasm
0x000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002a 
```

when `whatIsTheMeaningOfLife()` function is called.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/MagicNumber.sol";
import "forge-std/Script.sol";

contract Ex {
    constructor() {
        assembly {
            mstore(0, 0x602a60005260206000f3)
            return(0x16, 0x0a)
        }
    }
}

contract MagicNumberSol is Script {
    MagicNum public mn_ = MagicNum(0x20416BDe7653DaC534a51C744D7f572Bfd22aC76);

    function run() external {
        vm.startBroadcast();

        console.log("Before solver: ", mn_.solver());
        Ex ex = new Ex();
        mn_.setSolver(address(ex));
        console.log("After solver: ", mn_.solver());

        vm.stopBroadcast();
    }
}
```

When the command `mstore(0, 0x602a60005260206000f3)` is executed, the memory state becomes:

```nasm
0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000602a60005260206000f3
```

After that, `return(0x16, 0x0a)` is executed, returning only the last 10 bytes (the constructor code), which is:

```nasm
0x602a60005260206000f3
```