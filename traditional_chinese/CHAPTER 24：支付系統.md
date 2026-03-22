# 第11章 支付系統

在本章中，我們設計一個支付系統。近年來，電子商務在全球範圍內爆發式增長。支撐每一筆交易順利進行的核心，是一個可靠、可擴展且靈活的支付系統。

根據維基百科的定義：「支付系統是任何通過轉移貨幣價值來結算金融交易的系統。這包括實現交易所需的機構、工具、人員、規則、程序、標準和技術」[1]。

支付系統表面上容易理解，但對許多開發者而言卻充滿挑戰——因為一個微小的錯誤都可能導致巨額損失和信譽受損。不過別擔心！本章將一步步揭開支付系統的神秘面紗。

## 第1步 - 理解問題並確定設計範圍

支付系統對不同的人意味著不同的事。有些人認為它像 Apple Pay 或 Google Pay 這樣的數位錢包；而另一些人則認為它是處理支付的後端系統，如 PayPal 或 Stripe。因此，在設計之前，必須先明確需求。以下是候選人與面試官之間的典型對話：

**候選人**：我們要建構哪種支付系統？

**面試官**：假設你在為一個類似 Amazon.com 的電子商務應用設計支付後端。當客戶下單後，支付系統處理所有與資金流動相關的部分。

**候選人**：系統支援哪些支付方式？信用卡、PayPal、金融卡等？

**面試官**：實際系統會支援所有這些方式。但在本次設計中，我們以信用卡支付為例。

**候選人**：我們自己處理信用卡支付嗎？

**面試官**：否，我們使用第三方支付處理器，如 Stripe，Braintree 或 Square。

**候選人**：我們是否在系統中保存信用卡資訊？

**面試官**：不保存。由於涉及極高的安全性和合規要求，系統不直接儲存卡號，由第三方支付機構負責。

**候選人**：應用是全球性的么？是否需要多幣種和國際支付支援？

**面試官**：是的，假設應用面向全球使用者，但本次面試中我們僅假設使用一種貨幣。

**候選人**：系統每天處理多少筆交易？

**面試官**：每天約 100 萬筆。

**候選人**：是否需要支援賣家的打款流程（pay-out flow）？

**面試官**：是的，需要。

**候選人**：我認為我已經收集齊了所有的需求。還有什麼我應該注意的嗎？

**面試官**：有的。一個支付系統會與許多內部服務（例如會計，分析等）以及外部服務（例如支付服務提供商）互動。當某個服務出現故障時，我們可能會在不同的服務之間看到不一致的狀態。因此，我們需要執行**對帳（reconciliation）**並修復任何不一致的情況。這也是一個需求。

通過這些問題，我們對功能性和非功能性需求都有了清晰的認識。在本次面試中，我們重點設計一個支援以下功能的支付系統。

### 功能需求

- **收款流程（Pay-in flow）：** 支付系統代表賣家接收買家的付款。
- **打款流程（Pay-out flow）：** 支付系統向全球賣家發放款項。

### 非功能性需求

- **可靠性與容錯性：** 必須妥善處理支付失敗的情況。
- **對帳流程：** 系統需支援內部服務（支付、會計等）與外部服務（支付提供商）之間的的非同步對帳機制。該流程以非同步方式驗證各系統間支付資訊的一致性。

### 粗略估算

系統每天需要處理 1,000,000 筆交易，即

$$
\frac{1{,}000{,}000\ \text{transactions}}{10^5\ \text{seconds}} = 10\ \text{transactions per second (TPS)}
$$

也就是每秒約 $10$ 筆交易(TPS)。對典型資料庫而言，$10\ \text{TPS}$ 並不是一個高負載數字，這意味著本次設計重點在於**如何正確處理支付交易**，而不是追求高吞吐量。

## 第2步 - 提出高層設計並獲得認可

從宏觀層面來看，支付流程可以分為兩個階段，以反映資金流動的過程：

- **收款流程（Pay-in flow）**
- **打款流程（Pay-out flow）**

以 Amazon 為例：為買家下單後，資金首先流入 Amazon 的銀行帳戶，這就是收款流程。雖然資金暫時在 Amazon 的帳戶中，但大部分屬於賣家，Amazon 只是代管資金並收取服務費。當商品發貨，資金可以釋放時，Amazon 會將扣除手續費後的餘額從自己的銀行帳戶轉入賣家的帳戶，這一過程就是打款流程。

圖 1 展示了簡化的收款與打款資金流向。

![Figure11.1](../images/v2/chapter11/Figure11.1.png)

### 收款流程（Pay-in flow）

高層次的支付流入（Pay-in）流程設計圖如圖 2 所示。下面讓我們來看一下系統中每個元件的具體作用。

![Figure11.2](../images/v2/chapter11/Figure11.2.png)

#### 支付服務（Payment Service）

支付服務負責接收使用者的支付事件並協調整個支付過程。它首先會進行風險檢查（Risk Check），確保符合諸如反洗錢（AML）和反恐融資（CFT）等法規要求[2]，並檢測是否存在洗錢或恐怖融資等可疑行為。支付服務只會處理通過風險檢測的支付請求。風險檢測通常由專業的第三方服務提供，因為該領域複雜且專業性高。

