---
eip: <to be assigned>
title: Service-Friendly Token Standard
author: Atkins Chang <atkins@fstk.io> (@AtkinsChang), Noel Bao <noel@fstk.io> (@noeleon930)
discussions-to: <URL>
status: Draft
type: Standard
category: ERC
created: 2018-08-08
requires: ERC-20
---

# Service-Friendly Token Standard

## Simple Summary

<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->
<!-- A standard interface for service-friendly tokens, which aims for a grounded代幣化environment for business. -->

這個代幣介面標準是為了讓代幣 (Token) 能更方便地與服務型智能合約或鏈下服務對接，並提供對於代幣是友善的開發環境、使用環境。

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

原本專注於群眾募資的代幣技術與市場，遭遇到了即將要轉型成實用型 (Utility Token) 的陣痛期，非常多的專案或企業遇到了代幣的智能合約功能不足的問題，難以支撐基本的商業模式並應用於更多現實世界的服務或產品。

以下的諸多介面設計中，都是基於商業在經歷健康的代幣化 (Tokenisation) 上常會需要的基本功能，主要是面向移除智能合約間的安全連接困難、移除鏈上下的整合困難，以及我們 FundersToken 對於諸多代幣介面標準 (Token standard) 的理解跟改善，試圖建立原生代幣自主環境 (Native Token environment)，即對於代幣運作是友善的環境。

以及 FundersToken 原創的代幣傳輸轉發 (Token transfer relay)，為代幣以智能合約的方式模擬區塊鏈，可以讓終端使用者免於需要支付以太幣作為燃料費的限制。

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

我們將此介面標準中的功能們分成以下幾類:

1.  針對 ERC-20 做的補強
2.  針對服務友善的環境 (Service-Friendly) 所做出的補強
3.  針對健全的代幣化所做出的補強

ERC-20 作為最基本最普遍的代幣使用方式及儲存方式，著實被證明是一個可行的方向，但其中因著不同的實作方式，執行時所耗的燃料成本與數學上的安全性，就造成不少代幣遭遇到了濫用或服務停擺。

我們針對 `transfer` 與 `approve` 的實作方式進行了執行時間的優化與嚴格的數學檢查，以及如何儲存 `balance` 與 `allowance` 進行了小量規範。

關於何謂服務友善的環境，我們可以簡單地從金流與智能合約一開始的設計目的出發。

以太坊交易 (Transaction) 與金流的旅程:

    (EA) --[tx]-> (CA 1) --[msg]-> (CA 2) --[msg]-> ... --[msg]-> (A)

而 ERC-20 交易與金流的旅程:

    (EA) --[transfer]-> (A)
    或
    (EA) ---[approve]-> (CA)
    (EA) ------[call]-> (CA) --[transferFrom EA]-> (A)

> EA 是 External Account
>
> CA 是 Contract Account
>
> A 是 EA and CA

絕大部分現行的代幣標準難以在一次的交易中完成自動步驟，還要得 `approve` 之後觸發交易才行。

從上述即可看得出，代幣一開始就比以太幣 (Ether) 還要不方便使用，代幣是靠智能合約驅動出來的，智能合約的執行本身必須依循以太坊的 transaction 執行流程，導致代幣數字 `transfer` 流程的直覺理解與實際技術上的實作方式是不同的。

也因為代幣的帳本 (Ledger) 就在代幣的智能合約中，帳本裡數字的變化操作就也要被包在代幣智能合約裡，或者是驗證外部智能合約的邏輯使帳本的數字能被調動，但前者會讓代幣開發緩慢，而後者會讓執行成本升高與安全性風險增高。

FundersToken 在提供模組化智能合約與代幣化服務時，在開發過程中體驗到相當多的代幣不方便的環境。我們的流程目標為:

    (EA) --[transfer or call]-> (CA 1) --[transfer or call]-> (CA 2) --[transfer or call]-> ... --[transfer or call]-> (A)

簡單來說，我們希望代幣的金流或業務流程可以像是以太坊原本的方式一樣自然，並且讓代幣相關的服務開發起來是簡單直覺的，而非受了太多 ERC-20 沒有解決到的阻礙，造成業務擴展受到影響。

