---
eip: <to be assigned>
title: Service-Friendly Token Standard
author:
  Atkins Chang <atkins@fstk.io> (@AtkinsChang), Noel Bao <noel@fstk.io> (@noeleon930)
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

這個代幣介面標準是為了讓代幣 (Token) 能更方便地與服務型智能合約或鏈下服務對接，並提供對於代幣是友善的開發環境、使用環境。

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

原本專注於群眾募資的代幣技術與市場，遭遇到了即將要轉型成實用型 (Utility Token) 的陣痛期，非常多的專案或企業遇到了代幣的智能合約功能不足的問題，難以支撐基本的商業模式並應用於更多現實世界的服務或產品。

以下的諸多介面設計中，都是基於商業在經歷健康的代幣化 (Tokenisation) 上常會需要的基本功能，主要是面向移除智能合約間的安全連接困難、移除鏈上下的整合困難，以及我們 FundersToken 對於諸多代幣介面標準 (Token standard) 的理解跟改善，試圖建立原生代幣自主環境 (Native Token environment)，即對於代幣運作是友善的環境。

以及 FundersToken 原創的代幣傳送轉發 (Token transfer relay)，為代幣以智能合約的方式模擬區塊鏈，可以讓終端使用者免於需要支付以太幣作為燃料費的限制。

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
> CA 是 Contract Account  
> A 是 EA and CA

絕大部分現行的代幣標準難以在一次的交易中完成自動步驟，還要得 `approve` 之後觸發交易才行，甚至可能被其他智能合約攻擊，藉由故意消耗原意以外的 `allowance` 的方式。

從上述即可看得出，代幣一開始就比以太幣 (Ether) 還要不方便使用，代幣是靠智能合約驅動出來的，智能合約的執行本身必須依循以太坊的 transaction 執行流程，導致代幣數字 `transfer` 流程的直覺理解與實際技術上的實作方式是不同的。

也因為代幣的帳本 (Ledger) 就在代幣的智能合約中，帳本裡數字的變化操作就也要被包在代幣智能合約裡，或者是驗證外部智能合約的邏輯使帳本的數字能被調動，但前者會讓代幣開發緩慢，而後者會讓執行成本升高與安全性風險增高。

FundersToken 在提供模組化智能合約與代幣化服務時，在開發過程中體驗到相當多的代幣不方便的環境。我們的流程目標為:

```
(EA) --[transfer and call]-> (CA 1)
     --[transfer and call]-> (CA 2)
     --[transfer and call]-> ...
     --[transfer and call]-> (A)
```

簡單來說，我們希望代幣的金流或業務流程可以像是以太坊原本的方式一樣自然，並且讓代幣相關的服務開發起來是簡單直覺的，而非受了太多 ERC-20 沒有解決到的阻礙，造成業務擴展受到影響。

為了達成這些目的，我們針對 ERC-223 或 ERC-827 的 `transferAndCall` 的實作方法與潛在威脅進行了優化與增強，其中，讓 `receiverContract` 也就是服務型智能合約 (Service contract) 收到正確的代幣傳送數字 (Value) 與正確的代幣傳送者 (Transfer sender)，讓 `receiverContract` 不會攻擊代幣傳送者，詳細會在下面補充。

更進一步地說，以上不只是讓多個服務型智能合約的連接彈性與一致性獲得相當良好的提昇，讓智能合約間的業務流程模組化，並可以自由、信任地連接。這也使與鏈下接合 API 時，讓業務流程得到一次完整的一致性操作，大幅降低鏈下的狀態檢查或業務流程的影響，提高了更多鏈外開發者的導入意願度。

對於代幣化所做的再補強，是基於了服務友善化之後，進一步移除以太坊手續費、以太坊主動操作等這類的代幣化阻礙，有以下兩種：

1.  使代幣支援週期性的被動操作，例如定期直接扣款，就像是每月自動繳付信用卡費用一般
2.  使代幣支援一次性大量操作
3.  使代幣在被操作時，終端使用者不用負擔以太坊手續費

目前，被動操作在代幣上的實現方式為 `approve` 一個對象，使這個對象可以自行依照 `allowance` 的量進行代幣操作，而當業務流程上有個定期扣款的需求，終端使用者會變得要手動定期進行 `approve` 一個對象，對此我們實作了定期的直接扣款機制，讓被扣款方可以一次設定週期性設定，讓扣款方可以定期操作，也支援一次對多方進行直接扣款。