#### 支付執行器（Payment Executor）

支付執行器負責通過支付服務提供商（PSP）執行單筆支付訂單。一個支付事件可能包含多個支付訂單。

#### 支付服務提供商（PSP）

PSP 負責將資金從帳戶 A 轉移到帳戶 B。在此簡化的場景中，PSP 從買家的信用卡帳戶中扣款。

#### 卡組織（Card Schemes）

卡組織是負責處理信用卡交易的機構，例如 Visa、MasterCard、Discover 等。它們構成了一個龐大而複雜的生態系統[3]。

#### 總帳系統（Ledger）

總帳系統記錄每筆支付交易的財務資訊。例如，當使用者向賣家支付 \$1 時，在帳簿中記為使用者帳戶借方 -\$1，賣家帳戶貸方 +\$1。總帳系統對於後續的財務分析、網站營收計算以及未來預測都至關重要。

#### 錢包系統（Wallet）

錢包系統維護商家的帳戶餘額，並可記錄每個使用者的累計支付金額。

如圖 2 所示，一個典型的收款流程如下：

1. 使用者點擊「下單」按鈕時，會生成一個支付事件並發送至支付服務。
2. 支付服務將支付事件儲存到資料庫。
3. 若一個支付事件包含多個支付訂單（例如一次結帳包含多個賣家的商品），支付服務會為每個訂單調用支付執行器。
4. 支付執行器將支付訂單資訊存入資料庫。
5. 支付執行器調用外部 PSP 處理信用卡支付。
6. 當支付執行成功後，支付服務更新錢包系統，記錄賣家的可用餘額。
7. 錢包服務將更新後的餘額資訊寫入資料庫。
8. 當錢包更新成功後，支付服務調用總帳系統（Ledger）進行帳務更新。
9. 總帳服務將新的帳目資訊追加寫入資料庫。

### 支付服務 API 設計

支付服務遵循 RESTful API 設計規範。

#### POST /v1/payments

執行一次支付事件（一次支付事件可包含多個訂單）。請求參數範例：

![Table11.1](../images/v2/chapter11/Table11.1.png)

其中每個 `payment_order` 的結構如下：

![Table11.2](../images/v2/chapter11/Table11.2.png)

**注意：**`payment_order_id` 在全域範圍內唯一。當支付執行器向第三方 PSP 發送支付請求時。該 `payment_order_id` 會被 PSP 用作去重標識，也稱「冪等鍵（idempotency key）」。

你可能注意到，「amount」欄位的資料類型是字串而不是 double。使用 double 並不合適，原因如下：

1. 不同系統在序列化與反序列化時可能存在精度差異，導致捨入誤差。
2. 數值可能極大（如日本 GDP 約為 \(5 \times 10^{14}\) 日元）或極小（如比特幣最小單位 satoshi = \(10^{-8}\)）。

因此建議在資料傳輸與儲存階段使用字串，僅在顯示或計算時轉換為數值。

#### GET /v1/payments/{id}

此介面根據 `payment_order_id` 返回單個支付訂單的執行狀態。

上述支付 API 設計與一些知名 PSP 的介面風格類似。若想了解更全面的支付 API 設計，可以參考 Stripe 的官方文檔 [5]

### 支付服務的資料模型

我們需要兩張表：**支付事件表（Payment Event）**和**支付訂單表（Payment Order)**。當我們為支付系統選擇儲存解決方案時，**效能通常不是最重要的因素**。相反，我們更關注以下幾個方面：

1. 經過驗證的穩定性。該儲存系統是否已被其他大型金融公司使用多年（例如超過 5 年），並獲得積極回饋。
2. 支援工具的豐富性，例如監控和排查工具。
3. 資料庫管理員（DBA）就業市場的成熟度。我們是否能招聘到有經驗的 DBA 是一個非常重要的考慮因素。

通常，我們更傾向於選擇支援 ACID 交易的傳統關聯式資料庫，而不是 NoSQL 或 NewSQL。支付事件表包含詳細的支付事件資訊。它的結構如下所示：

![Table11.3](../images/v2/chapter11/Table11.3.png)

![Table11.4](../images/v2/chapter11/Table11.4.png)

在我們深入研究這些表格之前，先來看一些背景資訊。

- **checkout_id** 是外鍵。一次結帳會建立一個支付事件（payment event），該事件可能包含多個支付訂單（payment orders）。
- 當我們調用第三方支付服務提供商（PSP）從買家的信用卡中扣款時，資金不會直接轉入賣家的帳戶。相反，資金會先進入電子商務網站的銀行帳戶，這個過程稱為收款流程（pay-in）。當滿足付款條件（pay-out condition）時（例如商品已交付），賣家會發起付款（pay-out），此時資金才會從電子商務網站的銀行帳戶轉入賣家的銀行帳戶。因此，在收款流程中，我們只需要買家的銀行卡資訊，而不需要賣家的銀行帳戶資訊。