為了達成這些目的，我們針對 ERC-223 或 ERC-827 的 `transferAndCall` 的實作方法與潛在威脅進行了優化與增強，其中，讓 `receiverContract` 也就是服務型智能合約 (Service contract) 收到正確的代幣傳輸數字 (Value) 與正確的代幣傳送者 (Transfer sender)，讓 `receiverContract` 不會攻擊代幣傳送者，詳細會在下面補充。

更進一步地說，以上不只是讓多個服務型智能合約的連接彈性與一致性獲得相當良好的提昇，讓智能合約間的業務流程模組化，並可以自由、信任地連接。這也使與鏈下接合 API 時，讓業務流程得到一次完整的一致性操作，大幅降低鏈下的狀態檢查或業務流程的影響，提高了更多鏈外開發者的導入意願度。

對於代幣化所做的再補強，是基於了服務友善化之後，進一步移除以太坊手續費、以太坊主動操作等這類的代幣化阻礙，有以下兩種：

1.  使代幣支援週期性的被動操作，例如定期直接扣款，就像是每月自動繳付信用卡費用一般
2.  使代幣支援一次性大量操作
3.  使代幣在被操作時，終端使用者不用負擔以太坊手續費

目前，被動操作在代幣上的實現方式為 `approve` 一個對象，使這個對象可以自行依照 `allowance` 的量進行代幣操作，而當業務流程上有個定期扣款的需求，終端使用者會變得要手動定期進行 `approve` 一個對象，對此我們實作了定期的直接扣款機制，讓被扣款方可以一次設定週期性設定，讓扣款方可以定期操作，也支援一次對多方進行直接扣款。

最後我們談到，目前以太坊上的代幣環境受到最大阻力的一個技術性原因，就是當終端使用者在傳輸代幣的時候，要支付以太幣當作手續費。這件事情的脈落若是以太坊身為一個去中心計算平台、金流平台，執行智能合約時支付燃料來穩定網路、回饋挖礦者或驗證者的話，無不合理而且大家都贊同。但以代幣終端使用者角度而言，這件事情就變得非常不正確，「沒有以太幣則無法使用代幣服務」的限制，讓代幣環境遭受到最大的「代幣化」阻礙。

所以我們規範並實作了讓「代幣傳送者」簽署出特別的代幣傳送請求，讓「轉發者」可以檢查其中的代幣傳輸費、簽章，然後幫忙轉發此請求至代幣合約，也就是轉發者幫忙支付了以太坊燃料費，代幣合約將檢查並履行其中的代幣傳送。此外，我們也避免代幣傳送者攻擊轉發者，或者是轉發者攻擊代幣傳送者。

因著以上動機所做出的介面標準或實作，請參考下一個部份。

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### ERC-20 補強

#### 對於 `address` 與 `uint256` 的延伸

<details><summary>AddressExtension</summary>

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

<details><summary>Math</summary>

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

#### 基本的代幣資訊，一開始就指定好並且是常數性的。

- `name` 為代幣名稱
- `symbol` 為代幣代號
- `decimals` 為儲存代幣擁有者的數字時，儲存的位數精度

```
string public constant name;
string public constant symbol;
uint8 public constant decimals;
```

---

#### 優化過的儲存代幣擁有者的資訊，實作部份

在 `Account` 中

- `uint256 balance` 為擁有代幣數、餘額，與 `decimals` 有關
- `uint256 nonce` 為擁有者所操作過的 transfer (代幣傳輸) 個數，避免傳送，但只用於轉發模式，在後面將會說明
- `mapping (address => Instrument) instruments` 為儲存代幣擁有者與其他代幣擁有者之間的資料，包含

在 `Instrument` 中

- `uint256 allowance` 為代幣擁有者允許其他帳戶可以利用自己的多少額度
- `DirectDebit directDebit` 為代幣擁有者允許其他帳戶可以定期直接扣款的額度設定，`DirectDebit` 的部份在後面將會說明

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

#### 會變動的代幣資訊

- `totalSupply()` 為代幣總發行量
- `balanceOf(address)` 為查詢代幣擁有者的代幣餘額
- `allowance(address)` 為查詢代幣擁有者允許其他帳戶可以利用自己的多少額度
- `address issuer` 為代幣發行者位址，這雖然非 ERC20 標準，而於諸多操作中需要此資訊之檢查

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