最後我們談到，目前以太坊上的代幣環境受到最大阻力的一個技術性原因，就是當終端使用者在傳送代幣的時候，要支付以太幣當作手續費。這件事情的脈落若是以太坊身為一個去中心計算平台、金流平台，執行智能合約時支付燃料來穩定網路、回饋挖礦者或驗證者的話，無不合理而且大家都贊同。但以代幣終端使用者角度而言，這件事情就變得非常不正確，「沒有以太幣則無法使用代幣服務」的限制，讓代幣環境遭受到最大的「代幣化」阻礙。

所以我們規範並實作了讓「代幣傳送者」簽署出特別的代幣傳送請求，讓「轉發者」可以檢查其中的代幣傳送費、簽章，然後幫忙轉發此請求至代幣合約，也就是轉發者幫忙支付了以太坊燃料費，代幣合約將檢查並履行其中的代幣傳送。此外，我們也避免代幣傳送者攻擊轉發者，或者是轉發者攻擊代幣傳送者。

因著以上動機所做出的介面標準或實作，請參考下一個部份。

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### ERC-20 補強

#### 對於 `address` 與 `uint256` 的延伸:

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

#### 基本的代幣資訊，一開始就指定好並且是常數性的:

- `string name` 為代幣名稱
- `string symbol` 為代幣代號
- `uint8 decimals` 為儲存代幣擁有者的數字時，儲存的位數精度

```
string public constant name;
string public constant symbol;
uint8 public constant decimals;
```

---

#### 優化過的儲存代幣擁有者的資訊，實作部份:

在 `Account` 中

- `uint256 balance` 為擁有代幣數、餘額
- `uint256 nonce` 為擁有者所操作過的 transfer (代幣傳送) 個數，避免傳送，但只用於轉發模式，在後面將會說明
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

#### 會變動的代幣資訊:

- `totalSupply()` 為代幣總發行量
- `balanceOf(address)` 為查詢代幣擁有者的代幣餘額
- `allowance(address)` 為查詢代幣擁有者允許其他帳戶可以利用自己的多少額度
- `address issuer` 為代幣發行者位址，這雖然非 ERC-20 標準之要求，而於諸多操作中需要此資訊之檢查

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

#### 代幣事件:

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

#### 代幣的操作相關函數:

以下與 ERC-20 的介面標準是一樣的，但與數學相關的操作，特別是減法的部份就會以 `Math` 的延伸方法進行操作，並搭配讀取 `accounts` 映射表來降低映射表操作次數

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

#### 增強安全用代幣資訊、操作:

- `bool erc20ApproveChecking` 為一個狀態值紀錄是否要開啟更安全的 `approve` 相關執行檢查，預設為 `false`，只有 `issuer` 才能更動
- `SetERC20ApproveChecking(bool)` 為 `erc20ApproveChecking` 改變時會發射的事件，需要透過 `setERC20ApproveChecking(bool)` 引發
- `approve(address,uint256,uint256)` 會要求代幣擁有者輸入預期的 `allowance`，通過驗證才能繼續改變 `allowance`
- `increaseAllowance(address,uint256)` 可直接增加 `allowance`
- `decreaseAllowance(address,uint256)` 可直接減少 `allowance`，而當 `strict` 為 `true` 時，會用 `Math` 進行減法檢查
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

### Service-Friendly (服務友善化) 補強

#### Transfer and call (傳送呼叫):

為了讓傳送代幣與呼叫接收者智能合約 (receiverContract) 是一氣呵成，能讓這些呼叫可以連續地一個串一個串下去，並且同時也讓接收者智能合約可以得到真正的 `value` 與 `msg.sender`，對於參數的檢查與覆蓋就會變得非常嚴格

在 `transferAndCall(address,uint256,bytes)` 的參數中

- `address to` 為接收者智能合約的位址
- `uint256 value` 為代幣傳送量，與 `transfer` 的一樣意義
- `bytes data` 為後續所有連續動作都需要的參數資料，與 `receiverContractAddress.call(data)` 搭配使用，`data` 其中應內含 `signature`、 `value` 與 `msg.sender`