在支付訂單表（表 4）中，*payment_order_status* 是一個枚舉類型（enum），用於保存支付訂單的執行狀態。執行狀態包括：NOT_STARTED（未開始）, EXECUTING（執行中）, SUCCESS（成功）,FAILED（失敗）。更新邏輯如下：

1. 支付訂單的初始狀態是 **NOT_STARTED**。
2. 當支付服務將支付訂單發送給支付執行器時，狀態更新為 **EXECUTING**。
3. 支付服務根據支付執行器的回應，將狀態更新為 **SUCCESS** 或 **FAILED**。

一旦支付訂單狀態為 **SUCCESS**，支付服務會調用錢包服務（wallet service）更新賣家的帳戶餘額，並將 **wallet_updated** 欄位更新為 **TRUE**。在此處我們簡化設計，假設錢包更新總是成功的。

完成後，支付服務的下一步是調用帳本服務（ledger service）更新帳本資料庫，並將 **ledger_updated** 欄位更新為 **TRUE**。

當相同 **checkout_id** 下的所有支付訂單都成功處理後，支付服務會將支付事件表（payment event table）中的 **is_payment_done** 欄位更新為 **TRUE**。通常會有一個定時任務（scheduled job）以固定間隔運行，用於監控正在進行中的支付訂單狀態。如果某個支付訂單在閾值時間內未完成，系統會發送警報，以便工程師進行調查。

### 複式記帳系統（Double-entry Ledger System）

在帳本系統中有一個非常重要的設計原則：複式記帳原則（也稱為雙重記帳原則或複式簿記 [6])。複式記帳系統是任何支付系統的基礎，也是確保帳目準確的關鍵。它會將每一筆支付交易記錄到兩個獨立的帳本帳戶中，金額完全相同：一個帳戶借記（debit），另一個帳戶貸記（credit）相同的金額（見表 5）。

![Table11.5](../images/v2/chapter11/Table11.5.png)

複式記帳系統規定：所有交易記錄的借貸總和必須為 0。少了一分錢，就意味著另一個帳戶多了一分錢。這種機制提供了端到端的了追溯性，並確保整個支付流程中的一致性。想了解如何實現複式記帳系統，可以參考 Square 的工程部落格：《不可變的複式記帳資料庫服務（immutable double-entry accounting database service）》[7]。

### 托管支付頁面（Hosted Payment Page）

大多數公司傾向於不在內部儲存信用卡資訊，因為一旦這樣做，就必須遵守各種複雜的法規，例如美國的支付卡行業資料安全標準（PCI DSS）[8]。為了避免直接處理信用卡資訊，公司通常使用由支付服務提供商（PSP）提供的托管支付頁面（hosted payment page）。對於網站來說，這種托管頁面通常以小工具（widget）或 iframe 的形式嵌入；對於移動應用，則可能是支付 SDK 提供的預建構頁面（pre-built page）。圖 3 展示了一個與 PayPal 集成的結帳流程範例。這裡的關鍵點是：PSP 提供的托管支付頁面會直接收集客戶的信用卡資訊，而不是通過我們自己的支付服務來處理。

![Figure11.3](../images/v2/chapter11/Figure11.3.png)

### 打款流程（Pay-out Flow）

付款流（Pay-out flow）的各個元件與付款流入（Pay-in flow）非常相似。兩者的主要區別在於：在付款流入中，我們使用 PSP（支付服務提供商）將資金從買家的信用卡轉入電子商務網站的銀行帳戶；而在付款流出中，我們使用第三方付款服務提供商（pay-out provider）將資金從電子商務網站的銀行帳戶轉入賣家的銀行帳戶。

通常，支付系統會使用第三方的應付帳款服務提供商（如 Tipalti [9]）來處理付款流出。因為在付款流出過程中，同樣涉及大量的帳務記錄和合規監管要求。

## 步驟 3：深入設計（Design Deep Dive）

在本節中，我們將重點探討如何讓支付系統更快、更可靠、更安全。在分散式系統中，錯誤與故障不僅可能發生，而且非常常見。例如，如果使用者多次點擊「支付」按鈕，會不會被重複扣款？如果由於網路問題導致支付中斷，又該如何處理？接下來，我們將深入分析以下關鍵主題：

- 與支付服務提供商（PSP）的集成
- 對帳（Reconciliation）
- 支付處理延遲的應對
- 內部服務之間的通訊方式
- 支付失敗的處理機制
- 精確一次投遞（Exactly-once Delivery）
- 一致性（Consistency）
- 安全性（Security）

### PSP 集成（PSP Integration）

如果支付系統能夠直接連接銀行或卡組織（如 Visa、MasterCard），則可以不依賴 PSP 完成支付。但這種方式成本高、要求嚴格，通常只有大型公司才會採用。對於大多數企業而言，系統會通過以下兩種方式之一與 PSP 集成：

