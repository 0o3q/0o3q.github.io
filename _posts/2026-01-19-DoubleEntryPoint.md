---
layout: post
title: "[Ethernaut] 26.DoubleEntryPoint"
tags: blockchain ethernaut foundry ERC20 DelegateERC20 Forta double_entry_point
---

# Analysis

---

The goal is to figure out where the bug is in `CryptoVault` and protect it from being drained of tokens. by implementing a `detection bot` and registering it in the `Forta` contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

The Forta Contract allows registering a bot to raise alerts. 

The CryptoVault Contract can sweep tokens if the specific `token` is not the `underlying`. Additionally, the code comments in the CryptoVault Contract are empty.

The LegacyToken Contract minted `LGT token` and delegated logic to the DoubleEntryPoint Contract.

The DoubleEntryPoint Contract minted `DET token` and can manage transfer delegation by utilizing a forta bot.

# Exploit

---

Inspecting the instance address on Etherscan reveals the addresses fot the LegactToken, CryptoValut and Forta Contract.

The Constructor arguments for the DoubleEntryPoint Contract are ordered as follows: LegacyToken, CryptoVault, Forta and player.

![Image etherscan]({{site.url}}/images/2026-01-19-DoubleEntryPoint/etherscan.png)

To verify the address checksum, use the following command:

```bash
cast --to-checksum-address <address>
```

We identify the vulnerability in the `CryptoVault` contract.

The `sweepToken()` function is publicly callable and verifies that the requested `token` is not the `underlying` . 

However, sweeping the LegacyToken(`LGT token`) bypasses this protection.

While the function validates the `LGT` address the actual transfer logic is delegated to the `DoubleEntryPoint` Contract, resulting in the unintended transfer of `DET Token` .

The `notify()` function in the Forta Contract executes `handleTransaction()`, which can be implemented with custom logic.

The `delegateTransfer()` function calls include the `origSender` argument. Therfore, by verifying if `origSender` matches the CryptoVault address, the bot can prevent `DET Token` from being drained.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/DoubleEntryPoint.sol";
import "forge-std/Script.sol";

contract Bot is IDetectionBot {
    Forta public forta_ = Forta(0x8d38A390959fAb2BF008bA6E49aF8Eb3D19E24d7);
    CryptoVault public cv_ = CryptoVault(0x1ee6F5EC79B4f94862B3839D7Eb661bAcf79f2e1);

    function handleTransaction(address user, bytes calldata msgData) external override {
        (,, address origSender) = abi.decode(msgData[4:], (address, uint256, address));
        if (origSender == address(cv_)) {
            forta_.raiseAlert(user);
        }

    }

}

contract DoubleEntryPointSol is Script {
    Forta public forta_ = Forta(0x8d38A390959fAb2BF008bA6E49aF8Eb3D19E24d7);
    CryptoVault public cv_ = CryptoVault(0x1ee6F5EC79B4f94862B3839D7Eb661bAcf79f2e1);
    LegacyToken public lt_ = LegacyToken(0x3E847B9aE7613A08c1d8c8E52212e119903312CA);

    function run() external {
        vm.startBroadcast();

        Bot bot = Bot(address(new Bot()));
        forta_.setDetectionBot(address(bot));

        vm.stopBroadcast();
    }
}
```