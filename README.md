---
eip: <to be assigned>
title: Service-Friendly代幣Standard
author: Atkins Chang <atkins@fstk.io> (@AtkinsChang), Noel Bao <noel@fstk.io> (@noeleon930)
discussions-to: <URL>
status: Draft
type: Standard
category: ERC
created: 2018-07-31
requires: ERC-20

---

# Service-Friendly-Token-Standard

## Simple Summary

<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->
<!-- A standard interface for service-friendly tokens, which aims for a grounded代幣化environment for business. -->

這個代幣介面標準是為了讓代幣 (Token) 能更方便地與服務型智能合約或鏈下服務對接，並提供對於代幣是有善的開發環境、使用環境。

## Abstract

<!--A short (~200 word) description of the technical issue being addressed.-->

原本專注於群眾募資的代幣技術與市場，遭遇到了即將要轉形成實用型 (Utility Token) 的陣痛期，非常多的專案或企業遇到了代幣的智能合約功能不足的問題，難以支撐基本的商業模式並應用於更多現實世界的服務或產品。

以下的諸多介面設計中，都是基於商業在經歷健康的代幣化 (Tokenisation) 上常會需要的基本功能，以及我們 FundersToken 對於諸多代幣介面標準 (Token standard) 的理解跟改善，試圖建立原生代幣自主環境 (Native 代幣 environment)，即對於代幣運作是有善的環境。

以及 FundersToken 原創的代幣傳輸代轉 (Token transfer relay)，即使用智能合約模擬區塊鏈，可以讓終端使用者免於需要支付以太幣作為燃料費的限制。

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

我們將此介面標準中的功能們分成以下幾類:

1.  針對 ERC-20 做的補強
2.  針對服務友善的環境 (Service-Friendly) 所做出的補強
3.  針對健全的代幣化所做出的補強

ERC-20 作為最基本最普遍的代幣使用方式及儲存方式，著實被證明是一個可行的方向，但其中因著不同的實作方式，執行時所耗的燃料成本與數學上的安全性，就造成不少代幣遭遇到了濫用或服務停擺。

我們針對 `transfer` 與 `approve` 的實做方式進行了執行時間的優化與嚴格的數學檢查，以及如何儲存 `balance` 與 `allowance` 進行了小量規範。

關於何謂服務友善的環境，我們可以簡單地從金流與智能合約一開始的設計目的出發。

以太坊交易 (Transaction) 與金流的旅程:

    (EA) --[tx]-> (CA 1) --[msg]-> (CA 2) --[msg]-> ... --[msg]-> (A)

而 ERC-20 的 transaction 與金流的旅程:

    (EA) --[transfer]-> (A)
    或
    (EA) ---[approve]-> (CA)
    (EA) ------[call]-> (CA) --[transferFrom EA]-> (A)

> EA 是 External Account
>
> CA 是 Contract Account
>
> A 是 EA and CA

從上述即可看得出，代幣一開始就比以太幣 (Ether) 還要不方便使用，代幣是靠智能合約驅動出來的，智能合約的執行本身必須依循以太坊的 transaction 執行流程，導致代幣數字 `transfer` 流程的直覺理解與實際技術上的實做方式是不同的。

也因為代幣的帳本 (Ledger) 就在代幣的智能合約中，帳本裡的數字的變化就也要被包圍在代幣智能合約裡面，或者是驗證外部智能合約的邏輯使帳本的數字能被調動，但前者會讓代幣開發受阻，而後者會讓執行成本升高與安全性風險增高。

FundersToken 在提供模組化智能合約與代幣化服務時，在開發過程中體驗到相當多的代幣不方便的環境。我們的流程目標為:

    (EA) --[transfer]-> (CA 1) --[transfer or call]-> (CA 2) --[transfer or call]-> ... --[transfer or call]-> (A)

簡單來說，我們希望代幣的金流或業務流程可以像是以太坊原本的方式一樣自然，並且讓代幣相關的服務開發起來是簡單直覺的，而非受了太多 ERC-20 沒有解決到的阻礙，造成業務擴展受到影響。

為了達成這些目的，我們針對 ERC-223 或 ERC-827 的 `transferAndCall` 的實做方法與潛在威脅進行了優化與增強，其中，讓 `receiverContract` 也就是服務智能合約 (Service contract) 收到正確的代幣傳輸數字 (Value) 與正確的代幣傳送者 (Transfer sender)，讓 `receiverContract` 不會攻擊代幣代幣傳送者，詳細會在下面補充。

關於針對代幣化所做的補強，著重在移除手續費成本或時間成本，以及一些為代幣升級所作的準備，有以下兩個部份：

1.  使代幣支援週期性的被動操作
2.  使代幣在被操作時，終端使用者不用負擔以太坊手續費
3.  為代幣打造提供次層級資產，方便階層化各式服務
4.  為代幣的升級做好準備，例如凍結傳輸，或者換錢箱

目前被動操作在代幣上的實現為 `approve` 一個對象，使這個對象可以自行依照 `approval` 的量進行代幣操作，而當業務流程上有個定期扣款的需求，終端使用者會變得要手動定期進行 `approve` 一個對象，對此我們實做了定期的直接扣款機制，讓被扣款方可以一次設定週期性設定，讓扣款方可以定期操作，也支援一次對多方進行直接扣款。

## Specification

<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).

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