1. 如果一家公司能夠安全地儲存敏感支付資訊，並選擇這樣做，那麼可以通過 API 將 PSP（支付服務提供商）集成到系統中。公司需要負責開發支付網頁、收集並儲存敏感的支付資訊；而 PSP 則負責與銀行或信用卡組織（如 Visa、MasterCard）建立連接。
2. 如果一家公司由於複雜的法規要求或安全性考慮而選擇不儲存敏感支付資訊，那麼 PSP 會提供一個托管支付頁面（hosted payment page），用於收集信用卡支付資訊並安全地將其儲存在 PSP 系統中。這是大多數公司採用的方式。

我們將使用圖 4 來詳細說明托管支付頁面的工作原理。

![Figure11.4](../images/v2/chapter11/Figure11.4.png)

為了簡化說明，我們在圖 4 中省略了支付執行器（payment executor）、帳本（ledger）和錢包（wallet）。

支付服務（payment service）負責協調整個支付流程。

1. 使用者在用戶端瀏覽器中點擊「結帳（checkout）」按鈕，用戶端將支付訂單資訊發送給支付服務。
2. 支付服務在收到支付訂單資訊後，會向 PSP（支付服務提供商）發送一個支付註冊請求（payment registration request）。該請求包含支付相關的資訊，例如：金額、幣種、支付請求的到期時間以及重定向 URL（redirect URL）。由於每個支付訂單只能註冊一次，因此該請求中包含一個 UUID 欄位，用於確保「僅一次註冊（exactly-once registration）」。這個 UUID 也被稱為 nonce [10]，通常它就是支付訂單的唯一 ID。
3. PSP 返回一個 token（令牌）給支付服務。這個 token 是 PSP 端的一個 UUID，用於唯一標識此次支付註冊。我們之後可以使用這個 token 來查詢支付註冊及其執行狀態。
4. 支付服務在調用 PSP 托管支付頁面（hosted payment page）之前，會先將 token 存入資料庫。
5. 一旦 token 被保存，用戶端就會顯示一個 PSP 托管的支付頁面。移動端應用通常通過 PSP 的 SDK 來實現此功能。這裡以 Stripe 的網頁集成（web integration）為例（見圖 5）。Stripe 提供一個 JavaScript 庫，用於顯示支付介面（payment UI）、收集敏感支付資訊，並直接調用 PSP 完成支付。所有敏感支付資訊都由 Stripe 收集，並不會傳入我們的支付系統。托管支付頁面通常需要以下兩部分資訊：
   1. 步驟 4 中獲取的 token。PSP 的 JavaScript 代碼使用該 token 從 PSP 的後端獲取支付請求的詳細資訊，其中一個重要資訊是需要收取的金額。
   2. 重定向 URL（redirect URL）。這是支付完成後跳轉的網頁 URL。當 PSP 的 JavaScript 代碼完成支付後，會將瀏覽器重定向到該 URL。通常，這個重定向 URL 是電子商務網站上的結帳狀態頁面，用於顯示支付結果。請注意，redirect URL 與第 9 步中的 webhook URL 不同 [11]。

![Figure11.5](../images/v2/chapter11/Figure11.5.png)

6. 使用者在 PSP（支付服務提供商）托管的網頁上填寫支付資訊，例如信用卡號、持卡人姓名、有效期等，然後點擊「支付」按鈕。PSP 隨即開始處理支付流程。
7. PSP 返回支付狀態。
8. 網頁隨後被重定向到重定向 URL（redirect URL）。在第 7 步中接收到的支付狀態通常會附加到該 URL 後面。例如，完整的重定向 URL 可能是：
   https://your-company.com/?tokenID=JIOUIQ123NSF&payResult=X324FSa
9. PSP 還會非同步地通過 webhook（回調）將支付狀態發送給支付服務。這個 webhook 是在系統初始配置 PSP 時註冊的 URL，用於讓 PSP 向支付系統報告支付結果。當支付系統通過 webhook 收到支付事件時，它會提取支付狀態資訊，並更新資料庫中 Payment Order（支付訂單）表的 `payment_order_status` 欄位。

到目前為止，我們介紹了托管支付頁面（hosted payment page）的「理想流程」。但在現實中，網路連接可能不穩定，上述九個步驟中的任何一個都可能失敗。有沒有系統化的方法來處理這些失敗情況？答案是：對帳（Reconciliation）。

### 對帳（Reconciliation）

當系統元件以非同步方式通訊時，無法保證訊息一定會被送達，也無法保證一定會收到回應。這種情況在支付業務中非常常見，因為支付系統通常使用非同步通訊來提高系統效能。外部系統（如 PSP 或銀行）也傾向於使用非同步通訊。那麼，在這種情況下我們該如何確保系統的正確性呢？

答案是：對帳（Reconciliation）。對帳是一種定期比較相關服務之間狀態的做法，用來驗證它們的資料是否一致。它通常是支付系統中的最後一道防線。

每天晚上，PSP 或銀行都會向其客戶發送一份結算檔案（Settlement File）。該檔案包含當天銀行帳戶的餘額以及當天發生的所有交易記錄。對帳系統會解析結算檔案，並將其與帳本系統（Ledger System）中的資料進行比對。下方的圖（圖 6）展示了對帳過程在整個支付系統中的位置。

![Figure11.6](../images/v2/chapter11/Figure11.6.png)

