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

This Token standard is designed to allow Tokens to interact with service-based smart contracts and off-chain services seamlessly and without friction, providing a friendly environment for the Tokens.

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

Originally designed to be a crowdfunding tool, Tokens now have a painful period of transition to being a Utility Token, to provide their service to the Token holders. Many projects and companies lack sufficient Smart Contract functionalities in their Tokens, which makes it difficult to support the fundamentals of their business venture and to apply it to real-world services and products.

The following interface designs are based on the fundamental features and aspects of a **Robust Tokenisation** that businesses need. This including removing the difficulties of secure bindings among smart contracts and on-chains and off-chains integration. We, FST Network based on our experience, understanding of the Token Standards that are available in the market, FST Network have made several improvements on the Token Standards aiming to build a Native Token Environment, a friendly environment for Tokens.

FST Network have also developed a **Token transfer relay**, which simulates blockchains in the form of smart contracts for the Tokens, and releases end-users from the need and limitation of only using Ether as transaction fee (gas fee) when making a Token transfer.

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

We categorise this interface standard to the following:

1.  [The improvements made on ERC-20](#the-improvements-to-erc-20)
2.  [The improvements made to make a Token service-friendly](#the-improvements-made-to-make-a-token-service-friendly)
3.  [The improvements for Robust Tokenisation](#the-improvements-for-robust-tokenisation)

As the most basic and most common way of controlling and storing Tokens, ERC-20 has proved to be a feasible and viable direction, however due to different implementations, such as gas consumption and mathematical safety of execution, many Tokens have suffered abuse and denial-of-service that led to financial loss.

We have made optimisation and made mathematical checks for the implementation of `transfer` and `approve`, and how to store `balance` and `allowance` efficiently.

---

To define a service-friendly environment, we must first identify the design goals of the payment flow and the smart contracts on the Ethereum Blockchain.

The payment flow of an Ethereum transaction:

```
(EA) --[tx]-> (CA 1) --[msg]-> (CA 2) --[msg]-> ... --[msg]-> (A)
```

The payment flow of a ERC20 Token transaction:

```
(EA) --[transfer]-> (A)
```

or

```
(EA) ---[approve]-> (CA)
(EA) ------[call]-> (CA) --[transferFrom EA]-> (A)
```

> EA represents External Account  
> CA represents Contract Account  
>  A represents EA and CA

Most of the current Token standards have difficulties to compose multiple continuous processes in one Ethereum transaction, in addition, the transaction must be triggered after the process of `approve` is done, this process is also in risk to be attacked by other smart contracts, by deliberately consuming the `allowance` more than the intended consumption.

From the statement above, we can see the Tokens are less direct and dynamic compared to Ether. Since Tokens are driven by smart contracts, Tokens must follow the execution process of the Ethereum transaction, which means the recipient address of a Token transfer transaction is the Token smart contract instead of `to` in `transfer`. The process and implementation of the Token `transfer` is not intuitive as Ether's transfer.

And because the Token ledger is within the smart contract of the Token, any mutation to the ledger (the balance and the allowance) or the logics must be designed and wrapped in the Token smart contract. Otherwise, the Token smart contract has to authorise or approve external smart contracts to extend the logic that is related to the ledger. But the former option slows down the development cycle, the latter option will increase the execution cost and security risks.

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

As for the improvement made for Tokenisation, it is based after our service-friendly Token, and we took a step further to conduct important features for businesses such as CRM functionality and Token relay to achieve a De-Ether environment.

The important feature for CRM is compacting multiple Token transfers and making the process as light and predictable as possible, which allows businesses to have more flexibility for CRM applications.

The Token relay is to remove the biggest technical barrier, which is the need for end-users to pay Ether in a Token transfer as the transaction fee.  
If the situation is in a context of  that Ethereum is a decentralised computing platform and cash platform, and to execute smart contracts, the end-users must pay Ether to stablise the Ethereum network and incentivise the miners to sustain the network, then it's very rational and acceptable to everyone.  
But if it is in the context of Tokens, it becomes illogical and cloggy.

The idea of "No Ether, No Token usages" obstructs the utility of Tokenisation.  
So we have implemented a feature that allows the origin of a Token transfer to sign a specific **Token transfer request**, and the **Relayers** check its transfer fee (in Token) and the signature, then the Relayers relay the request by sending the request to the Token smart contract, which also means the Relayers pay the Ethereum gas fee for the request.  
Then the Token smart contract checks the relayed transfers and avoid any attack among transfer origin, relayers and the receivers.

Further details are in the next section.

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### The improvements to ERC-20

Index:

1. [Extension to `address` and `uint256`](#extension-to-address-and-uint256)
2. [Immutable Token basic info](#immutable-token-basic-info)
3. [Optimised Token holders' storage](#optimised-token-holders-storage)
4. [Mutable Token basic info](#mutable-token-basic-info)
5. [Events of the Token](#events-of-the-token)
6. [Operation functions of the Token](#operation-functions-of-the-token)
7. [More secure Token](#more-secure-token)

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

_`DirectDebit` is a demonstration of data between Token holders that can be placed in `Instrument`. It is not included in this standard, but it is included in the full version of Service-Friendly Token Standard._

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

- `totalSupply()` is the total supply of Token issued.
- `balanceOf(address)` allows users to check their total balance.
- `allowance(address,address)` allows users to check the quota of Tokens that the Token holder that have authorised other holders to have access to.
- `address issuer` is the address of the Token issuer, although this is not a requirement for the ERC-20 standard, however this information is required for inspection in several actions.

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

- `Transfer(address,address,uint256)` is the event that triggers any change in the amount of Token.
- `Approval(address,address,uint256)` is the event where Token holder authorise access to different Token holders.

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

When `erc20ApproveChecking` is `true`, `approve(address,uint256)` will do extra checks to make sure the `allowance` of `spender` is 0 before this call, to prevent front-running attack by `spender`.

---

#### More secure Token:

- `bool erc20ApproveChecking` is a toggle to activate extra checks for `approve`, the value is `false` by default, and only `issuer` can change this value.
- `SetERC20ApproveChecking(bool)` is an event emitted when `erc20ApproveChecking` is changed via `setERC20ApproveChecking(bool)`.
- `approve(address,uint256,uint256)` is an `approve` that requires `expectedValue` before assigning new `allowance`
- `increaseAllowance(address,uint256)` can directly increment `allowance`
- `decreaseAllowance(address,uint256,bool)` can directly decrease `allowance`, and when `strict` is `true`, it will do the substraction via `Math` library.
- `spendableAllowance(address,address)` provides the `Math.Min` of **the balance of the Token holder** and **the allowance for the `spender`**.

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

Index:

1. [Transfer and call](#transfer-and-call)

---

#### Transfer and call:

To make Token transfer and calling `receivcerContract` in one go, and to make these calls chainable and trustworthy, which guarantees that `value` and `msg.sender` are secured by the Token smart contract, the arguments checks and the replacement in Token smart contract will be very strict.

In `transferAndCall(address,uint256,bytes)`,

- `address to` is the address of the `receiverContract`.
- `uint256 value` is the Token value to be transferred.
- `bytes data` is the calldata for all post continuous processes, which is used with `to.call(data)`, including `fuction signature`, `value` and `msg.sender`

Because `data` must contain the calldata to the `receiverContract`, the length would be at least **4 bytes signature + 32 bytes value + 32 bytes sender** = **68 bytes**.

And this function does the checks below:

- `to` must not be Token smart contract itself.
- The length of `data` must be equal or longer than 68 bytes
- The Token transfer needs to be successful before calling `receiverContract`

Then replaces first two arguments in `data` by force, the original `data` must be in the form:

```
[4 bytes signature][32 bytes value][32 bytes msg.sender][other calldata.....]
```

We put `uint256 value` before `address sender` (`address from`) to not be confused by `address to` + `uint256 value` pair in normal `transfer(address,uint256)`.

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

Also, the method in the receiverContract must have first two arguments that match `uint256 value` and `address from` order.  
For instance:

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

And for another instance, to allow end-users purchase some TokenB in 100 TokenA, just encode the tx input below, and let it be signed and broadcast.

Assume that `msg.sender` (`from`) is `0x83b21dbd0e60b9709d647de183f5ae0c31b54c2a`, and the `receiverContract` (VendorMachine) is `0x1234567890123456789012345678901234567890`:

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000083b21dbd0e60b9709d647de183f5ae0c31b54c2a0000000000000000000000000000000000000000000000056bc75e2d63100000");
```

Or place arbitrary bytes after the function signature ( `"0x" + keccak256("purchase(uint256,address)")[0~7]` = `0xae77c237` )

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001");
```

### The improvements for Robust Tokenisation

Index:

1. [Multiple Token transfer](#multiple-token-transfer)
2. [Delegated Token transfer and call](#delegated-token-transfer-and-call)

---

#### Multiple Token transfer

- `transfer(uint256[])` allows Token holders to transfer Token to multiple recipients in one single transaction.
- `uint256[] data` in `transfer(uint256[])` consists of the elements (`uint256`) that each forms in **20 bytes recipientAddress + 12 bytes value**

We merge the recipient and value into one `uint256` in order to compact the multiple transfers.

Although now each recipient can only receive 12 bytes `uint`, but for the Token that has `18 decimals` is very ample, because it can represent 79,228,162,514.264337593543950335 Token in maximum, which is far more than the sane `totalSupply` of a Robust Tokenisation. `0xffffffffffffffffffffffff / (10 ** 18)`

And to be correctly displayed on Blockchain explorers, every `Transfer(address,address,uint256)` must be emitted, and to optimise the I/O of the balance storage, the final result of the Token sender is set in the end of the multiple Token transfer.

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

#### Delegated Token transfer and call:

This is the key component of a Robust Tokenisation, allowing the Token transfer to pay the fee in Token but not to pay Ethereum Gas Fee anymore.

This delegation is also toggleable:

- `bool isDelegateEnable` is a toggle to activate delegation, the value is `false` by default, and only `issuer` can change this value.
- `SetDelegate(bool)` 為 `isDelegateEnable` is an event emitted when `isDelegateEnable` is changed via `SetDelegate(bool)`.

```
bool public isDelegateEnable;

event SetDelegate(bool isDelegateEnable);

function setDelegate(bool delegate) public {
  require(msg.sender == issuer);
  emit setDelegate(isDelegateEnable = delegate);
}
```

---

In `delegateTransferAndCall(uint256,uint256,uint256,address,uint256,bytes,uint8,uint8,bytes32,bytes32)`,

- `uint256 nonce` is the count of the transfer of the Token holder (Token transfer origin).
- `uint256 fee` is the Token transfer fee for the relayer set by the Token transfer origin.
- `uint256 gasAmount` is the gas amount set by the Token transfer origin to avoid wasting attack.
- `address to` is the address of the recipient.
- `uint256 value` is same as the `value` in `transfer(address,uint256)`.
- `bytes data` is same as the `data` in `transferAndCall(address,uint256,bytes)`.
- `DelegateMode mode` sets the delegation mode to define who can receive the Token transfer fee.
- `uint8 v` is the `v` in the ECDSA signature signed by the origin of the Token transfer.
- `bytes32 r` is the `r` in the ECDSA signature signed by the origin of the Token transfer.
- `bytes32 s` is the `s` in the ECDSA signature signed by the origin of the Token transfer.

The origin of the Token transfer needs to compose the informations described above and sign them, then send the informations to the relayers.  
The implementation of off-chain part is not restricted, but the generation of the ECDSA signature must follow:

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

There are 4 options for `DelegateMode`:

- 0: `PublicMsgSender` represents any relayer can relay the transfer request, and the `fee` will be transferred to `msg.sender` in the delegation.
- 1: `PublicTxOrigin` represents any relayer can relay the transfer request, and the `fee` will be transferred to `tx.origin` in the delegation.
- 2: `PrivateMsgSender` represents the origin of the Token transfer has assigned a specific relayer, and the `fee` will be transferred to `msg.sender` in the delegation.
- 3: `PrivateTxOrigin` represents the origin of the Token transfer has assigned a specific relayer, and the `fee` will be transferred to `tx.origin` in the delegation.

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

Check the `nonce`:

- `nonceOf(address)` allows users to check the nonce of their accounts.

```
function nonceOf(address owner) public view returns (uint256) {
  return accounts[owner].nonce;
}
```

If the origin of the Token transfer needs to cancel the mis-sent Token transfer or prevent any kinds of front-running attack, it can be cancelled by manually increasing the nonce of the origin.

- `IncreaseNonce(address,uint256)` is an event emitted when the `nonce` increases via `delegateTransferAndCall()` and `increaseNonce()`.
- `increaseNonce()` allows the Token holder (the origin of the Token transfer) to manually increase the nonce to let mis-sent Token transfer failed.

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

As ERC-20 only specifies the interface and has lacked the requirements and suggestion for real-world practices. While building on top of the ERC-20 standard, we have encountered a few challenges and after finding the solution, we decided to infuse our solution into this standard. For enterprise users and Token end-users, a vivid operating logic and low-cost is a must, with that in mind, we have implemented several optimisation for storage. Taking `accounts` as an example, we have changed the mapping of accounts to only read the struct of the reference and only to take the necessary data, by doing this, the process would be lighter compared to read the all accounts-related mappings.

We have also made improvements on security, specifically on preventing front-running attack by the `spender`, when the Token holder is assigning `allowance`. If this process is not checked, the process of assigning `allowance` from 1,000 to 500, in a front-running attack scenario, the attacker would first spend 1,000 that was previously assigned, and as the Token holder  approve another 500, making the `allowance`: **1,000 -> 0 -> 500**.

In a safer environment, the requirement before approving a new value, at least under the ERC-20 interface (`approve(address,uint256)`), the `allowance` must return to 0. While this mechanism makes it safer, but this would mean that 2 Ethereum transactions would be needed. However we would suggest to utilise the interface of `expectedValue`, as both `increaseAllowance` and `decreaseAllowance` are in risk of front-running attack.

### The improvements made to make a Token service-friendly

First of all, we intentionally named the function as `transferAndCall`, instead of function overload, that is because the definition of `transfer` is from one account to another account, sometimes even a smart contract, which is logical, as some kinds of the procedures must first sent to the smart contract, awaiting for the next step (the next transaction).

However if we prohibit ERC-20 `transfer` just because of too many Tokens are sent to a smart contract that are unable to transfer the Token, it would be putting the cart before the horse. As this is not the fault of ERC-20 but the fault of the one who transfers the Tokens and the service provider.

We intentionally used a different terms to express different functionality, most importantly different aims and different motives. Because we wanted to make Token-related transaction and transfer process to be intuitive and easy to plan, we have allowed the mechanism for receiving smart contract to perform continuous actions and to replace the original calldata with the correct calldata by force, thus no one would be able to abuse any resources.

Imagine a single Ethereum transaction to perform a continuous `transferAndCall`, it is similar to putting a complex payment flow on chain, complex yet stream-lined procedure, and can be extensive, scalable by will, to achieve a service-friendly modularisation for Tokens.

This is why this section is called **Service-Friendly**, because a Service-Friendly smart contract is easy to use, trustworthy, also to be able to integrate the business logics off-chain, to be able to do all of these on Ethereum, that'd be a blessing.

### The improvements for Robust Tokenisation

The last modification that we did to improve the flow of payment is to removing the need of paying Ether to make a Token transfer. Although the concept of **Account Abstraction** is set to release in the near future, however in the concept of Account Abstraction, the service provider is the one who carries the burden of paying Ether, and this action will bring hard-cost to the business, making it hard to scale.

A **volunteer**, also known as a **Relayer** must be able to voluntarily pick a normal, non-error Token transfer request and relay it to the Ethereum blockchain, then via smart contract to confirm and check what object was specified and what modification was made by the relayer, just to make sure the origin of the Token transfer won't be able to attack the relayer and the relayer won't be able to attack the origin of the Token transfer.    

The list below shows the potential attack and the solution for the attacks:

1. Relayer repeatedly sends Token transfer request or broadcast a wrong Token transfer to the Token smart contract.
   > This can be solved by adding in `nonce` or manually increase the `nonce` (`increaseNonce()`). Even when the Token transfer execution has failed, `nonce` will increase regardless.
2. Relayer took the Token transfer request and send it to another Token smart contract with the same `nonce`
   > Can be solved by adding data of `tokenAddress` into signature
3. The origin of the Token transfer wasting Relayer's Ethereum Gas fee.
   > This can be solved by adding the data for `gasAmount` into the signature, in a scenario where the origin of the Token transfer provides a very small incentive to the relayer, the relayer will be able to see it from the start, hence will be able to drop the request.
   > And also, to make sure this type of attack will inevitably fail, and to ensure relayer to get the `fee` for the transfer, thus even in a scenario of a failed Token transfer request, it'd be sent regradless. Meaning that the origin of the Token transfer should check the condition before making the Token transfer, by doing so, it'd also prevent relayer to waste resources conversely. Similar to what Ethereum is doing.
4. Relayer collected Token `fee` yet assigned insufficient Ethereum gas fee, leaving the operations to be half done.
   > Add data of `gasAmount` to the signature, because the origin of the Token transfer has defined this number, hence this is solvable, even if the number is wrong, we can still fix this issue similar to **scenario 1.**. In a worst case scenario, the transaction will fail due to insufficient gas fee, which itself is solvable.

We have considered all possible scenarios of attacks during our implementation and practices to ensure that we have a robust Token relay model up and running.  
In other words, relayers will compete each other, thus providing a higher Ethereum gas price, making the Token transfer request to be confirmed faster, that is also we have emphasised on the design and improvement of `fee` and `gasAmount`, because only via a fair mechanism we can attract relayers to relay Token transfers and allow them to earn Tokens, and enable Token holders to enjoy a faster Token verification, making it a win-win situation.

### Conclusion

This Token is larger and complex compared to the others, yet it is still able to fit into a single block during deployment. To provide an extendable, scalable smart contract is our main goal, we also hope that this Token Standard would serve as the benchmark for **Service-Friendly Token Standard** that is yet to come, and to be the standard to allow **Utility Tokens** to be grounded. 

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This Token Standard is fully compatible with ERC-20.

## Test Cases

<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

We have tested the Token via the scripts off-chain, and several testing smart contracts on-chain.

Service-Friendly Token source code:  
https://github.com/fstnetwork/Service-Friendly-Token-Standard/blob/develop/MinServiceFriendlyToken.sol

The test cases will be open-sourced soon.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

The Token that FST Network (https://fst.network) has issued, is one example of the Service-Friendly Token mentioned above, and it is integrated with the modules that FST Network provide via our Decentralised Tokenisation Platform to form a robust service-based smart contracts.

Mainnet address (Funder Smart Token, FST):  
https://etherscan.io/address/0x51c028bc9503874d74965638a4632a266d31f61f#code

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