並且因為 `data` 最少要包含要傳遞給接收者智能合約的資料，故長度至少為 **4 bytes signature + 32 bytes value + 32 bytes sender** = **68 bytes**

也會進行下列檢查

- 禁止 `to` 為合約本身
- 檢查 `data` 的長度需大於等於 68 bytes
- 檢查確定代幣傳送已經完成

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

假設 `msg.sender` (from) 為 `0x83b21dbd0e60b9709d647de183f5ae0c31b54c2a`，也假設接收者智能合約 (VendorMachine) 為 `0x1234567890123456789012345678901234567890`

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

### Tokenisation (代幣化) 補強

#### 週期性的直接扣款:

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

---

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
event SetupDirectDebit(address indexed debtor, address indexed receiver, DirectDebitInfo info);

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

---

要檢查直接扣款相關設定時:

- `directDebit(address,address)` 為查看直接扣款資訊的操作

```
function directDebit(address debtor, address receiver) public view returns (DirectDebit) {
  return accounts[debtor].instruments[receiver].directDebit;
}
```

例如設定了每期 10 代幣，開始時間為 2019-01-01T08:08:08.000Z，每期間隔為 2 天

```
 |     epoch 1     |     epoch 2     |     epoch 3     |     epoch 4     |
 |-----10token-----|-----10token-----|-----10token-----|-----10token-----|----------
 S              S+2days           S+4days           S+6days           S+8days
```

假如現在在 epoch N 的時間區段中，則直接扣款方就可以收取 epoch 1 ~ N 該扣到的款項，也就是可以累積，但不應會超過收取或重複

而假如扣款方的第一次扣款在 2019-01-05T08:08:08.000Z (S+4days) 這一瞬間，則一次可以扣款到 60 token，等於滿足到了 epoch 3，下次扣款只能在最早 2019-01-07T08:08:08.000Z (S+6days) 之後，也就是 epoch 4

---

扣款方在直接扣款的操作中:

- `withdrawDirectDebit(address)` 為扣款方指定被扣款方並進行扣款的操作，並會觸發 `Transfer(address,address,uint256)` 事件

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

---

一旦代幣擁有者想要撤銷某個地址的定期直接扣款，則直接將 `directDebit` 移除即可

- `TerminateDirectDebit(address,address)` 為代幣擁有者撤銷直接扣款權力時所發射的事件，透過 `terminateDirectDebit(address)` 觸發
- `terminateDirectDebit(address)` 為代幣擁有者撤銷直接扣款時的操作

```
event TerminateDirectDebit(address indexed debtor, address indexed receiver);

function terminateDirectDebit(address receiver) public returns (bool) {
  delete accounts[msg.sender].instruments[receiver].directDebit;

  emit TerminateDirectDebit(msg.sender, receiver);

  return true;
}
```

#### 一次性大量操作:

一次性的多個傳送代幣

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

---

一次性的多個直接扣款

- `WithdrawDirectDebitFailure(address,address)` 為當一次性多個直接扣款中，`strict = true` 時所發動的事件
- `withdrawDirectDebit(address[],bool)` 為扣款方要一次性多個直接扣款時，要填入 `address[] debtors` 被扣款方們的地址陣列，並且選擇 `bool strict`

`strict` 為 `true` 表示當其中一個人直接扣款失敗時，整個操作都會失敗，並 `revert()`

如 `strict` 為 `false` 表示有人失敗則發射 `WithdrawDirectDebitFailure(address,address)` 事件，而不中斷面向其他帳戶的直接扣款，方便鏈外環境可以偵測問題

並且為了正確顯示在區塊鏈瀏覽器上，任何成功的直接扣款都必須發射 `Transfer(address,address,uint256)` 事件

```
event WithdrawDirectDebitFailure(address indexed debtor, address indexed receiver);

function withdrawDirectDebit(address[] debtors, bool strict) public returns (bool result) {
  require(isDirectDebitEnable);

  Account storage receiverAccount = accounts[msg.sender];
  result = true;
  uint256 total;

  for (uint256 i = 0; i < debtors.length; i++) {
    address debtor = debtors[i];
    Account storage debtorAccount = accounts[debtor];
    DirectDebit storage debit = debtorAccount.instruments[msg.sender].directDebit;

    uint256 epoch = (block.timestamp.sub(debit.info.startTime) / debit.info.interval).add(1);
    uint256 amount = epoch.sub(debit.epoch).mul(debit.info.amount);

    require(amount > 0);

    uint256 debtorBalance = debtorAccount.balance;

    if (amount > debtorBalance) {
      if (strict) {
        revert();
      }
      result = false;
      emit WithdrawDirectDebitFailure(debtor, msg.sender);
    } else {
      debtorAccount.balance = debtorBalance - amount;
      total += amount;
      debit.epoch = epoch;

      emit Transfer(debtor, msg.sender, amount);
    }
  }

  receiverAccount.balance += total;
}
```