對帳還用於驗證支付系統內部的一致性。例如，帳本（ledger）和錢包（wallet）中的狀態可能會出現偏差，我們可以使用對帳系統來檢測這些差異。

為了修復在對帳過程中發現的不匹配（mismatch），我們通常依賴財務團隊進行人工調整（manual adjustment）。這些不匹配和調整通常分為三類：

1. **可分類且可自動調整的差異（Classifiable & Automatable）**。在這種情況下，我們知道差異的原因，也知道如何修復它，並且編寫程式自動執行調整是划算的。工程師可以自動化差異分類和調整兩個步驟。
2. **可分類但無法自動調整的差異（Classifiable but Non-Automatable）**，在這種情況下，我們知道差異的原因，也知道修復方法，但編寫自動調整程式的成本太高。這種差異會被放入任務佇列（job queue）中，由財務團隊手動修復。
3. **無法分類的差異（Unclassifiable）**，在這種情況下，我們不知道差異是如何產生的。這種差異會被放入特殊任務佇列（special job queue）中，由財務團隊進行人工調查和處理.

### 支付處理延遲（Handling Payment Processing Delays）

正如前面討論的那樣，一個端到端的支付請求會經過許多元件，並涉及內部和外部的多個系統。在大多數情況下，支付請求會在幾秒鐘內完成，但有時支付請求可能會卡住，甚至需要幾個小時或幾天才能完成或被拒絕。

以下是一些導致支付請求耗時較長的常見情況：

- PSP（支付服務提供商）認為該支付請求風險較高，需要人工審核。
- 信用卡需要額外的安全驗證，比如 3D Secure 認證 [13]，要求持卡人提供額外資訊以驗證購買行為。

支付服務必須能夠處理這些需要較長時間完成的支付請求。如果購買頁面由外部 PSP 托管（這在如今非常常見），PSP 會通過以下方式處理這些長時間運行的支付請求：

- PSP 會向用戶端返回一個待處理（pending）狀態。用戶端會將此狀態顯示給使用者，同時提供一個頁面，讓客戶可以隨時查看目前支付狀態。
- PSP 會代表我們追蹤該筆待處理的支付請求，並通過支付服務在註冊時提供的 webhook 回調地址通知任何支付狀態的更新。

當支付請求最終完成時，PSP 會調用前面提到的 webhook，支付服務收到通知後會更新內部系統，並繼續處理後續業務，例如向客戶發貨。

或者，某些 PSP 不使用 webhook 通知支付結果，而是要求支付服務主動輪詢（polling）PSP，以獲取所有待處理支付請求的最新狀態更新。

### 內部服務之間的通訊（Communication among internal services）

內部服務之間通常有兩種通訊模式：同步通訊（synchronous）和非同步通訊（asynchronous）。下面分別介紹這兩種方式。

#### 同步通訊（Synchronous communication）

同步通訊（例如 HTTP 調用）在小規模系統中運行良好，但隨著系統規模的擴大，其缺點會逐漸顯現。這種方式在多個服務之間形成了一個長的請求-回應鏈條，導致系統的整體效能和可靠性都受到限制。

同步通訊的主要缺點包括：

- **效能低（Low performance）**如果調用鏈中的某個服務效能不佳，整個系統的回應速度都會受到影響。
- **故障隔離差（Poor failure isolation）**如果 PSP（支付服務提供商）或其他依賴服務出現故障，用戶端將無法獲得回應。
- **耦合度高（Tight coupling）**請求的發送方必須知道接收方的具體資訊，這使得系統擴展或替換服務變得困難。
- **難以擴展（Hard to scale）**如果不使用訊息佇列作為緩衝層，系統就難以應對突發的大流量。

#### 非同步通訊（Asynchronous communication）

非同步通訊可以分為兩種類型：

- **單接收者：** 每個請求（訊息）只會被一個接收者或服務處理。這種模式通常通過共用訊息佇列（shared message queue）來實現。訊息佇列可以有多個訂閱者，但一旦某條訊息被處理完成，它就會從佇列中移除。我們來看一個具體的範例：在圖 9 中，服務 A 和服務 B 都訂閱了同一個共用訊息佇列。當訊息 m1 被服務 A 消費、訊息 m2 被服務 B 消費後，這兩條訊息都會從佇列中刪除，如圖 10 所示。

  ![Figure11.8](../images/v2/chapter11/Figure11.8.png)

- **多接收者：** 每個請求（訊息）會被多個接收者或服務處理。在這種場景下，Kafka 的表現非常出色。

  當消費者接收訊息時，訊息**不會**從 Kafka 中被移除，因此同一條訊息可以被不同的服務同時處理。這種模型非常適合支付系統，因為同一個請求可能會觸發多個「副作用」（side effects），例如：發送推送通知、更新財務報表、觸發分析統計等。如圖 11 所示，這是一個典型的例子：支付事件會被發布到 Kafka，然後被不同的服務（如支付系統、分析系統、帳單系統等）共同消費。

  ![Figure11.9](../images/v2/chapter11/Figure11.9.png)

