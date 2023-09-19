# _Introduction of Flash Translation Layer_

# FTL

* FTL 是 SSD FW 內部一個重要的功能，它的位置介於前端 Host 與後端 Nand Flash 之間
* FTL 的誕生與 Nand Flash 的特性有關，主要是因為 Nand Flash Erase/Program 的單位不同，所以需要 FTL 來管理 lba pba 的對應

# Nand Flash

* Nand Flash Program 前需要先 Erase， Erase 的單位是 Block，Program 的單位是 Page
* 若資料使用 in-place update，會影響到整個 Block (先 Erase 再 Program)，而且 Erase 的速度遠大於 Program
* 使用 out-of-place update，會需要一個 logical to physical 的 table (L2P)

# L2P

* 最簡單的 L2P table，L2P 一對一對應，全部放在 dram 上
* 一個 lba 4 byte，對應一個 page 4K byte，所以是 1:1024 的比例。L2P 的大小是容量的 1/1024
* 有了 L2P，寫入的資料就能定位，讀取之前查詢 L2P table 就能知道要讀的 nand address 在哪裡
* L2P 是 in-place update，但是 data 是 out-of-place update，最終會寫滿，於是需要 Garbage Collection (GC)

# GC

* GC 是將同一個 Block (GC source) 上的 valid 資料挑出，重新寫到一個新的 Block (GC Destination)，這樣 GC Source 就會全部都是 invalid，可以 Erase 再重新使用
* 流程: init 檢查要做哪種 GC，挑 GC Destination，挑 GC Source，挑 valid lbn 出來，read/write，GC 結束的條件 (Free 夠多的 Source or 寫滿一個 Destination)，Update L2P，Free Source
* 除了產出 Free Block 的 GC，還有其他需求可以使用 GC 的流程。例如平衡 Erase Count 的 Wear leveling，避免讀取次數過多造成 Read Disturb，或是 Program / Read Fail 的 bad block 處理
* 挑 Destination 的考量: 可以像 Host Write 一樣挑 Erase Count 最小的 Block 來自然的消除 Wear leveling，或是為了把冷資料搬走而故意挑 Erase Count 大的 Block 來使用
* 挑 Source 的考量: 大部分的情況會挑 Valid 最少的 Block，這樣做一次 GC 就可以換到最多的 Free Block，但是也有其他考量，譬如 Wear leveling 會挑 Erase Count 小的冷資料的 Block
* 有時候挑太多 Source 會造成 Free Block 的產出速度太慢，影響 Host Write 的速度，所以可以設定一個門檻之後故意挑 Valid 多的當作 Source，讓 GC 可以結束 Free 出 Block
* 搜尋整份 L2P 來找 GC source 有哪裡是 valid 太慢了，所以使用一個紀錄 valid 的 table 來加速
* 省空間的方法用 P2L table，要求速度的話可以用 valid bitmap
* Host Write 和 GC 之間需要 Throttle，以避免 Host Write 的速度太慢，或 GC 太慢造成 Free Block 歸零

# Snapshot & Rebuild

* dram 上的 L2P table，斷電之後會遺失，下次上電就沒辦法找回資料
* Snapshot 機制，在特定的時間把 L2P table 寫入 Nand Flash 中，這樣斷電後就可以把 L2P table 讀回來
* 保留其他的 table (如: block info, status, sequence, directory) 有助於斷電後 table rebuild
* 如果遇到 table 讀不出來的情況，可能會需要一個 page 一個 page scan
* Rebuild 的時間有限制，超時的話作業系統可能會判定為異常，所以 reubild 的時間要盡量縮短
* GC 的 Destination Block 寫到一半斷電的話，可以把他放棄，只要資料還在 Source 上面就沒有問題
* Rebuild 時如果遇到 Host Write 和 GC Write 同時間寫入的情況，通常是優先保留 Host Write 的資料為最新的
* 企業級的 SSD 可以加上電容，斷電時就有比較充裕的時間儲存 table，平時也不用花太多時間在寫 table

# dramless

* 消費級的 SSD 為了節省成本，設計出不用 dram 的 SSD
* 沒有 dram 影響最大的是 L2P table，table 無法常駐，平常保存在 Nand Flash 裡，有需要的時候再把部分的 table 讀出來放到 Sram
* 當 SSD 越來越大，Sram 放不下 directory 的時候，就會需要再多一個 table 指向 directory table
* 因為從 Nand Flash 讀 table 要花很多時間，所以會設計 cache table 的機制，或是使用 HMB (Host Memory Buffer) 來暫存 table
* GC 的流程也要配合 dramless 的設計修改，因為能用的空間不多，從 Nand Flash 讀 table 很花時間，所以會先把要讀取的 Nand Addr 先排序過

# Zoned Namespace

* Zoned Namespace 是一種新的架構，不同於傳統的 NVMe SSD，Zoned Namespace SSD 有一套新的讀寫方式
* 把 Namespace 分割成數個相同大小的 Zone，每個 Zone 的 LBA 是連續的，而且只能循序寫，有其他的指令可以控制 Zone (reset, open, close, finish)
* 因為這種特殊的設定，Zoned Namespace SSD 的 L2P table 相較於傳統的架構，可以使用更少的空間
* Zoned Namespace SSD 由 Host 管理數據，只能循序的寫入，所以也可以減少 GC 的發生

# Conclusion

* FTL 是 SSD FW 內部一個重要的功能，他最主要的功能就是管理 L2P table
* 會需要 L2P table 的原因是因為 Nand Flash 的特性，Erase 和 Program 的單位不同所造成的
* 因為 data 是 out-of-place update，SSD 最終會充滿 invalid 的 data，所以需要 Garbage Collection
* GC 挑 Source 和 Destination，會因為不同的情況而有不同的策略
* Host Write 和 GC 之間的 Throttle 是很重要的，會影響到 SSD 速度的穩定
* 斷電的問題也是一個大考驗，FTL 設計的流程必須考慮重新上電後能不能把 table 讀回來
* dramless SSD 是目前消費級 SSD 的趨勢，需要考慮 L2P cache 和減少從 Flash 讀取 table 的次數
* Zoned Namespace SSD 可以大幅減少 L2P table 的使用空間，相對於傳統的 NVMe SSD，可以減少成本，而且性能也更穩定