#### 代幣傳送委派、代幣轉發:

此為代幣化關鍵的一個介面，讓代幣的傳送不再需要以太幣當作手續費，而是以也付代幣當作手續費的轉變

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

以下我們來補充細節與做個總結

### ERC-20 補強

因為 ERC-20 只規範了介面標準，而缺乏實作方面的建議，我們把自己從頭實作 ERC-20 的時候發現的問題融入了這標準裡頭，我們認為對於終端使用者或企業使用者而言，降低成本與邏輯明確是必要的，所以做了諸多儲存時的優化，像是跟帳戶有關的就是一個 `accounts` 的映射表，讀取結構 (struct) 的位址並且拿出需要的資料們，總比讀取自數個映射表還來的輕量。

而關於增強安全用的部份，主要是針對 `spender` 會插隊攻擊代幣擁有者在指定新 `allowance` 的時候，沒有特別地檢查的話，從 1,000 指定至 500 的 `allowance` 的過程中，插隊攻擊就會先把 1,000 花掉，然後讓代幣擁有者額外又 `approve` 了 500，**1,000 -> 0 -> 500**

故安全版的要求就會是 `approve` 一個新值前，至少在 ERC-20 的介面下是 `allowance` 必須先回到 0，雖然安全但是變成要兩個交易了，我們更是建議使用有預想值 (`expectedValue`) 的那一個介面，畢竟 `increaseAllowance` 與 `decreaseAllowance` 也可能會被插隊攻擊。

### Service-Friendly (服務友善化) 補強

首先，我們故意讓函數名稱取為 `transferAndCall`，而非用函數多載，是因為 `transfer` 到一個任何一個帳戶，甚至是智能合約，本來就是被允許並且適合乎邏輯的，因為本來有些流程就是傳到智能合約，然後另外一方過一段時間後再進行下一步之類的流程。假如因太多代幣被誤傳到無法移動代幣的智能合約，而去禁止 ERC-20 的 `transfer` 傳代幣至智能合約，是有點本末倒置的，因為這不是 ERC-20 的錯，是代幣傳送者或者是服務提供者的錯。

故我們故意取了一個不同一點的名字來表達功能的不同，以及更重要的，目的、動機的不同。因為我們是為了讓代幣相關的交易流程能變得跟以太坊交易一樣直覺好規劃，所以才做了讓接收者智能合約可以連續動作的機制，並且因著覆蓋參數所以更可以確保沒有人可以濫用任何資源。想像一個以太坊交易可以做到一連串複雜的 `transferAndCall`，就像是把複雜的金流相關服務搬到了鏈上一般，複雜但不失其規劃過的流程，並且終究可以被自由地延伸、擴展，以達到服務型智能合約們的模組化。

這也是為什麼這個部份稱作為 **Service-Friendly (服務友善)**，因為對於服務型的智能合約們是非常好用的、可信任的，加上要串接鏈下的現有商業邏輯也能以一個以太坊交易解決的話，更是福音。

### Tokenisation (代幣化) 補強

對於一個金流而言，一定有來源跟目的地，以及提出金流請求是哪一方，來源可以數個，目的地也可以數個。我們可以得出以下最粗糙的 8 種金流組合:

1. 來源請求，一個來源，一個目的地
   > ERC-20 的 `transfer(addres,uint256)` 即可解決
2. 來源請求，一個來源，多個目的地
   > 一次性的多重傳送代幣 `transfer(uint256[])` 即可解決
3. 來源之一請求，多個來源，一個目的地
   > 因為其中一個來源請求就可以觸發多個來源一起傳給多個目的地的話，想必來源間一定是有授權的，所以這要用智能合約解決，透過做出一個 Escrow Box (履約保證、交割保證箱) 就可以了，而這種智能合約跟 `transferAndCall` 是非常搭配的，因為會有很多傳送代幣了之後還要觸發的情境