一般來說，同步通訊（synchronous communication）的設計更簡單，但它無法讓服務實現真正的自治（autonomous）。隨著系統中依賴關係圖（dependency graph）的不斷擴大，整體效能會逐漸下降。非同步通訊（asynchronous communication）則在一定程度上犧牲了設計的簡潔性和資料一致性，以換取更高的可擴展性（scalability）和故障恢復能力（failure resilience）。對於一個業務邏輯複雜、依賴大量第三方服務的大型支付系統來說，非同步通訊無疑是更好的選擇。

### 支付失敗的處理（Handling Failed Payments）

每個支付系統都必須處理失敗的交易。可靠性和容錯性是關鍵要求。下面我們將回顧一些應對這些挑戰的技術。

#### 跟蹤支付狀態（Tracking payment state）

在支付週期的任何階段，擁有一個明確的支付狀態都是至關重要的。每當發生故障時，我們就能確定支付交易的目前狀態，並判斷是否需要重試或退款。支付狀態可以儲存在一個僅追加（append-only）的資料庫表中，以保證不可篡改的記錄。

#### 重試佇列與死信佇列（Retry queue and Dead letter queue）

為了優雅地處理失敗情況，我們使用重試佇列（retry queue）和死信佇列（dead letter queue），如圖 12 所示。

- **重試佇列（Retry queue）：** 可重試的錯誤（例如暫時性錯誤）會被路由到重試佇列中。
- **死信佇列（Dead letter queue）：** 如果一條訊息多次重試仍然失敗，它最終會被放入死信佇列中。死信佇列對於除錯（debugging）和隔離問題訊息（isolating problematic messages）非常有用，可以幫助我們檢查並確定這些訊息為什麼沒有被成功處理。

![Figure11.10](../images/v2/chapter11/Figure11.10.png)

1. 檢查故障是否可重試。
   - 1a. 可重試的故障會被路由到重試佇列中。
   - 1b. 對於不可重試的故障（例如無效輸入），錯誤資訊會被儲存到資料庫中。
2. 支付系統會從重試佇列中讀取事件，並對失敗的支付交易進行重試。
3. 如果支付交易再次失敗：
   - 3a. 如果重試次數未超過閾值，該事件會再次被路由到重試佇列。
   - 3b. 如果重試次數超過閾值，該事件會被放入死信佇列。這些失敗的事件可能需要進一步調查。

如果你對使用這些佇列的真實案例感興趣，可以了解一下 Uber 的支付系統，它使用 Kafka 來滿足可靠性和容錯性的要求 [16]。

### 精確一次投遞（Exactly-once Delivery）

支付系統可能遇到的最嚴重問題之一就是對客戶重複扣款。在系統設計中，必須保證支付系統能夠**「恰好執行一次」（exactly-once）**支付指令。[16]

乍一看，實現「恰好一次」似乎非常困難，但如果我們將問題拆分為兩個部分，就容易得多。從數學上講，一個操作若要「恰好執行一次」，必須同時滿足以下兩個條件：

1. 至少執行一次（at-least-once）；
2. 至多執行一次（at-most-once）。

我們將解釋如何通過「重試（retry）」實現「至少執行一次」，以及如何通過「冪等性檢查（idempotency check）」實現「至多執行一次」。

#### 重試（Retry）

有時由於網路錯誤或超時，我們需要重試支付交易。重試機制可以保證「至少執行一次」。例如，如圖 13 所示，用戶端嘗試進行一筆 10 美元的支付，但由於網路連接不良，請求不斷失敗。在此範例中，網路最終恢復正常，請求在第四次嘗試時成功

![Figure11.11](../images/v2/chapter11/Figure11.11.png)

決定重試之間的時間間隔非常重要。以下是一些常見的重試策略。

- **立即重試：** 用戶端立即重新發送請求。
- **固定間隔：** 在支付失敗與新重試嘗試之間等待固定的時間。
- **遞增間隔：** 用戶端在第一次重試前等待較短的時間，然後在隨後的重試中逐步增加等待時間。
- **指數退避（Exponential backoff）**[17]：在每次重試失敗後，將重試之間的等待時間加倍。例如，第一次請求失敗後，我們在 1 秒後重試；如果第二次失敗，我們在 2 秒後重試；第三次失敗，則在 4 秒後再試。
- **取消：** 用戶端可以取消請求。這在故障是永久性的，或重複請求成功的可能性很低時，是一種常見做法。

確定合適的重試策略並不容易。沒有一種「放之四海而皆準」的解決方案。一般來說，如果網路問題在短時間內不太可能解決，可以使用指數退避策略。過於頻繁的重試會浪費計算資源，並可能導致服務過載。一個良好的實踐是，在回應中提供一個帶有 Retry-After 頭部的錯誤代碼。

重試的一個潛在問題是重複支付。我們來看兩個場景。

**場景 1：**支付系統通過托管支付頁面與 PSP 集成，而用戶端連續點擊兩次「支付」按鈕。

**場景 2：**支付被 PSP 成功處理，但由於網路錯誤，回應未能到達我們的支付系統。使用者再次點擊「支付」按鈕，或者用戶端自動重試支付。

