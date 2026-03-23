# 第09章：網路爬蟲設計


本章重點介紹網路爬蟲設計：一道有趣且經典的系統設計面試題。

網路爬蟲被稱為機器人或蜘蛛。搜索引擎廣泛使用它來發現 Web 上的新內容或更新內容。內容可以是網頁、圖像、視訊、PDF 檔案等。網路爬蟲首先收集一些網頁，然後按照這些頁面上的連結收集新內容。圖 9-1 顯示了抓取過程的可視化範例。

![](../images/chapter9/figure9-1.jpg)

爬蟲有多種用途：

* 搜尋引擎索引：這是最常見的用例。爬蟲收集網頁以為搜尋引擎建立本地索引。例如，Googlebot 是 Google 搜尋引擎背後的網路爬蟲。
* 網路存檔：這是從網路收集資訊以儲存資料以備將來使用的過程。例如，許多國家圖書館運行爬蟲來存檔網站。著名的例子是美國國會圖書館 \[1] 和歐盟網路檔案館 \[2]。
* 網路挖掘：網路的爆炸式成長為資料挖掘提供了前所未有的機會。 網路挖掘有助於從 Internet 中發現有用的知識。例如，頂級金融公司使用爬蟲下載股東大會和年度報告以了解公司的關鍵舉措。
* 網路監控：這些爬蟲有助於監控 Internet 上的版權和商標侵權行為。例如，Digimarc \[3] 利用爬蟲來發現盜版作品和報告。

開發網路爬蟲的複雜性取決於我們打算支援的規模。它可以是一個只需要幾個小時即可完成的小型學校專案，也可以是一個需要專門的工程團隊不斷改進的大型專案。因此，我們將在下面探討支援的規模和功能。

### 第1步：了解問題並確定設計範圍

網路爬蟲的基本演算法很簡單：

* 給定一組URLs，下載所有由URLs指向的網頁。
* 從這些網頁中提取 URL。
* 將新的 URL 新增到要下載的 URL 列表中。重複這3個步驟。

網路爬蟲真的像這個基本演算法一樣簡單嗎？不完全是。設計一個高度可擴展的網路爬蟲是一項極其複雜的任務。任何人都不太可能在面試時間內設計出一個龐大的網路爬蟲。在進入設計之前，我們必須提出問題以**了解需求並建立設計範圍**：

候選人：爬蟲的主要目的是什麼？它用於搜尋引擎索引、資料挖掘或其他用途嗎？

面試官：搜尋引擎索引。

候選人：網路爬蟲每個月收集多少網頁？

採訪者：10 億頁。

候選人：包括哪些內容類型？僅 HTML 還是其他內容類型（如 PDF 和圖像）？

面試官：只有 HTML。

候選人：我們是否考慮新增或編輯的網頁？

面試官：是的，我們應該考慮新新增或編輯的網頁。

候選人：我們需要儲存從網路上爬取的 HTML 頁面嗎？

面試官：是的，最多5年

應聘者：我們如何處理內容重複的網頁？

面試官：重複內容的頁面應該忽略。

以上是您可以向面試官提出的一些範例問題。了解需求並釐清歧義很重要。即使你被要求設計一個簡單的產品，比如網路爬蟲，你和你的面試官可能不會有相同的假設。

除了要向面試官釐清的功能之外，記住優秀網路爬蟲的以下特徵也很重要：

* 可伸縮性（Scalability）：網路非常大。那裡有數十億個網頁。使用並行化網路爬行應該非常有效。
* 魯棒性（Robustness）：網路充滿了陷阱。錯誤的 HTML、無回應的伺服器、崩潰、惡意連結等都很常見。爬蟲必須處理所有這些邊緣情況。
* 禮貌（Politeness）：爬蟲不應該在短時間間隔內向網站發出太多請求。
* 可擴展性（Extensibility）：系統非常靈活，因此只需進行最少的更改即可支援新的內容類型。比如我們以後要抓取圖片檔案，應該不需要重新設計整個系統。

#### 粗略估算

以下估算基於許多假設，與面試官溝通以達成共識很重要。