#### 代幣事件

- `Transfer(address,address,uint256)` 為任何一個代幣數字變動時應發射的事件
- `Approval(address,address,uint256)` 為任何一次的代幣擁有者允許其他帳戶使用時發射的事件

```
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);
```

其中，因為支持絕大部分的區塊鏈瀏覽器服務，如 Etherscan ，在代幣一開始被建構時，發射事件，表示初始代幣發行。

```
emit Transfer(address(0), <tokenIssuer>, <totalSupply>);
```

---

#### 代幣的操作相關函數

以下與 ERC20 的介面標準是一樣的，但與數學相關的操作，特別是減法的部份就會以 `Math` 的延伸方法進行操作，並搭配讀取 `accounts` 映射表來降低映射表操作次數

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

`approve(address,uint256)` 中的 `erc20ApproveChecking` 請見下一個部份。

而當 `erc20ApproveChecking` 開啟時，此 `approve` 中會額外做的檢查為，檢查 `spender` 目前的 `allowance` 是否為 0，以防 spender 插隊攻擊代幣擁有者。

---

#### 增強安全用代幣資訊、操作

- `bool erc20ApproveChecking` 為一個狀態值紀錄是否要開啟更安全的 `approve` 相關執行檢查，預設為 `false`，只有 `issuer` 才能更動
- `SetERC20ApproveChecking(bool)` 為 `erc20ApproveChecking` 改變時會發射的事件，需要透過 `setERC20ApproveChecking(bool)` 引發
- `approve(address,uint256,uint256)` 會要求代幣擁有者輸入預期的 `allowance`，通過驗證才能繼續改變 `allowance`
- `increaseAllowance(address,uint256)` 可直接增加 `allowance`
- `decreaseAllowance(address,uint256)` 可直接減少 `allowance`，而當 `strict` 為 `true` 時，會用 `Math` 進行減法檢查
- `spendableAllowance(address)` 可直接得知被允許之帳戶可以實際上消耗多少額度

<details><summary>Secure ERC20 (安全版 ERC20)</summary>

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

### Service-Friendly (服務友善化) 補強

#### Transfer and call (傳輸呼叫)

為了讓傳輸代幣與呼叫接收者智能合約 (receiverContract) 是一氣呵成，能讓這些呼叫可以連續地一個串一個串下去，並且同時也讓接收者智能合約可以得到真正的 `value` 與 `msg.sender`，對於參數的檢查與覆蓋就會變得非常嚴格

在 `transferAndCall(address,uint256,bytes)` 的參數中

- `address to` 為接收者智能合約的位址
- `uint256 value` 為代幣傳輸量，與 `transfer` 的一樣意義
- `bytes data` 為後續所有連續動作都需要的參數資料，與 `receiverContractAddress.call(data)` 搭配使用，`data` 其中應內含 `signature`、 `value` 與 `msg.sender`

並且因為 `data` 最少要包含要傳遞給接收者智能合約的資料，故長度至少為 **4 bytes signature + 32 bytes value + 32 bytes sender** = **68 bytes**

也會進行下列檢查

- 禁止 `to` 為合約本身
- 檢查 `data` 的長度需大於等於 68 bytes
- 檢查確定代幣傳輸已經完成

以及對於 data 的前兩個參數進行強制覆蓋，讓 `data` 中必定是

```
[4 bytes signature][32 bytes value][32 bytes msg.sender][其他原先的資料們]
```

故意讓 `value` 先而 `sender` 後的原因為，不與 `to` `value` 的組合順序搞混

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

以及，接收者智能合約的範例:

```
// Receiver Contract (Vendor machine, sells TokenB)

function purchase(
  uint256 value,
  address buyer
)
  public
  payable
  returns (bool)
{
  require(msg.sender == address(TokenA));

  return TokenB.transfer(buyer, calculateAmount(value));
}
```

所以要使終端使用者可以用 100 TokenA 購買 TokenB 時，只要能編碼下列 tx input，簽署並送出即可