為了避免重複支付，支付操作必須最多執行一次。這種「最多一次執行」的保證也被稱為冪等性（idempotency）。

#### 冪等性

冪等性是確保「最多一次執行」保證的關鍵。根據維基百科的定義，冪等性是數學和電腦科學中某些操作的一種屬性，即該操作可以被多次執行，而不會改變第一次執行之後的結果。從 API 的角度看，冪等性意味著用戶端可以多次發起相同的調用，並且得到相同的結果。

在用戶端（網頁和移動應用）與伺服器之間的通訊中，冪等鍵（idempotency key）通常是一個由用戶端生成的唯一值，並在一段時間後過期。UUID 通常被用作冪等鍵，並被許多科技公司（如 Stripe 和 PayPal）推薦使用。要執行一個冪等的支付請求，可以在 HTTP 請求頭中添加冪等鍵，例如：<idempotency-key: key_value>。

現在我們了解了冪等性的基本概念，讓我們看看它如何幫助解決上面提到的重複支付問題。

**場景 1：**如果客戶快速點擊兩次「支付」按鈕，會怎樣？

如圖 14 所示，當使用者點擊「支付」按鈕時，一個冪等鍵會作為 HTTP 請求的一部分被發送到支付系統中。在電子商務網站中，冪等鍵通常是結帳前購物車的 ID。

對於第二個請求，系統會將其視為一次重試，因為支付系統已經見過相同的冪等鍵。當我們在請求頭中包含先前指定的冪等鍵時，支付系統會返回上一個請求的最新狀態。

![Figure11.12](../images/v2/chapter11/Figure11.12.png)

如果檢測到多個並發請求使用相同的冪等鍵（idempotency key），系統只會處理其中一個請求，其他請求將收到「429 Too Many Requests（請求過多）」的狀態碼。

為了支援冪等性，我們可以利用資料庫的唯一鍵約束。例如，可以將資料庫表的主鍵作為冪等鍵來使用。其工作原理如下：

1. 當支付系統接收到一筆支付請求時，它會嘗試向資料庫表中插入一條記錄。
2. 如果插入成功，說明這是一個新的支付請求，系統之前未處理過。
3. 如果插入失敗，是因為相同的主鍵已經存在，說明系統之前已經處理過這筆支付請求。此時第二個請求將不會被再次處理。

**場景 2：支付已被 PSP 成功處理，但由於網路錯誤，回應未能返回到我們的支付系統，然後使用者再次點擊「支付」按鈕。**

如圖 4（步驟 2 和步驟 3）所示，支付服務向 PSP 發送一個隨機數（nonce），PSP 返回一個對應的令牌（token）。這個隨機數唯一標識支付訂單，而令牌唯一對應這個隨機數，因此令牌也唯一映射到該支付訂單。

當使用者再次點擊「支付」按鈕時，支付訂單是相同的，因此發送給 PSP 的令牌也是相同的。由於 PSP 端使用該令牌作為冪等鍵，它能夠識別出這是一次重複支付，並返回上一次執行的狀態結果。

### 一致性（Consistency）

在一次支付執行過程中，會調用多個有狀態服務（stateful services）：

1. 支付服務（Payment Service）：保存與支付相關的資料，例如 nonce（隨機數）、token（令牌）、支付訂單、執行狀態等。
2. 帳本服務（Ledger）：保存所有會計資料。
3. 錢包服務（Wallet）：保存商家的帳戶餘額。
4. 支付服務提供商（PSP）：保存支付執行的狀態。
5. 為了提高可靠性，資料可能會在不同的資料庫副本（replicas）之間進行複製。

在分散式環境中，任意兩個服務之間的通訊都有可能失敗，從而導致資料不一致（data inconsistency）。下面我們來看一些用於解決支付系統中資料不一致問題的常用技術。

為了維持內部服務之間的資料一致性，確保僅執行一次（exactly-once）的處理非常重要。這意味著每一筆支付操作只能被處理一次，不會被重複執行，也不會被遺漏。

為了在內部服務與外部服務（PSP）之間保持資料一致性，我們通常依賴冪等性（idempotency）和對帳（reconciliation）。如果外部服務支援冪等性，那麼在支付重試操作時，我們應使用相同的冪等鍵（idempotency key）。即使外部服務支援冪等 API，仍然需要進行對帳，因為我們不應假設外部系統的結果總是正確的。

如果資料被複製，複製延遲可能會導致主資料庫與副本之間的資料不一致。通常有兩種方法可以解決這個問題：

1. 只使用主資料庫來處理讀寫請求。這種方法設置簡單，但显而易见的缺點是可擴展性較差。副本僅用於資料可靠性保障，但並不承擔流量，從而造成資源浪費。
2. 確保所有副本始終保持同步。我們可以使用，諸如 Paxos[21] 或 Raft[22] 這樣的共識演算法，或者使用基於共識的分散式資料庫，例如 YugabyteDB[23] 或 CockroachDB[24]。

### 支付安全（Payment Security）

支付安全非常重要。在本系統設計的最後部分，我們將簡要介紹一些用於防範網路攻擊和信用卡盜用的技術手段。