* 假設每月下載 10 億個網頁。
* QPS： $$1,000,000,000 / 30 天 / 24 小時 / 3600 秒 = 400 頁/秒。$$
* $$峰值 QPS = 2 \times QPS = 800$$
* 假設平均網頁大小為 500k
* $$10 億頁 \times 500k = 每月 500 TB$$ 儲存空間。如果您對數字儲存單元不清楚，請重新閱讀第 2 章中的「2 的冪」部分。
* 假設資料儲存五年， $$500 TB \times 12 個月 \times 5 年 = 30 PB$$。需要 30 PB 的儲存來儲存五年的內容。

### 第2步：提出高層次的設計方案並獲得認同

一旦需求明確了，我們就開始進行高層設計。受以前關於網路抓取的研究\[4]\[5]的啟發，我們提出了一個高層設計，如圖9-2所示。

![](../images/chapter9/figure9-2.jpg)

首先，我們探索每個設計元件以了解它們的功能。然後，我們逐步檢查爬蟲工作流程。

#### Seed URLs

網路爬蟲使用種子 URL 作為爬網過程的起點。例如，要抓取大學網站的所有網頁，選擇種子 URL 的一種直觀方法是使用大學的網域。

要抓取整個網路，我們需要創造性地選擇種子 URL。一個好的種子 URL 是一個很好的起點，爬蟲可以利用它來遍歷盡可能多的連結。一般的策略是將整個 URL 空間分成更小的空間。第一個提議的方法是基於位置的，因為不同的國家可能有不同的流行網站.

另一種方法是根據主題選擇種子網址；例如，我們可以將 URL 空間劃分為購物、體育、醫療保健等。種子 URL 選擇是一個開放式問題，您不應該給出完美的答案，先大膽想想。

#### URL Frontier

大多數現代網路爬蟲將爬行狀態分為兩種：待下載和已下載。儲存待下載的URL的元件被稱為URL Frontier。你可以把它稱為先進先出（FIFO）佇列。關於URL Frontier的詳細資訊，請參考深入研究的內容。

#### HTML Downloader

HTML Downloader 從網際網路上下載網頁。這些URL是由URL Frontier提供的。

#### DNS Resolver