假設 `msg.sender` (buyer) 為 `0x83b21dbd0e60b9709d647de183f5ae0c31b54c2a`，也假設接收者智能合約 (VendorMachine) 為 `0x1234567890123456789012345678901234567890`

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000083b21dbd0e60b9709d647de183f5ae0c31b54c2a0000000000000000000000000000000000000000000000056bc75e2d63100000");
```

或者擺隨意的 bytes 在後面，但 signature 不能影響到 ( ```"0x" + keccak256("purchase(uint256,address)")[0~7]``` = `0xae77c237` )

```
transferAndCall(
  "0x1234567890123456789012345678901234567890",
  "100000000000000000000",
  "0xae77c23700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001");
```

### Tokenisation (代幣化) 補強

#### 週期性的直接扣款

直接扣款系列的實作是使代幣擁有者可以週期性地允許外部服務週期性地扣款

在上述的 `Instrument` 結構中的 `DirectDebit` 中:

 - `DirectDebitInfo info` 為直接扣款資訊
 - `uint256 epoch` 為紀錄已經被扣款過的期數

在 `DirectDebitInfo` 中:

 - `uint256 amount` 為每期的允許扣款額度
 - `uint256 startTime` 為允許的開始扣款時間
 - `uint256 interval` 為每期的週期間隔時間

```
struct DirectDebit {
  DirectDebitInfo info;
  uint256 epoch;
}

struct DirectDebitInfo {
  uint256 amount;
  uint256 startTime;
  uint256 interval;
}
```

直接扣款也是一個可以開啟或關閉的功能:

- `bool isDirectDebitEnable` 為一個狀態值紀錄是否要開啟更安全的 `approve` 相關執行檢查，預設為 `false`，只有 `issuer` 才能更動
- `SetDirectDebit(bool)` 為 `isDirectDebitEnable` 改變時會發射的事件，需要透過 `setDirectDebit(bool)` 引發

```
bool public isDirectDebitEnable;

event SetDirectDebit(bool isDirectDebitEnable);

function setDirectDebit(bool directDebit) public {
  require(msg.sender == issuer);
  emit SetDirectDebit(isDirectDebitEnable = directDebit);
}
```

設定直接扣款的操作中:

 - `SetupDirectDebit(address,address,(uint256,uint256,uint256))` 為當一個代幣擁有者對某個位址設定了允許直接扣款時，所發射的事件
 - `setupDirectDebit(address,(uint256,uint256,uint256))` 為代幣擁有者允許某個位址定期直接扣款的操作

```
function setupDirectDebit(
  address receiver,
  DirectDebitInfo info
)
  public
  returns (bool)
{
  accounts[msg.sender].instruments[receiver].directDebit = DirectDebit({
    info: info,
    epoch: 0
  });

  emit SetupDirectDebit(msg.sender, receiver, info);
  return true;
}
```

例如設定了每期 10 代幣，開始時間為 2019-01-01T08:08:08.000Z，每期間隔為 2 天

```
       epoch 1           epoch 2           epoch 3
 |-----10token-----|-----10token-----|-----10token-----|
 S              S+2days           S+4days           S+6days
```

假如現在在 epoch N 的時間區段中，則直接扣款方就可以收取 epoch 1 - N 該扣到的款項，也就是可以累積，但不應會超過收取或重複

扣款方在直接扣款的操作中:

 - `withdrawDirectDebit(address)` 為扣款方指定被扣款方並進行扣款的操作，並會觸發 `Transfer(address,address,uint256)`  事件

```
function withdrawDirectDebit(address debtor) public returns (bool) {
  require(isDirectDebitEnable);

  Account storage debtorAccount = accounts[debtor];
  DirectDebit storage debit = debtorAccount.instruments[msg.sender].directDebit;

  uint256 epoch = (block.timestamp.sub(debit.info.startTime) / debit.info.interval).add(1);
  uint256 amount = epoch.sub(debit.epoch).mul(debit.info.amount);
  
  require(amount > 0);
  
  debtorAccount.balance = debtorAccount.balance.sub(amount);
  accounts[msg.sender].balance += amount;
  debit.epoch = epoch;

  emit Transfer(debtor, msg.sender, amount);
  
  return true;
}
```

#### 一次性大量操作

#### 代幣傳送委派、代幣轉發

---

## Rationale

<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Test Cases

<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