![Table16.](../images/v2/chapter11/Table11.6.png)

## 第4步 - 總結

在本章中，我們研究了收款流程（Pay-in flow）和打款流程（Pay-out flow）。我們深入探討了重試機制（retry）、冪等性（idempotency）以及一致性（consistency）。在本章的結尾，我們還討論了支付錯誤處理和安全性相關的內容。

支付系統極其複雜。儘管我們已經涵蓋了許多主題，但仍有一些值得進一步討論的內容。下面列出的是一些具有代表性但並不完整的相關主題。

- **監控（Monitoring）**: 監控關鍵指標是現代應用程式中非常重要的一部分。通過完善的監控，我們可以回答如下問題：「某種支付方式的平均成功率是多少？」、「我們伺服器的 CPU 使用率是多少？」等。我們可以將這些指標建立並展示在監控儀表板（dashboard）上
- **告警（Alerting）**: 當系統出現異常時，及時通知值班開發人員（on-call developers）非常重要，以便他們能夠迅速回應。
- **除錯工具（Debugging Tools）**: 「為什麼支付失敗？」是一個常見的問題。為了讓工程師和客服更容易排查問題，開發能夠查看交易狀態、處理伺服器歷史記錄、PSP 記錄等的工具非常重要。
- **貨幣兌換（Currency Exchange）**: 在為國際使用者群設計支付系統時，貨幣兌換是一個重要的考慮因素。
- **地理因素（Geography）**: 不同地區可能有完全不同的支付方式。
- **現金支付（Cash Payment）**: 現金支付在印度、巴西和其他一些國家非常普遍。Uber [28] 和 Airbnb [29] 曾撰寫過詳細的技術部落格，介紹他們是如何處理基於現金的支付方式的。
- **Google Pay / Apple Pay 集成。**: 如有興趣，可參考文獻 [30] 了解更多內容。

恭喜你讀完本章！現在請為自己鼓掌或拍一下肩膀表示讚賞。幹得漂亮！

### 章節總結

![Summary.png](../images/v2/chapter11/Figure11.13.png)

## 參考資料

[1] Payment system: https://en.wikipedia.org/wiki/Payment_system

[2] AML/CFT: https://en.wikipedia.org/wiki/Money_laundering

[3] Card scheme: https://en.wikipedia.org/wiki/Card_scheme

[4] ISO 4217: https://en.wikipedia.org/wiki/ISO_4217

[5] Stripe API Reference: https://stripe.com/docs/api

[6] Double-entry bookkeeping: https://en.wikipedia.org/wiki/Double-entry_bookkeeping

[7] Books, an immutable double-entry accounting database service:
https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/

[8] Payment Card Industry Data Security Standard:
https://en.wikipedia.org/wiki/Payment_Card_Industry_Data_Security_Standard

[9] Tipalti: https://tipalti.com/

[10] Nonce: https://en.wikipedia.org/wiki/Cryptographic_nonce

[11] Webhooks: https://stripe.com/docs/webhooks

[12] Customize your success page: https://stripe.com/docs/payments/checkout/custom-success-page

[13] 3D Secure: https://en.wikipedia.org/wiki/3-D_Secure

[14] Kafka Connect Deep Dive – Error Handling and Dead Letter Queues:
https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/

[15] Reliable Processing in a Streaming Payment System:
https://www.youtube.com/watch?v=5TD8m7w1xE0&list=PLLEUtp5eGr7Dz3fWGUpiSiG3d_WgJe-KJ

[16] Chain Services with Exactly-Once Guarantees:
https://www.confluent.io/blog/chain-services-exactly-guarantees/

[17] Exponential backoff: https://en.wikipedia.org/wiki/Exponential_backoff

[18] Idempotence: https://en.wikipedia.org/wiki/Idempotence

[19] Stripe idempotent requests: https://stripe.com/docs/api/idempotent_requests

[20] Idempotency: https://developer.paypal.com/docs/platforms/develop/idempotency/

[21] Paxos: https://en.wikipedia.org/wiki/Paxos_(computer_science)

[22] Raft: https://raft.github.io/

[23] YogabyteDB: https://www.yugabyte.com/

[24] Cockroachdb:https://www.cockroachlabs.com/

[25] What is DDoS attack: https://www.cloudflare.com/learning/ddos/what-is-a-ddos-attack/

[26] How Payment Gateways Can Detect and Prevent Online Fraud: https://www.chargebee.com/blog/optimize-online-billing-stop-online-fraud/

[27] Advanced Technologies for Detecting and Preventing Fraud at Uber: https://eng.uber.com/advanced-technologies-detecting-preventing-fraud-uber/

[28] Re-Architecting Cash and Digital Wallet Payments for India with Uber Engineering: https://eng.uber.com/india-payments/

[29] Scaling Airbnb's Payment Platform: https://medium.com/airbnb-engineering/scaling-airbnbs-payment-platform-43ebfc99b324

[30] Payments Integration at Uber: A Case Study: https://www.youtube.com/watch?v=yooCE5B0SRA