要下載網頁，必須將 URL 轉換為 IP 位址。 HTML Downloader 呼叫 DNS Resolver 為 URL 取得相應的 IP 位址。例如，截至 2019 年 3 月 5 日，URL [www.wikipedia.org](http://www.wikipedia.org/) 已轉換為 IP 位址 198.35.26.96。

#### Content Parser

下載網頁後，必須對其進行解析和驗證，因為格式錯誤的網頁可能會引發問題並浪費儲存空間。在爬網伺服器中實現內容解析器會減慢爬網過程。因此，Content Parser（內容解析器）是一個單獨的元件。

#### Content Seen?

線上研究\[6]顯示，29%的網頁是重複的內容，這可能導致同一內容被多次儲存。我們引入了「Content Seen? 」資料結構，以消除資料的冗餘，縮短處理時間。它有助於檢測以前儲存在系統中的新內容。為了比較兩個HTML文件，我們可以逐個字元進行比較。然而，這種方法既慢又費時，特別是當涉及到數十億的網頁時。完成這項任務的一個有效方法是比較兩個網頁的雜湊值\[7]。

#### Content Storage

它是一個用於儲存HTML內容的儲存系統。儲存系統的選擇取決於資料類型、資料大小、存取頻率、壽命等因素，磁碟和記憶體都被使用。

* 大部分內容儲存在磁碟上，因為資料集太大而無法放入記憶體。
* 熱門內容保存在記憶體中以減少延遲。

#### URL Extractor

URL Extractor（網址提取器） 從 HTML 頁面解析和提取連結。圖 9-3 顯示了連結提取過程的範例。通過添加「https://en.wikipedia.org」前綴將相對路徑轉換為絕對 URL。

![](../images/chapter9/figure9-3.jpg)

#### URL Filter

URL Filter 排除某些內容類型、檔案副檔名、錯誤連結和「黑名單」站點中的 URL。

#### URL Seen？

「URL Seen? 」是一個資料結構，用於追蹤之前被存取過的或已經在Frontier中的URL。「URL Seen? 」有助於避免多次新增相同的URL，因為這可能會增加伺服器負載並導致潛在的無限迴圈。

布隆過濾器和雜湊表是實現「URL Seen? 」元件的常用技術。我們不會在這裡介紹布隆過濾器和雜湊表的詳細實現。欲了解更多資訊，請參考參考資料\[4]\[8]。

#### URL Storage

URL Storage 儲存已經存取過的 URL。到目前為止，我們已經討論了每個系統元件。接下來，我們將它們放在一起來解釋工作流程。

#### 網路爬蟲工作流程

為了更好地逐步解釋工作流程，在設計圖中新增了序列號，如圖 9-4 所示。

![](../images/chapter9/figure9-4.jpg)

第 1 步：將種子 URL 新增到 URL Frontier

第 2 步：HTML 下載器從 URL Frontier 取得 URL 列表。

第 3 步：HTML 下載器從 DNS 解析器取得 URL 的 IP 位址並開始下載。

第 4 步：Content Parser 解析 HTML 頁面並檢查頁面是否格式錯誤。

第 5 步：內容經過解析和驗證後，傳遞給「Content Seen?」元件。

第 6 步：「Content Seen」元件檢查 HTML 頁面是否已在儲存中。

* 如果在儲存中，這意味著不同URL 中的相同內容已經被處理過。在這種情況下，HTML 頁面將被丟棄。
* 如果不在儲存中，則系統之前沒有處理過相同的內容。內容被傳遞給連結提取器。

第 7 步：網址提取器從 HTML 頁面中提取網址。

第 8 步：將提取的網址傳遞給 URL 過濾器。

第 9 步：網址過濾後，傳遞給「URL Seen?」元件。

第 10 步：「URL Seen」元件檢查一個URL是否已經在儲存中，如果是，則之前處理過，不需要做任何事情。

第 11 步：如果一個 URL 以前沒有被處理過，它被新增到 URL Frontier。

### 第3步：深入設計

到目前為止，我們已經討論了高層設計。接下來，我們將深入討論最重要的構建元件和技術：

* 深度優先搜尋 (DFS) vs 廣度優先搜尋 (BFS)
* URL Frontier
* HTML Downloader
* 魯棒性（Robustness）
* 可擴展性（Extensibility）
* 檢測並避免有問題的內容

#### DFS vs BFS

你可以把網路想像成一個有向圖，其中網頁作為節點，超連結（URL）作為邊。抓取過程可以被視為從一個網頁到其他網頁的有向圖的遍歷。兩種常見的圖形遍歷演算法是DFS和BFS。然而，DFS通常不是一個好的選擇，因為DFS的深度可能很深。

BFS 通常被網路爬蟲使用，並通過先進先出 (FIFO) 佇列實現。在 FIFO 佇列中，URL 按照它們入佇的順序出佇。但是，這種實現有兩個問題：

1.  來自同一網頁的大多數連結都連結回同一主機。在圖 9-5 中，[wikipedia.com](http://wikipedia.com/) 中的所有連結都是內部連結，使得爬蟲忙於處理來自同一主機（[wikipedia.com](http://wikipedia.com/)）的 URL。當爬蟲試圖並行下載網頁時，維基百科伺服器將被請求淹沒。這被認為是「不禮貌的」

    ![](../images/chapter9/figure9-5.jpg)
2. 標準的BFS沒有考慮到一個URL的優先級。網路很大，不是每個頁面都有相同的品質和重要性。因此，我們可能希望根據頁面排名、網路流量、更新頻率等來確定URL的優先級。

#### URL Frontier

URL Frontier 有助於解決這些問題。 URL Frontier 是一種儲存要下載的 URL 的資料結構。 URL Frontier 是確保禮貌、URL 優先級和新鮮度的重要組成部分。參考資料 \[5] \[9] 中提到了一些關於 URL Frontier 的值得注意的論文。這些論文的研究結果如下：

*   禮貌性

    一般來說，網路爬蟲應該避免在短時間內向同一個託管伺服器發送過多的請求。發送過多請求會被視為「不禮貌」，甚至被視為拒絕服務 (DOS) 攻擊。例如，在沒有任何限制的情況下，爬蟲可以每秒向同一個網站發送數千個請求。這會使 Web 伺服器不堪重負。

    強制禮貌的一般想法是一次從同一主機下載一個頁面。可以在兩個下載任務之間添加延遲。禮貌約束是通過維護從網站主機名稱到下載（工作）執行緒的映射來實現的。每個下載執行緒都有一個單獨的 FIFO 佇列，並且只下載從該佇列中取得的 URL。圖 9-6 顯示了管理禮貌的設計。

    ![](../images/chapter9/figure9-6.jpg)

    * Queue router：它確保每個佇列（b1，b2，... bn）僅包含來自同一主機的 URL。
    *   Mapping table：它將每個主機映射到一個佇列

        ![](../images/chapter9/table9-1.jpg)
    * FIFO 佇列 b1、b2 到 bn：每個佇列包含來自同一主機的 URL。
    * Queue selector：每個工作執行緒都映射到一個 FIFO 佇列，它只從該佇列下載 URL。佇列選擇邏輯由Queue selector完成
    * Worker thread 1 到 N：一個工作執行緒從同一台主機上一個接一個地下載網頁，可以在兩個下載任務之間添加延遲。
*   優先級

    一個關於 Apple 產品的討論論壇上的隨機貼文與蘋果首頁上的貼文具有非常不同的權重。儘管它們都有「Apple」這個關鍵字，但爬蟲首先抓取 Apple 首頁是明智之舉。

    我們根據實用性對 URL 進行優先排序，這可以通過 PageRank \[10]、網站流量、更新頻率等來衡量。「Prioritizer」是處理 URL 優先級的元件。有關此概念的深入資訊，請參閱參考資料 \[5] \[10]。

    圖 9-7 顯示了管理 URL 優先級的設計。

    ![](../images/chapter9/figure9-7.jpg)

    * Prioritizer：它將 URL 作為輸入並計算優先級。
    * Queue f1 到 fn：每個佇列都有一個分配的優先級。優先級高的佇列被選中的概率更高。
    * Queue selector：隨機選擇一個偏向於具有更高優先級的佇列

    圖 9-8 展示了 URL frontier 設計，它包含兩個模組：

    * 前端佇列：管理優先級
    * 後端佇列：管理禮貌

    ![](../images/chapter9/figure9-8.jpg)
*   新鮮度

    網頁不斷被新增、刪除和編輯。網路爬蟲必須定期重新抓取下載的頁面以保持我們的資料集最新。重新抓取所有 URL 既耗時又耗費資源。下面列出了幾種優化新鮮度的策略：

    * 根據網頁的更新歷史重新抓取。
    * 對URL進行優先排序，優先和頻繁地重新抓取重要頁面。
*   URL Frontier 儲存

    在搜尋引擎的真實世界抓取中，frontier 的 URL 數量可能達到數億 \[4]。將所有內容都放在記憶體中既不耐用也不可擴展。將所有內容都保存在磁碟中是不可取的，因為磁碟很慢；它很容易成為抓取的瓶頸。我們採用了混合方法。大多數 URL 都儲存在磁碟上，因此儲存空間不是問題。為了降低從磁碟讀取和寫入磁碟的成本，我們在記憶體中維護緩衝區以進行入佇/出佇操作。緩衝區中的資料會定期寫入磁碟。

#### HTML 下載器

HTML下載器使用HTTP協議從網際網路上下載網頁。在討論HTML下載器之前，我們先看一下Robots排除協議。

**Robots.txt**

Robots.txt，稱為Robots排除協議，是網站用來與爬蟲溝通的標準。它規定了爬蟲可以下載哪些頁面。在嘗試爬行一個網站之前，爬蟲應首先檢查其相應的robots.txt，並遵守其規則。為了避免重複下載 robots.txt 檔案，我們對該檔案的結果進行了快取。該檔案會定期下載並保存到快取中。下面是取自https://www.amazon.com/robots.txt 的robots.txt檔案的一個片段。一些目錄，如creatorhub，是不允許谷歌機器人存取的。

```http
User-agent: Googlebot
Disallow: /creatorhub/*
Disallow: /rss/people/*/reviews
Disallow: /gp/pdp/rss/*/reviews
Disallow: /gp/cdp/member-reviews/
Disallow: /gp/aw/cr/
```

除了 robots.txt，效能優化是我們將為HTML下載器介紹的另一個重要概念。

**效能優化**

以下是HTML下載器的效能優化列表.

1.  分散式抓取

    為了實現高效能，抓取工作被分配到多個伺服器，每個伺服器運行多個執行緒。URL空間被分割成更小的部分；因此，每個下載器負責URL的一個子集。圖9-9顯示了一個分散式抓取的例子。

    ![](../images/chapter9/figure9-9.jpg)
2.  快取DNS解析器

    DNS解析器是爬蟲的一個瓶頸，因為由於許多DNS介面的同步性，DNS請求可能需要時間。DNS回應時間從10ms到200ms不等。一旦爬蟲執行緒對DNS進行了請求，其他執行緒就會被阻斷，直到第一個請求完成。維護我們的DNS快取以避免頻繁呼叫DNS是一種有效的速度優化技術。我們的DNS快取保持網域名稱到IP位址的映射，並通過cron作業定期更新。
3.  位置

    按地理分佈抓取伺服器。當爬行伺服器離網站主機較近時，爬行者會體驗到更快的下載時間。設計定位適用於大多數系統元件：抓取伺服器、快取、佇列、儲存等。
4.  短暫的超時

    有些網路伺服器回應緩慢，或者根本不回應。為了避免漫長的等待時間，指定了一個最大的等待時間。如果一個主機在預定的時間內沒有反應，爬蟲將停止工作並抓取一些其他的網頁。

#### 魯棒性

除了效能優化，魯棒性也是一個重要的考慮因素。我們提出了一些提高系統魯棒性的方法。

* 一致性哈希：這有助於在下載者之間分配負載。 可以使用一致性哈希新增或刪除新的下載伺服器。 有關詳細資訊，請參閱第 5 章：設計一致性哈希。
* 儲存爬行狀態和資料：為了防止失敗，爬行狀態和資料被寫入儲存系統。 通過載入儲存的狀態和資料，可以輕鬆地重新啟動中斷的爬網。
* 異常處理：錯誤在大型系統中是不可避免的，也是常見的。 爬蟲必須在不使系統崩潰的情況下優雅地處理異常。
* 資料校驗：這是防止系統出錯的重要措施。

#### 可擴展性（Extensibility）

差不多每個系統都在不斷發展，設計目標之一是使系統足夠靈活，以支援新的內容類型。抓取器可以通過插入新的模組來擴展。圖9-10顯示了如何新增新模組。

![](../images/chapter9/figure9-10.jpg)

* PNG下載器模組是用於下載PNG檔案的插件。
* 新增了網路監控模組，以監控網路並防止版權和商標侵權。

#### 檢測並避免有問題的內容

本節討論冗餘、無意義或有害內容的檢測和預防。

1.  冗餘內容

    如前所述，近30%的網頁是重複的。雜湊值或校驗和有助於檢測重複\[11]。
2.  搜尋引擎蜘蛛陷阱

    搜尋引擎蜘蛛陷阱是導致爬蟲陷入無限迴圈的網頁。 例如，一個無限深的目錄結構如下：`http://www.spidertrapexample.com/foo/bar/foo/bar/foo/bar/...` 可以通過設定 URL 的最大長度來避免此類蜘蛛陷阱。但是，不存在檢測蜘蛛陷阱的萬能解決方案。 包含蜘蛛陷阱的網站很容易識別，因為在此類網站上發現的網頁數量異常多。 很難開發自動演算法來避免蜘蛛陷阱； 但是，使用者可以手動驗證和識別蜘蛛陷阱，並從爬蟲中排除這些網站或應用一些自訂的 URL 過濾器。
3.  垃圾資料

    有些內容價值很小或沒有價值，例如廣告、程式碼片段、垃圾郵件 URL 等。這些內容對爬蟲沒有用，應盡可能排除。

### 第4步：總結

在本章中，我們首先討論了一個好的爬蟲的特徵：可伸縮性、禮貌性、可擴展性和健壯性。 然後，我們提出了設計方案並討論了關鍵元件。 構建可擴展的網路爬蟲並不是一項簡單的任務，因為網路非常龐大且充滿陷阱。 即使我們涵蓋了所有主題，我們仍然遺漏了許多相關的討論要點：

* 伺服器端渲染：眾多的網站使用JavaScript、AJAX等腳本來即時生成連結。如果我們直接下載並解析網頁，我們將無法檢索到動態生成的連結。為了解決這個問題，我們在解析網頁之前先進行伺服器端的渲染（也叫動態渲染）\[12]。
* 過濾不需要的頁面：有限的儲存容量和抓取資源，反垃圾資訊元件有利於過濾掉低品質和垃圾頁面 \[13] \[14]。
* 資料庫複製和分片：複製和分片等技術用於提高資料層的可用性、可擴展性和可靠性。
* 水平擴展：對於大規模爬取，需要數百甚至數千台伺服器來執行下載任務。 關鍵是保持伺服器無狀態。
* 可用性、一致性和可靠性。這些概念是任何大型系統成功的核心。我們在第1章中詳細討論了這些概念。重温一下你對這些主題的記憶。
* 分析：收集和分析資料是任何系統的重要組成部分，因為資料是微調的關鍵要素。

恭喜你走到了這一步！現在給自己一個鼓勵，幹得漂亮！

### 參考資料

* \[1] US Library of Congress: [https://www.loc.gov/websites/](https://www.loc.gov/websites/)
* \[2] EU Web Archive: [http://data.europa.eu/webarchive](http://data.europa.eu/webarchive)
* \[3] Digimarc: [https://www.digimarc.com/products/digimarc-services/piracy-intelligence](https://www.digimarc.com/products/digimarc-services/piracy-intelligence)
* \[4] Heydon A., Najork M. Mercator: A scalable, extensible web crawler World Wide Web, 2 (4) (1999), pp. 219-229
* \[5] By Christopher Olston, Marc Najork: Web Crawling. [http://infolab.stanford.edu/\~olston/publications/crawling\_survey.pdf](http://infolab.stanford.edu/\~olston/publications/crawling\_survey.pdf)
* \[6] 29% Of Sites Face Duplicate Content Issues: [https://tinyurl.com/y6tmh55y](https://tinyurl.com/y6tmh55y)
* \[7] Rabin M.O., et al. Fingerprinting by random polynomials Center for Research in Computing Techn., Aiken Computation Laboratory, Univ. (1981)
* \[8] B. H. Bloom, "Space/time trade-offs in hash coding with allowable errors," Communications of the ACM, vol. 13, no. 7, pp. 422–426, 1970.
* \[9] Donald J. Patterson, Web Crawling: [https://www.ics.uci.edu/\~lopes/teaching/cs221W12/slides/Lecture05.pdf](https://www.ics.uci.edu/\~lopes/teaching/cs221W12/slides/Lecture05.pdf)
* \[10] L. Page, S. Brin, R. Motwani, and T. Winograd, "The PageRank citation ranking: Bringing order to the web," Technical Report, Stanford University, 1998.
* \[11] Burton Bloom. Space/time trade-offs in hash coding with allowable errors. Communications of the ACM, 13(7), pages 422--426, July 1970.
* \[12] Google Dynamic Rendering: [https://developers.google.com/search/docs/guides/dynamic-rendering](https://developers.google.com/search/docs/guides/dynamic-rendering)
* \[13] T. Urvoy, T. Lavergne, and P. Filoche, "Tracking web spam with hidden style similarity," in Proceedings of the 2nd International Workshop on Adversarial Information Retrieval on the Web, 2006.
* \[14] H.-T. Lee, D. Leonard, X. Wang, and D. Loguinov, "IRLbot: Scaling to 6 billion pages and beyond," in Proceedings of the 17th International World Wide Web Conference, 2008.
