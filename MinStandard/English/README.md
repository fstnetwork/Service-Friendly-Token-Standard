---
eip: <to be assigned>
title: Service-Friendly Token Standard
author: Atkins Chang <atkins@fstk.io> (@AtkinsChang), Noel Bao <noel@fstk.io> (@noeleon930), Jack Chu <jack@fstk.io>, Leo Chou <leo@fstk.io>, Darren Goh <darren@fstk.io>
discussions-to: <URL>
status: Draft
type: Standard
category: ERC
created: 2018-08-08
requires: 20
---

# Service-Friendly Token Standard

## Simple Summary

<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->
<!-- A standard interface for service-friendly tokens, which aims for a grounded代幣化environment for business. -->

This Token standard is designed to allow Tokens to interact with service-based smart contracts and off-chain services seamlessly and without friction, providing a friendly environment for the Tokens.

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

Originally designed to be a crowdfunding tool, Tokens now have a painful period of transition to being a Utility Token, to provide their service to the token holders. Many projects and companies lack sufficient Smart Contract functionalities in their Tokens, which makes it difficult to support the fundamentals of their business venture and to apply it to real-world services and products.

The following interface designs are based on the fundamental features and aspects of a **Robust Tokenisation** that businesses need. This including removing the difficulties of secure bindings among smart contracts and on-chains and off-chains integration. We, FundersToken based on our experience, understanding of the Token Standards that are available in the market, FundersToken have made several improvements on the Token Standards aiming to build a Native Token Environment, a friendly environment for Tokens.

FundersToken have also developed a **Token transfer relay**, which simulates blockchains in the form of smart contracts for the Tokens, and releases end-users from the need and limitation of only using Ether as transaction fee (gas fee) when making a token transfer.

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

We categorise this interface standard to the following:

1.  [The improvements made on ERC-20](#erc-20-補強)
2.  [The improvements made to make a Token service-friendly](#service-friendly-服務友善化-補強)
3.  [The improvements for Robust Tokenisation](#tokenisation-代幣化-補強)

As the most basic and most common way of controlling and storing Tokens, ERC-20 has proved to be a feasible and viable direction, however due to different implementations, such as gas consumption and mathematical safety of execution, many Tokens have suffered abuse and denial-of-service that led to financial loss.

We have make optimisation and made mathematical checks for the implementation of `transfer` and `approve`, and how to store `balance` and `allowance` efficiently.

---

To define a service-friendly environment, we must first identify the design goals of the payment flow and the smart contracts on the Ethereum Blockchain.

The payment flow of an Ethereum transaction:

```mermaid
graph LR

EA((EA))
CA1((CA 1))
CA2((CA 2))
CA3((CA 3))
A((A))

EA  --- Z1[tx]
Z1  --> CA1
CA1 --- Z2[msg]
Z2  --> CA2
CA2 --- Z3[msg]
Z3  --> CA3
CA3 -.- Z4[msg]
Z4  -.-> A
```

The payment flow of a ERC20 token transaction:

```mermaid
graph LR

EA((EA))
A1((A))

EA --- Z1[transfer]
Z1 --> A1

```

or

```mermaid
graph LR

EA1((EA))
CA1((CA))

EA2((EA))
CA2((CA))

EA1 --- Z1[approve]
Z1  --> CA1

EA2 --- Z2[call]
Z2  --> CA2
CA2 --- Z3[transferFrom EA]
Z3  --> A1((A))

```

> EA represents External Account  
> CA represents Contract Account  
>  A represents EA and CA

Most of the current Token standards have difficulties to compose multiple continuous processes in one Ethereum transaction, in additional the transaction must be triggered after the process of `approve` is done, this process is also in risk to be attacked by other smart contracts, by deliberately consuming the `allowance` more than the intended consumption.

From the statement above, we can see the Tokens are less direct and dynamic compared to Ether. Since Tokens are driven by smart contracts, Tokens must follow the execution process of the Ethereum transaction, which means the recipient address of a Token transfer transaction is the Token smart contract instead of `to` in `transfer`. The process and implementation of the Token `transfer` is not intuitive as Ether's transfer.

And because the Token ledger is within the smart contract of the token, any mutation to the ledger (the balance and the allowance) or the logics must be designed and wrapped in the Token smart contract. Otherwise, the Token smart contract has to authorize or approve external smart contracts to extend the logic that is related to the ledger. But the former option slows down the development cycle, the latter option will increase the execution cost and security risks.

We had experienced the inconvenience during the development of smart contract module and providing modularisation services. Our goal is to make the Token payment flow described  below:

```
(EA) --[transfer and call]-> (CA 1)
     --[transfer and call]-> (CA 2)
     --[transfer and call]-> (CA 3)
     ...
     ...
     --[transfer and call]-> (A)
```

In short, we hope to make payment flow and execution flow of the Tokens are as natural as Ether's, and make the services provided by the Tokens more direct, intuitive and easier to develop, instead of setting back the business due to the inconvenience of ERC-20 Token standard.

To achieve this goal, we have improved the `transferAndCall` in ERC-223 and ERC-827, and ensure the `receiverContract` (the Service smart contract) always gets the real `value` and the real `from` (the origin of the Token transfer), and make the `receiverContract` unable to attack the `from`. We will explain this in detail later.

Moreover, what was mentioned above is not only to increase the consistency and the linking flexibility among the service-based smart contracts, making the business logic and the payment flow more modularised and secured, but to make on-chain-off-chain integrations more complete and more consistent, reducing the needs of status checking or multi-phase commit development, encouraging more developers' adoption.

---

As the improvement made for tokenisation, it is based after our service-friendly Token, and we took a step further to conduct important features for businesses such as CRM functionality and Token relay to achieve a De-Ether environment.

The important feature for CRM is compacting multiple Token transfers and making the process as light and predictable as possible, which allows businesses to have more flexibility for CRM applications.

The Token relay is to remove the biggest technical barrier, which is the need for end-users to pay Ether in a Token transfer as the transaction fee.  
If the situation is in a context of  that Ethereum is a decentralised computing platform and cash platform, and to execute smart contracts, the end-users must pay Ether to stablise the Ethereum network and incentivise the miners to sustain the network, then it's very rational and acceptable to everyone.  
But if it is in the context of Tokens, it becomes illogical and cloggy.

The idea of "No Ether, No Token usages" obstructs the utility of tokenisation.  
So we have implemented a feature that allows the origin of a token transfer to sign a specific **Token transfer request**, and the **Relayers** check its transfer fee (in Token) and the signature then the Relayers relay the request by sending the request to the Token smart contract, which also means the Relayers pay the ETH transaction gas for the request.  
Then the Token smart contract checks the relayed transfers and avoid any attack among transfer origin, relayers and the receivers.

Further details are in the next section.

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### The improvements to ERC-20

Index:

1. [Extension to `address` and `uint256`]
2. [Immutable Token basic info]
3. [Optimised Token holders' storage]
4. [Mutable Token basic info]
5. [Events of the Token]
6. [Operation functions of the Token]
7. [More secure Token]

---

#### Extension to `address` and `uint256`:

We extended the methods in type `address` and `uint256`

<details><summary>AddressExtension Soucre Code</summary>

```
pragma solidity ^0.4.24;
pragma experimental "v0.5.0";
pragma experimental ABIEncoderV2;

library AddressExtension {

  function isValid(address _address) internal pure returns (bool) {
    return 0 != _address;
  }

  function isAccount(address _address) internal view returns (bool result) {
    assembly {
      result := iszero(extcodesize(_address))
    }
  }

  function toBytes(address _address) internal pure returns (bytes b) {
   assembly {
      let m := mload(0x40)
      mstore(add(m, 20), xor(0x140000000000000000000000000000000000000000, _address))
      mstore(0x40, add(m, 52))
      b := m
    }
  }
}
```

</details>

<details><summary>Math Soucre Code</summary>

```
pragma solidity ^0.4.24;
pragma experimental "v0.5.0";
pragma experimental ABIEncoderV2;

library Math {

  struct Fraction {
    uint256 numerator;
    uint256 denominator;
  }

  function isPositive(Fraction memory fraction) internal pure returns (bool) {
    return fraction.numerator > 0 && fraction.denominator > 0;
  }

  function mul(uint256 a, uint256 b) internal pure returns (uint256 r) {
    r = a * b;
    require((a == 0) || (r / a == b));
  }

  function div(uint256 a, uint256 b) internal pure returns (uint256 r) {
    r = a / b;
  }

  function sub(uint256 a, uint256 b) internal pure returns (uint256 r) {
    require((r = a - b) <= a);
  }

  function add(uint256 a, uint256 b) internal pure returns (uint256 r) {
    require((r = a + b) >= a);
  }

  function min(uint256 x, uint256 y) internal pure returns (uint256 r) {
    return x <= y ? x : y;
  }

  function max(uint256 x, uint256 y) internal pure returns (uint256 r) {
    return x >= y ? x : y;
  }

  function mulDiv(uint256 value, uint256 m, uint256 d) internal pure returns (uint256 r) {
    // try mul
    r = value * m;
    if (r / value == m) {
      // if mul not overflow
      r /= d;
    } else {
      // else div first
      r = mul(value / d, m);
    }
  }

  function mulDivCeil(uint256 value, uint256 m, uint256 d) internal pure returns (uint256 r) {
    // try mul
    r = value * m;
    if (r / value == m) {
      // mul not overflow
      if (r % d == 0) {
        r /= d;
      } else {
        r = (r / d) + 1;
      }
    } else {
      // mul overflow then div first
      r = mul(value / d, m);
      if (value % d != 0) {
        r += 1;
      }
    }
  }

  function mul(uint256 x, Fraction memory f) internal pure returns (uint256) {
    return mulDiv(x, f.numerator, f.denominator);
  }

  function mulCeil(uint256 x, Fraction memory f) internal pure returns (uint256) {
    return mulDivCeil(x, f.numerator, f.denominator);
  }

  function div(uint256 x, Fraction memory f) internal pure returns (uint256) {
    return mulDiv(x, f.denominator, f.numerator);
  }

  function divCeil(uint256 x, Fraction memory f) internal pure returns (uint256) {
    return mulDivCeil(x, f.denominator, f.numerator);
  }

  function mul(Fraction memory x, Fraction memory y) internal pure returns (Math.Fraction) {
    return Math.Fraction({
      numerator: mul(x.numerator, y.numerator),
      denominator: mul(x.denominator, y.denominator)
    });
  }
}
```

</details>

---

#### Immutable Token basic info:

- `string name` The Token name.
- `string symbol` The Token symbol.
- `uint8 decimals` The decimals of stored number in ledger.

```
string public constant name;
string public constant symbol;
uint8 public constant decimals;
```

---

#### Optimised Token holders' storage:

In `Account`,

- `uint256 balance` is the balance of the Token holder.
- `uint256 nonce` is the transfer count of the Token holder, avoiding double spending, but only used in Token relay, the details are published in [Tokenisation section]().
- `mapping (address => Instrument) instruments` is the storage that stores the data among Token holders.

In `Instrument` ,

- `uint256 allowance` is the quota of Tokens that the Token holder that have authorised other holders to have access to.

_`DirectDebit directDebit` shows the detail of agreement which the Token holder allows withdrawal by other holders on a particular date in a designated time-frame._

_`DirectDebit` is a demonstration of data between token holders that can be placed in Instrument. It is not included in this standard, but it is included in the full version of Service-Friendly Token Standard._

```
struct Instrument {
  uint256 allowance;
  DirectDebit directDebit;
}

struct Account {
  uint256 balance;
  uint256 nonce;
  mapping (address => Instrument) instruments;
}

mapping(address => Account) internal accounts;
```

---

#### Mutable Token basic info:

- `totalSupply()` is the total supply of token issued.
- `balanceOf(address)` allows token holders to check their total balance.
- `allowance(address,address)` allows token holders to check the quota of Tokens that the Token holder that have authorised other holders to have access to.
- `address issuer` is the address of the Token issuer，although this is not a requirement for the ERC-20 standard, however this information is required for inspection in several actions.

```
function totalSupply () public view returns (uint256);

function balanceOf(address owner) public view returns (uint256) {
  return accounts[owner].balance;
}

function allowance(address owner, address spender) public view returns (uint256) {
  return accounts[owner].instruments[spender].allowance;
}

address public issuer;
```

---

#### Events of the Token:

- `Transfer(address,address,uint256)` is the event that triggers any change in the amount of token.
- `Approval(address,address,uint256)` is the event where Token holder authorize access to different Token holders.

```
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);
```

Also, to support Blockchain Explorers such as Etherscan, when the Token is first deployed, the `Transfer` event would be emitted.

```
emit Transfer(address(0), <tokenIssuer>, <totalSupply>);
```

---

#### Operation functions of the Token:

The following interface is the same as the ERC-20, except for the math-related operations, especially during subtraction, it will be operated in `Math` extension, and by loading the mapping of `accounts` to reduce the number of mapping related operations.

```
function transfer(address to, uint256 value) public returns (bool) {
  Account storage senderAccount = accounts[msg.sender];

  // guarded by Math
  senderAccount.balance = senderAccount.balance.sub(value);
  // guarded by totalSupply
  accounts[to].balance += value;

  emit Transfer(msg.sender, to, value);

  return true;
}

function transferFrom(address from, address to, uint256 value) public returns(bool) {
  Account storage fromAccount = accounts[from];
  Instrument storage senderInstrument = fromAccount.instruments[msg.sender];

  // guarded by Math
  fromAccount.balance = fromAccount.balance.sub(value);
  // guarded by Math
  senderInstrument.allowance = senderInstrument.allowance.sub(value);
  // guarded by totalSupply
  accounts[to].balance += value;

  emit Transfer(from, to, value);

  return true;
}

function approve(address spender, uint256 value) public returns (bool) {
  Instrument storage spenderInstrument = accounts[msg.sender].instruments[spender];

  if (erc20ApproveChecking) {
    require((value == 0) || (spenderInstrument.allowance == 0));
  }

  emit Approval(
    msg.sender,
    spender,
    spenderInstrument.allowance = value
  );

  return true;
}
```

Please refer `erc20ApproveChecking` from `approve(address,uint256)` in the next section

When `erc20ApproveChecking` is `true`，`approve(address,uint256)` will do extra checks that if the `allowance` of `spender` is 0 before this call，to prevent front-running attack by `spender`.

---

#### More secure Token:

- `bool erc20ApproveChecking` 為一個狀態值紀錄是否要開啟更安全的 `approve` 相關執行檢查，預設為 `false`，只有 `issuer` 才能更動
- `SetERC20ApproveChecking(bool)` 為 `erc20ApproveChecking` 改變時會發射的事件，需要透過 `setERC20ApproveChecking(bool)` 引發
- `approve(address,uint256,uint256)` 會要求代幣擁有者輸入預期的 `allowance`，通過驗證才能繼續改變 `allowance`
- `increaseAllowance(address,uint256)` 可直接增加 `allowance`
- `decreaseAllowance(address,uint256,bool)` 可直接減少 `allowance`，而當 `strict` 為 `true` 時，會用 `Math` 進行減法檢查
- `spendableAllowance(address,address)` 可直接得知被允許之帳戶可以實際上消耗多少額度

<details><summary>Secure ERC20 Approve Checking Soucre Code</summary>

```
bool public erc20ApproveChecking;

event SetERC20ApproveChecking(bool approveChecking);

function setERC20ApproveChecking(bool approveChecking) public {
  require(msg.sender == issuer);
  emit SetERC20ApproveChecking(erc20ApproveChecking = approveChecking);
}

function approve(address spender, uint256 expectedValue, uint256 newValue) public returns (bool) {
  Instrument storage spenderInstrument = accounts[msg.sender].instruments[spender];
  require(spenderInstrument.allowance == expectedValue);

  emit Approval(
    msg.sender,
    spender,
    spenderInstrument.allowance = newValue
  );

  return true;
}

function increaseAllowance(address spender, uint256 value) public returns (bool) {
  Instrument storage spenderInstrument = accounts[msg.sender].instruments[spender];

  emit Approval(
    msg.sender,
    spender,
    // guarded by Math
    spenderInstrument.allowance = spenderInstrument.allowance.add(value)
  );

  return true;
}

function decreaseAllowance(address spender, uint256 value, bool strict) public returns (bool) {
  Instrument storage spenderInstrument = accounts[msg.sender].instruments[spender];

  uint256 currentValue = spenderInstrument.allowance;
  uint256 newValue;
  if (strict) {
    // guarded by Math
    newValue = currentValue.sub(value);
  } else if (value < currentValue) {
    // guarded by if
    newValue = currentValue - value;
  }

  emit Approval(
    msg.sender,
    spender,
    spenderInstrument.allowance = newValue
  );

  return true;
}

function spendableAllowance(address owner, address spender) public view returns (uint256) {
  Account storage ownerAccount = accounts[owner];
  return Math.min(
    ownerAccount.instruments[spender].allowance,
    ownerAccount.balance
  );
}
```

</details>

### The improvements made to make a Token service-friendly

索引:

1. [Transfer and call (傳送呼叫)](#transfer-and-call-傳送呼叫)

---

#### Transfer and call (傳送呼叫):

為了讓傳送代幣與呼叫接收者智能合約 (receiverContract) 是一氣呵成，能讓這些呼叫可以連續地一個串一個串下去，並且同時也讓接收者智能合約可以得到真正的 `value` 與 `msg.sender`，對於參數的檢查與覆蓋就會變得非常嚴格

在 `transferAndCall(address,uint256,bytes)` 的參數中

- `address to` 為接收者智能合約的位址
- `uint256 value` 為代幣傳送量，與 `transfer` 的一樣意義
- `bytes data` 為後續所有連續動作都需要的參數資料，與 `to.call(data)` 搭配使用，`data` 其中應內含 `signature`、 `value` 與 `msg.sender`

並且因為 `data` 最少要包含要傳遞給接收者智能合約的資料，故長度至少為 **4 bytes signature + 32 bytes value + 32 bytes sender** = **68 bytes**

也會進行下列檢查

- 禁止 `to` 為合約本身
- 檢查 `data` 的長度需大於等於 68 bytes
- 檢查確定代幣傳送已經完成

以及對於 data 的前兩個參數進行強制覆蓋，讓 `data` 中必定是

```
[4 bytes signature][32 bytes value][32 bytes msg.sender][其他原先的資料們]
```

故意讓 `uint256 value` 先而 `address sender` (`address from`) 後的原因為，不與 `address to` + `uint256 value` 的組合順序搞混

```
// Token Contract (TokenA, decimals = 18)

function transferAndCall(
  address to,
  uint256 value,
  bytes data
)
  public
  payable
  returns (bool)
{
  require(
    to != address(this) &&
    data.length >= 68 &&
    transfer(to, value)
  );
  assembly {
      mstore(add(data, 36), value)  // 32 (length) + 4 (signature)
      mstore(add(data, 68), caller) // 32 (length) + 4 (signature) + 32 (1st arg)
  }
  require(to.call.value(msg.value)(data));
  return true;
}
```

以及，接收者智能合約的函數就需要配合前兩個參數為 `uint256 value` 以及 `address from`  
範例:

```
// Receiver Contract (Vendor machine, sells TokenB)

function purchase(
  uint256 value,
  address from
)
  public
  payable
  returns (bool)
{
  require(msg.sender == address(TokenA));

  return TokenB.transfer(from, calculateAmount(value));
}
```

所以要使終端使用者可以用 100 TokenA 購買 TokenB 時，只要能編碼下列 tx input，簽署並送出即可

假設 `msg.sender` (`from`) 為 `0x83b21dbd0e60b9709d647de183f5ae0c31b54c2a`，也假設接收者智能合約 (VendorMachine) 為 `0x1234567890123456789012345678901234567890`

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000083b21dbd0e60b9709d647de183f5ae0c31b54c2a0000000000000000000000000000000000000000000000056bc75e2d63100000");
```

或者擺隨意的 bytes 在後面，但 signature 不能影響到 ( `"0x" + keccak256("purchase(uint256,address)")[0~7]` = `0xae77c237` )

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001");
```

### The improvements for Robust Tokenisation

索引:

1. [一次性多個傳送代幣](#一次性多個傳送代幣)
2. [代幣傳送委派、代幣轉發](#代幣傳送委派代幣轉發)

---

#### 一次性多個傳送代幣

- `transfer(uint256[])` 為一次性傳送代幣給多個對象時所作的操作
- `transfer(uint256[])` 中的參數 `uint256[] data` 內容是各元素為 **20 bytes receiverAddress + 12 bytes value** 的 `uint256` 數字的不限長度陣列

為減少所需要帶上的參數，我們將接收者位址 (receviers) 跟 代幣傳送量 (values) 合在了一起，在一個 32 bytes 的 `uint256` 數字裡面就能紀錄接收者地址與代幣傳送量

只不過 12 bytes 能紀錄的量對於 decimals = 18 的代幣而言，就不能傳太過於大的數字了，但至少對每個接收者也有 79,228,162,514.264337593543950335 個代幣可傳 (已經經過了 decimals 的處理以便人類理解))，以一個健康的代幣而言已經超越總代幣發行量，也就是 `0xffffffffffffffffffffffff / (10 ** 18)`

並且為了正確顯示在區塊鏈瀏覽器上，必須每個 `Transfer(address,address,uint256)` 事件都要射出，然後也為了優化 storage 的讀寫，最後才會將所花上的餘額寫入代幣擁有者的餘額中

```
function transfer(uint256[] data) public returns (bool) {
  Account storage senderAccount = accounts[msg.sender];
  uint256 totalValue;

  for (uint256 i = 0; i < data.length; i++) {
    address receiver = address(data[i] >> 96);
    uint256 value = data[i] & 0xffffffffffffffffffffffff;

    totalValue = totalValue.add(value);
    accounts[receiver].balance += value;

    emit Transfer(msg.sender, receiver, value);
  }

  senderAccount.balance = senderAccount.balance.sub(totalValue);

  return true;
}
```

#### 代幣傳送委派、代幣轉發:

此為代幣化關鍵的一個介面，讓代幣的傳送不再需要以太幣當作手續費，而是以也付代幣當作手續費的轉變

代幣轉發也是一個可以開啟或關閉的功能:

- `bool isDelegateEnable` 為一個狀態值紀錄是否要開啟代幣轉發功能，預設為 `false`，只有 `issuer` 才能更動
- `SetDelegate(bool)` 為 `isDelegateEnable` 改變時會發射的事件，需要透過 `setDelegate(bool)` 引發

```
bool public isDelegateEnable;

event SetDelegate(bool isDelegateEnable);

function setDelegate(bool delegate) public {
  require(msg.sender == issuer);
  emit setDelegate(isDelegateEnable = delegate);
}
```

---

在 `delegateTransferAndCall(uint256,uint256,uint256,address,uint256,bytes,uint8,uint8,bytes32,bytes32)` 中

- `uint256 nonce` 代表此被委派的傳送是第幾個傳送，這是為了防止雙花攻擊
- `uint256 fee` 代表代幣傳送者 (Token transfer origin) 願意給轉發者 (Relayer) 多少代幣當作手續費
- `uint256 gasAmount` 代表代幣傳送者指定的以太坊燃料量，使轉發者可以事先檢查並且不受浪費攻擊
- `address to` 代表代幣傳送的接收者地址，可以為智能合約地址
- `uint256 value` 代幣傳送量，與 `transfer(address,uint256)` 中的 `value` 的意義一樣
- `bytes data` 與 `transferAndCall(address,uint256,bytes)` 中的 `data` 的意義一樣
- `DelegateMode mode` 代表為代幣傳送者想要指定轉發者及指定誰可收取 `fee` 的委派模式
- `uint8 v` 為證明代幣傳送者簽署上述參數的簽章 (ECDSA signature) 中的 `v`
- `bytes32 r` 為證明代幣傳送者簽署上述參數的簽章 (ECDSA signature) 中的 `r`
- `bytes32 s` 為證明代幣傳送者簽署上述參數的簽章 (ECDSA signature) 中的 `s`

代幣傳送者需要在鏈下先編織好以上的資訊並且簽署才能將參數們交給轉發者，鏈外部份的實作將會以參考的方式補充進來，而非有硬性要求

資料的簽署方法為

```
ECDSA_Sign(
  keccak256(
    abi.encodePacked(
      tokenAddress,
      nonce,
      fee,
      gasAmount,
      to,
      value,
      data,
      mode,
      relayerAddress (or 0)
    )
  )
)
```

`DelegateMode` 則有以下幾種:

- `PublicMsgSender` 代表是任何人都可以是轉發者，並且`fee` 是將給 `msg.sender`
- `PublicTxOrigin` 代表是任何人都可以是轉發者，並且`fee` 是將給 `tx.origin`
- `PrivateMsgSender` 代表是代幣傳送者指定了轉發者，並且`fee` 是將給 `msg.sender`
- `PrivateTxOrigin` 代表是代幣傳送者指定了轉發者，並且`fee` 是將給 `tx.origin`

<details><summary>DelegateTransferAndCall Soucre Code</summary>

```
function delegateTransferAndCall(
  uint256 nonce,
  uint256 fee,
  uint256 gasAmount,
  address to,
  uint256 value,
  bytes data,
  DelegateMode mode,
  uint8 v,
  bytes32 r,
  bytes32 s
)
  public
  returns (bool)
{
  require(isDelegateEnable);

  require(to != address(this));
  address signer;
  address relayer;
  if (mode == DelegateMode.PublicMsgSender) {
    signer = ecrecover(
      keccak256(abi.encodePacked(this, nonce, fee, gasAmount, to, value, data, mode, address(0))),
      v,
      r,
      s
    );
    relayer = msg.sender;
  } else if (mode == DelegateMode.PublicTxOrigin) {
    signer = ecrecover(
      keccak256(abi.encodePacked(this, nonce, fee, gasAmount, to, value, data, mode, address(0))),
      v,
      r,
      s
    );
    relayer = tx.origin;
  } else if (mode == DelegateMode.PrivateMsgSender) {
    signer = ecrecover(
      keccak256(abi.encodePacked(this, nonce, fee, gasAmount, to, value, data, mode, msg.sender)),
      v,
      r,
      s
    );
    relayer = msg.sender;
  } else if (mode == DelegateMode.PrivateTxOrigin) {
    signer = ecrecover(
      keccak256(abi.encodePacked(this, nonce, fee, gasAmount, to, value, data, mode, tx.origin)),
      v,
      r,
      s
    );
    relayer = tx.origin;
  } else {
    revert();
  }

  Account storage signerAccount = accounts[signer];
  // nonce
  require(nonce == signerAccount.nonce);
  emit IncreaseNonce(signer, signerAccount.nonce += 1);

  // guarded by Math
  signerAccount.balance = signerAccount.balance.sub(value.add(fee));
  // guarded by totalSupply
  accounts[to].balance += value;
  // guarded by totalSupply
  if (fee != 0) {
    accounts[relayer].balance += fee;
    emit Transfer(signer, relayer, fee);
  }

  if (!to.isAccount() && data.length >= 68) {
    assembly {
      mstore(add(data, 36), value)  // 32 (length) + 4 (signature)
      mstore(add(data, 68), signer) // 32 (length) + 4 (signature) + 32 (1st arg)
    }
    if (to.call.gas(gasAmount)(data)) {
      emit Transfer(signer, to, value);
    } else {
      signerAccount.balance += value;
      accounts[to].balance -= value;
    }
  } else {
    emit Transfer(signer, to, value);
  }

  return true;
}
```

</details>

---

查看 nonce:

- `nonceOf(address)` 可查找任何帳戶的 nonce

```
function nonceOf(address owner) public view returns (uint256) {
  return accounts[owner].nonce;
}
```

而因代幣傳送者要有備援方案針對誤發出去的代幣傳送請求 (Token transfer request) 進行補救，進行強制覆蓋，故須 nonce 方面的操作

- `IncreaseNonce(address,uint256)` 為 nonce 增加時所發射的事件，唯有 `delegateTransferAndCall()` 與 `increaseNonce()` 觸發
- `increaseNonce()` 為代幣傳送者手動增加 nonce 之操作

```
event IncreaseNonce(address indexed from, uint256 nonce);

function increaseNonce() public returns (bool) {
  emit IncreaseNonce(msg.sender, accounts[msg.sender].nonce += 1);
}
```

## Rationale

<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

We will summarise and add in the details in the following.

### The improvements made on ERC-20

As ERC-20 only specifies the interface and has lack the requirements and suggestion for real-world practices. While building on top of the ERC-20 standard, we have encountered a few challenges and after finding the solution, we decided to infuse our solution into this standard. For entreprise users and Token end-users, a vivid operating logic and low-cost is a must, with that in mind, we have implemented several optimisation for storage. Taking `accounts` as an example, we have changed the mapping of accounts to only read the struct of the reference and only to take the necessary data, by doing this, the process would be lighter compared to read the all accounts-related mappings.

We have also made improvements on security, specifically on preventing front-running attack by the `spender`, when the token holder is assigning `allowance`. If this process is not checked, the process of assigning `allowance` from 1,000 to 500, in a front-running attack scenario, the attacker would first spend 1,000 that was previously assigned, and as the token holder  approve another 500, making the `allowance`: **1,000 -> 0 -> 500**.

In a safer environment, the requirement before approve a new value, at least under the ERC-20 interface, the `allowance` would return to 0. While this mechanism makes it safer, but this would mean that 2 Ethereum transactions would be needed. However we would suggest to utilize the interface of `expectedValue`, as both `increaseAllowance` and `decreaseAllowance` are in risk of front-running attack.

### The improvements made to make a Token service-friendly

First of all, we intentionally named the function as `transferAndCall`，instead of function overload, that is because the definition of `transfer` is from one account to another account, sometimes even a smart contract, which is logical, as some kinds of the procedures must first sent to the smart contract, awaiting for the next step (the next transaction).

However if we prohibit ERC-20 `transfer` just because of too many tokens are sent to a smart contract that are unable to transfer the token, it would be putting the cart before the horse. As this is not the fault of ERC-20 but the fault of the one who transfers the tokens and the service provider.

We intentionally used a different terms to express different functionality, most importantly different aims and different motives. Because we wanted to make Token-related transaction and transfer process to be intuitive and easy to plan, we have allow the mechanism for receiving smart contract to perform continuous actions and to replace the original calldata with the correct calldata by force, thus no one would be able to abuse any resources.

Imagine a single Ethereum transaction to perform a continuous `transferAndCall`, it is similar to putting a complex payment flow on chain, complex yet stream-lined procedure, and can be extensive, scalable by will, to achieve a service-friendly modularisation for Tokens.

This is why this section is called **Service-Friendly**, because a Service-Friendly smart contract is easy to use, trustworthy, also to be able to integrate the business logics off-chain, to be able to do all of these on Ethereum, that’d be a blessing.

### The improvements for Robust Tokenisation

The last modification that we did to improve the flow of payment is to removing the need of paying Ether to make a token transfer. Although the concept of **Account Abstraction** is set to release in the near future, however in the concept of Account Abstraction, the service provider is the one who carries the burden of paying Ether, and this action will bring hard-cost to the business, making it hard to scale.

A **volunteer**, also known as a **Relayer** must be able to voluntarily pick a normal, non-error token transfer request and relay it to the Ethereum blockchain, then via smart contract to confirm and check what object was specified and what modification was made by the relayer, just to make sure the origin of the token transfer won't be able to attack the relayer and the relayer won't be able to attack the origin of the token transfer.    

The list below shows the potential attack and the solution for the attacks:

1. Relayer repeatedly sends token transfer request or broadcast a wrong token transfer to the token smart contract.
   > This can be solved by adding in `nonce` or manually increase the `nonce` (`increaseNonce`()). Even when the token transfer request has failed，`nonce` will increase regardless.
2. Relayer took the token transfer request and send it to another Token smart contract with the same `nonce`
   > Can be solved by adding data of `tokenAddress` into signature
3. The origin of the token transfer wasting Relayer's Ethereum Gas fee.
   > This can be solved by adding the data for `gasAmount` into the signature, in a scenario where the origin of the token transfer provides a very small incentive to the relayer, the relayer will be able to see it from the start, hence will be able to drop the request.
   > And also, to make sure this type of attack will inevitably fail, and to ensure relayer to get the `fee` for the transfer, thus even in a scenario of a failed token transfer request, it'd be sent regradless. Meaning that the origin of the token transfer should check the condition before making the token transfer, by doing so, it'd also prevent relayer to waste resources conversely. Similar to what Ethereum is doing.
4. Relayer collected token `fee` yet assigned insufficient Ethereum gas fee, leaving the operations to be half done.
   > Add data of `gasAmount` to the signature, because the origin of the token transfer has defined this number, hence this is solvable, even if the number is wrong, we can still fix this issue similar to **scenario 1.**. In a worst case scenario, the transaction will fail due to insufficient gas fee, which itself is solvable.

We have considered all possible scenarios of attacks during our implementation and practices to ensure that we have a robust token relay model up and running.  
In other words, relayers will compete each other, thus providing a higher Ether gas price, making the token transfer request to be confirmed faster, that is also we have emphasises on the design and improvement of `fee` and `gasAmount` because only via a fair mechanism we can attract relayers to relay token transfers and allow them to earn tokens and enable token holders to enjoy a faster token verification, making it a win-win situation.

### Conclusion

This Token design is larger and complex compared to the others, yet it is still able to fit into a single block. To provide an extendable, scalable smart contract is our main goal, we also hope that this Token Standard would serve as the benchmark for **Service-Friendly Token Standard** that is yet to come, and to be the standard to allow **Utility Tokens** to be grounded. 

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This Token Standard is fully compatible with ERC-20.

## Test Cases

<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

We have tested the Token via the scripts off-chain, and several testing smart contracts on-chain.

Service-Friendly Token source code:  
https://github.com/funderstoken/Service-Friendly-Token-Standard/blob/develop/MinStandard/MinServiceFriendlyToken.sol

The test cases will be open-sourced soon.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

The Token that FundersToken (https://fstk.io) has issued, is one example of the Service-Friendly Token mentioned above，and it is integrated with the modules that FundersToken provide via our Decentralised Tokenisation Platform to form a robust service-based smart contracts.

Mainnet address:  
https://etherscan.io/address/0x51c028bc9503874d74965638a4632a266d31f61f#code

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