4. 來源之一請求，多個來源，多個目的地
   > 同 **3.**，因為需要來源間的授權，搭配一次性的傳送給多個目的地的功能，即可達成
5. 目的地請求，一個來源，一個目的地
   > 可以是 ERC-20 的 `approve(address,uint256)` 與 `transferFrom(address,address,uint256)` 搭配使用，因為想必也是得需要來源的授權的，也就是 `approve` 這個動作。但假如有週期性的需求，就要使用 `withdrawDirectDebit(address)` 了
6. 目的地之一請求，一個來源，多個目的地
   > 同 **3.**，因為需要來源的授權，搭配一次性的傳送給多個目的地的功能，即可達成
7. 目的地請求，多個來源，一個目的地
   > 一次性的多重直接扣款即可，`withdrawDirectDebit(address[],bool)`
8. 目的地之一請求，多個來源，多個目的地
   > 同 **3.**，因為需要來源間的授權，搭配一次性的傳送給多個目的地的功能，即可達成

最後，就是對於金流服務所做出的一個修正，也就是傳送代幣不應該花上以太幣，而即使 Account Abstraction 出來了之後，也會因只能服務提供者自行承擔以太坊成本而容易形成商業硬成本，故首先讓「自願者」能出來協助轉發交易才是首要重點。

自願者，也就是轉發者 (Relayer) 必須要能自行挑選正常、無錯的代幣傳送請求，並代為發送至鏈上，再讓智能合約檢查代幣傳送者指定了哪些事物，轉發者又做了哪些調整，並且避免代幣傳送者攻擊轉發者，反之也要避免轉發者攻擊代幣傳送者。

我們將攻擊模式與解決方式列出:

1. 轉發者會重複送出代幣傳送請求，或送出錯誤的代幣傳送請求至代幣智能合約
   > 新增 `nonce` 這個資料或手動增加 (`increaseNonce()`) 自己的 `nonce` 的方式可以解決，或即使代幣傳送請求失敗，`nonce` 亦會增加
2. 轉發者將此代幣傳送請求送至另一個剛好 `nonce` 也對應好的代幣智能合約
   > 新增 `tokenAddress` 資料至簽章中的方式可以解決
3. 代幣傳送者浪費轉發者的以太幣燃料費
   > 新增 `gasAmount` 資料至簽章中，假如代幣傳送者給得太低，轉發者一開始就能知道，故可以先於鏈下捨棄此代幣傳送請求。也因為為了讓此類別攻擊失效，轉發者無論如何都會送出可以得到 `fee` 的交易，故針對即使會失敗的代幣傳送請求也是會送出的，也就是說，代幣傳送者在編織代幣傳送請求前就應該要檢查好，也就能避免反過來代幣傳送者被浪費資源。
4. 轉發者故意以不足的以太幣燃料量拿到了 `fee` 但讓原有該做完的操作沒做完
   > 新增 `gasAmount` 資料至簽章中，因為代幣傳送者自己定義了這個數字，故可解決，即使這個數字錯了，也可以用 **1.** 的方法，或也因為燃料不足時會交易失敗，故還是可以解決

我們的實作都有將攻擊模式考慮於其中

### 總結

此代幣介面標準是稍微龐大複雜的，但我們是以一個區塊 (blocksize) 還可以裝得下，並且可以提供大量的延伸性、擴展性的智能合約作為我們的目標，也希望這也是一個讓服務型智能合約，或是說實用型代幣 (Utility Token) 真正能落地的一個介面

## Backwards Compatibility

<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

此代幣標準完全支援並相容 ERC-20 標準

## Test Cases

<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

經過來自鏈外的交易測試腳本，以及鏈上的測試智能合約測試
原始碼於: https://github.com/funderstoken/Service-Friendly-Token-Standard/blob/develop/ServiceFriendlyToken.sol

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

FundersToken (https://fstk.io) 所發行之 Funder Smart Token 為一種 Service-Friendly Token (服務友善型代幣)，也與多個 FundersToken 去中心平台的智能合約模組連接，形成強壯可靠的智能合約服務。

於 mainnet 的位址在: https://etherscan.io/address/0x51c028bc9503874d74965638a4632a266d31f61f#code

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
