# Holien Secrets Manager 規格文件

| 項目 | 內容 |
|------|------|
| 文件版本 | 0.10.0（草稿） |
| 狀態 | 待審閱 |
| 最後更新 | 2026-10-28 |
| 適用範圍 | 全專案（單機桌面應用程式、CLI、Google Drive 儲存層） |

> 本文件為專案起始規格。專案目前為空專案，文件中所有「假設」與「待決議」項目皆需在開發前確認，詳見第 13 章與附錄 A。
>
> 0.2.0 版重大變更：由多使用者伺服器架構改為**個人使用、本機執行、資料加密後存放於個人 Google Drive** 的架構。
>
> 0.3.0 版變更：明確定位為**單機桌面應用程式**（圖形介面為主、CLI 為輔），並加入**從 Chrome 匯入已儲存的網址與帳密**。
>
> 0.4.0 版變更：Drive 上的**資料夾名稱與所有檔案名稱皆由使用者自訂**，系統只提供預設值；檔案辨識改以 Drive 檔案屬性為準，不依賴檔名。
>
> 0.5.0 版變更：新增 5.1.1 至 5.1.6，逐步說明存放檔案的**加密與解密流程**、金鑰推導、主密碼變更與復原。
>
> 0.10.0 版變更（依第五輪外部審閱意見）：每把 VK 有全域唯一 `key_id`，資料修訂標頭改記 `key_id`，世代號只負責排序；輪替開始時在標頭記錄各邏輯檔案 head 的**快照**，遷移只處理快照內的 head；**子修訂的世代低於父修訂一律拒絕**；外洩輪替中快照之外出現的舊世代修訂只**隔離**不自動遷移；定義並行輪替的合併（保留雙方金鑰）與標頭的多父合併格式；解鎖、改密碼、復原金鑰、agent 全部改為處理「所有未清除金鑰的集合」，區分 top（MAC）與 active（寫入）；新增三個驗收案例。
>
> 0.9.0 版變更（依第四輪外部審閱意見）：空 trailer 的雜湊統一為真正的 SHA-256(空)；**取消輪替不再清除新世代**，被取消的世代以 `retired` 留在標頭，等下一次前進的輪替 settle 時一併清除，使「MAC 用最高未清除世代」在每個轉換都成立；狀態機改以（active 世代, retired 集合）描述；標頭驗證改為**從信任點沿雜湊鏈逐步檢查每個相鄰轉換**，並說明授權的錨點是 head 的 MAC 金鑰必須能用本機密碼從 head 自己的包裝解出；回滾判定改為「沒有任何 head 是信任點的後代」，有共同祖先的兄弟分支進入正常衝突處理；同步區分「曾驗證的歷史」與「本次遠端存在的修訂」，遠端刪除可被偵測；新增五個驗收案例。
>
> 0.8.0 版變更（依第三輪外部審閱意見）：head 判定改為**只有已驗證的子修訂才能取消父修訂的 head 身分**，所有新候選一律下載驗證；祖先關係改為**只靠雜湊鏈驗證、不需解密**（子修訂的 `parent_hash` 覆蓋父修訂完整 bytes），合併父節點移到明文 trailer；標頭加入**狀態轉換授權表**，MAC 金鑰由轉換決定，舊世代金鑰無法簽出降級標頭，因外洩而起的輪替不可取消；交接確認改為綁定**輪替操作 ID 與方向**並要求 pending 已清；**世代編號永不重用**，世代條目只標記 `purged` 不移除；修正「舊憑證無法解開任何東西」與事故清單的說法；新增五個驗收案例。
>
> 0.7.0 版變更（依第二輪外部審閱意見）：Vault 標頭加入以 VK 衍生金鑰計算的 **MAC**，解開 VK 後必須先驗證整份標頭才採用修訂鏈、世代狀態與命名規則；修訂鏈改為**雜湊鏈**（每個修訂的標頭含父修訂雜湊），祖先關係只由已驗證的修訂內容建立，appProperties 只用於搜尋候選；全面輪替改為**狀態機**，輪替前必須重新取得並視外洩原因替換憑證，舊世代金鑰保留到所有裝置確認後才清除，取消流程同樣保留兩把金鑰；第一版 **不清理任何修訂**，歷史清理延至有「經驗證檢查點」設計之後；標頭發現改以 Vault ID 分組，多個歷史標頭為正常；衝突未解決時禁止敏感操作；備份外層長度設上限；全面輪替提前至 Phase 1a。
>
> 0.6.0 版變更（依外部審閱意見）：改為**三層金鑰結構**（主密碼與復原金鑰各自包裝同一把 Vault Key），修正改密碼會使復原金鑰失效的矛盾；「變更主密碼」與「全面輪替金鑰」分為兩種操作，後者採世代與明確提交點；備份檔改為**可獨立解鎖**的格式；統一加密檔案格式並對長度與 KDF 參數設硬性上限；同步改為**每次寫入產生不可覆寫的新版本**並偵測分支與回滾；Windows 改以 ACL 保護檔案；新增 GUI 與 agent 強化、CLI 移除命令列明文密碼、金鑰版本與主機部署狀態分離；縮小第一版範圍並加入復原導向的驗收測試。

---

## 目錄

1. [專案概述](#1-專案概述)
2. [名詞定義](#2-名詞定義)
3. [使用情境](#3-使用情境)
4. [功能需求](#4-功能需求)
5. [安全設計](#5-安全設計)
6. [系統架構](#6-系統架構)
7. [Google Drive 整合](#7-google-drive-整合)
8. [資料格式](#8-資料格式)
9. [介面規格](#9-介面規格)
10. [非功能需求](#10-非功能需求)
11. [建議技術選型](#11-建議技術選型)
12. [開發階段規劃](#12-開發階段規劃)
13. [待決議事項](#13-待決議事項)
- [附錄 A：假設清單](#附錄-a假設清單)
- [附錄 B：錯誤與結束碼](#附錄-b錯誤與結束碼)

---

## 1. 專案概述

### 1.1 目標

Holien Secrets Manager（以下簡稱 HSM，不與硬體安全模組混淆，見第 2 章）是一套**個人使用的單機桌面應用程式**，在使用者自己的電腦上執行，沒有伺服器端，將資料加密後存放在使用者自己的 Google Drive，管理下列兩大類機密：

1. **帳號密碼（Credential）**：網站與服務的登入帳密、API Token、資料庫連線字串等。
2. **伺服器金鑰（Key Pair）**：個人管理的伺服器所使用的非對稱金鑰對，包含 SSH 金鑰與 TLS 憑證私鑰，涵蓋產生、匯入、儲存、取用、輪替、撤銷與綁定主機的完整生命週期。

### 1.2 要解決的問題

- 帳密與伺服器私鑰散落在多台電腦的 `~/.ssh`、筆記、瀏覽器中，換電腦或重灌時容易遺失。
- 多年累積在 Chrome 密碼管理器裡的網址與帳密，想一次搬進自己掌控、自己加密的地方。
- 想利用現成的 Google Drive 做跨裝置同步與備份，但不希望 Google 或任何取得 Drive 存取權的人能讀到內容。
- 伺服器金鑰缺乏盤點與輪替紀錄，不確定哪把金鑰放在哪台主機上。

### 1.3 核心設計：三個不變條件

1. **資料永遠在上傳前於本機加密**。Google Drive 上只存在密文，檔名與內容皆不洩漏機密與其 metadata。
2. **主密碼與由其推導的金鑰永不離開本機**，不存於 Google Drive、不傳送到任何伺服器。
3. **沒有任何第三方伺服器**。工具只與 Google Drive API 溝通，不依賴專案方架設的任何服務。

### 1.4 範圍

**納入範圍（In Scope）**

- 單一使用者的機密資料加密儲存、版本管理。
- 以個人 Google 帳號授權，資料存放於 Google Drive 指定資料夾；資料夾不存在時自動建立。資料夾與檔案名稱皆可由使用者自訂，系統提供預設值。
- 多台裝置透過同一 Google 帳號與同一主密碼存取同一份資料，含同步與衝突處理。
- SSH / TLS 金鑰對的產生、匯入、匯出、指紋計算、輪替、撤銷、主機綁定。
- 從 Chrome 匯入已儲存的網址、帳號與密碼。
- 本機離線快取與本機操作紀錄。
- 桌面圖形介面（主要介面）與 CLI（輔助，供終端機與腳本使用），兩者共用同一核心程式庫。

**排除範圍（Out of Scope）**

- 多使用者、團隊共享、權限管理。
- 自架伺服器、伺服器端 API。
- 自動將金鑰推送部署到目標伺服器（第三階段評估）。
- 憑證簽發（CA）功能。
- 瀏覽器擴充套件與自動填表（匯入 Chrome 資料是一次性搬移，不是持續同步）。
- Google Drive 以外的儲存後端（設計上保留儲存層介面，但第一版只實作 Google Drive）。

---

## 2. 名詞定義

| 名詞 | 定義 |
|------|------|
| HSM | 本專案 Holien Secrets Manager 之簡稱。**不是**硬體安全模組。 |
| Item（項目） | 系統中任何被加密保存的資料單位，有型別（type）：帳密、通用、金鑰對、TLS 憑證。 |
| Credential（帳密） | Item 的一種型別，由帳號、密碼、網址、備註等欄位組成。 |
| Key Pair（金鑰對） | Item 的一種型別，包含私鑰、公鑰、演算法、指紋等欄位。 |
| Host（主機） | 一台伺服器的紀錄，可與多把金鑰綁定。 |
| Vault（保險庫） | 一份完整資料集，對應 Google Drive 上的一個資料夾。一位使用者通常只有一個 Vault，但允許多個（例如工作與個人分開）。 |
| Version（版本） | Item 的值每次變更即產生一個新版本，舊版本保留供回溯。 |
| Master Password（主密碼） | 使用者記憶的唯一密碼，用於推導 PK。主密碼與復原金鑰皆遺失即無法解密資料。 |
| Recovery Key（復原金鑰） | 建立 Vault 時產生的隨機金鑰，可在遺忘主密碼時解開 VK。選用但預設啟用。 |
| PK | Password Key，由主密碼經 Argon2id 推導的 256-bit 金鑰，只用來包裝 VK。 |
| RK_key | 由復原金鑰經 HKDF 推導的 256-bit 金鑰，只用來包裝 VK。 |
| VK | Vault Key，建立 Vault 時隨機產生的 256-bit 金鑰，平常不變，用於包裝每個檔案的 DEK。主密碼與復原金鑰各自包裝同一把 VK。 |
| DEK | Data Encryption Key，每個檔案版本一把的隨機 256-bit 金鑰，用於實際加密內容。 |
| key_id | 每把 VK 的全域唯一識別（UUID v4，建立該 VK 時產生）。資料修訂標頭以 key_id 指明包裝 DEK 的金鑰。 |
| 金鑰世代（Generation） | 每把 VK 附帶的整數，只用於排序與狀態機；兩把不同的 VK 可能因並行輪替而有相同世代號，因此世代號不是識別。只有「全面輪替」會產生新世代；變更主密碼不會。 |
| top / active | top：標頭中最高世代的未清除金鑰，用於標頭 MAC。active：狀態機指定的寫入金鑰，用於新資料修訂的 DEK 包裝。兩者在 reverting 狀態與取消後會不同，文件中不再混稱「目前世代」。 |
| 金鑰集合（Key Set） | 標頭中所有未清除的金鑰（含 active、retiring、retired、reverting）。解鎖、改密碼、復原金鑰、agent 一律對整個集合操作。 |
| Rev（修訂） | 一個檔案的一次寫入。每次寫入都在 Drive 上建立新檔案，舊修訂不被覆寫；修訂以 `parent_rev` 與 `parent_hash`（父修訂整個檔案的 SHA-256）串成雜湊鏈。 |
| 信任點（Trust Point） | 本機裝置為每個邏輯檔案記錄的「最後一個已驗證並採用的修訂」：rev 與整個檔案的 SHA-256。新 head 必須能沿雜湊鏈驗證回到信任點。 |
| 已驗證修訂（Verified Rev） | 兩種方式之一通過驗證的修訂：(a) 內容驗證：以目前可用的 VK 解密成功（AES-GCM）；(b) 關係驗證：其完整 bytes 的 SHA-256 等於某個已驗證子修訂所宣告的 `parent_hash`（或 trailer 中的合併父雜湊）。關係驗證不需要解密，因此不需要舊世代 VK。 |
| 輪替操作（Rekey Op） | 一次全面輪替的識別，含唯一 `op_id`、來源與目標世代、原因、方向（forward / revert）。交接確認綁定於它。 |
| HK | Header MAC Key，由 VK 經 HKDF 推導，只用來對 Vault 標頭計算 HMAC。 |
| Lock / Unlock | 本機尚未載入 VK 的狀態為 locked；輸入主密碼解開 VK 後為 unlocked。 |
| Drive Folder（Drive 資料夾） | Google Drive 上存放此 Vault 所有檔案的資料夾。名稱由使用者自訂，預設 `HolienSecretsManager`。 |
| Naming Scheme（命名規則） | 此 Vault 在 Drive 上各類檔案的名稱設定，存於 Vault 標頭，所有裝置共用。 |
| Device（裝置） | 執行 HSM 的一台電腦。每台裝置有獨立的 device_id，用於同步與衝突判斷。 |
| Chrome 設定檔（Profile） | Chrome 存放單一使用者資料的目錄，內含 `Login Data` SQLite 資料庫，儲存網址、帳號與以作業系統金鑰加密的密碼。 |

---

## 3. 使用情境

| 編號 | 情境 |
|------|------|
| UC-01 | 使用者第一次在筆電上啟動應用程式，輸入主密碼、完成 Google 授權，接受預設的資料夾與檔案名稱（或改成自己想要的名稱），工具在 Drive 建立資料夾與空的 Vault。 |
| UC-01a | 使用者希望 Drive 上看起來不像密碼庫，將資料夾命名為 `Photos/2019`、標頭檔命名為 `notes.txt`、項目檔副檔名改為 `.jpg`；工具照常運作，因為辨識靠檔案屬性而非檔名。 |
| UC-02 | 使用者為新伺服器產生一把 ed25519 SSH 金鑰，取得公鑰貼到伺服器 `authorized_keys`，並記錄此金鑰綁定於該主機的 `deploy` 帳號。 |
| UC-03 | 使用者換到桌機，執行 `hsm init --existing`，完成 Google 授權並輸入主密碼，Vault 自動從 Drive 同步下來。 |
| UC-04 | 使用者在桌機上將某把私鑰安裝到 `~/.ssh/`，或直接載入 ssh-agent，不經過磁碟。 |
| UC-05 | 使用者將舊的 RSA 金鑰輪替為新金鑰，新舊版本並存，更新完所有主機後將舊版撤銷。 |
| UC-06 | 使用者在沒有網路的環境查詢某個網站的密碼，工具使用本機快取回應。 |
| UC-07 | 使用者在兩台裝置離線時各自修改同一個 Item，上線後工具偵測衝突，保留兩個版本並提示處理。 |
| UC-08 | 使用者遺忘主密碼，以當初保存的復原金鑰解鎖並設定新主密碼。 |
| UC-09 | 使用者匯出整個 Vault 為加密備份檔，離線保存於 USB 隨身碟。 |
| UC-10 | 使用者在 Chrome 的密碼管理器匯出 CSV，拖進應用程式視窗，預覽後匯入數百筆網址與帳密，重複項目自動合併。 |
| UC-11 | 使用者在應用程式中點「從 Chrome 匯入」，應用程式自動找到本機的 Chrome 設定檔，直接讀取並解密已儲存的帳密，不需手動匯出 CSV。 |

---

## 4. 功能需求

需求編號格式：`FR-<模組>-<序號>`。優先級：**P0** 必要（MVP）、**P1** 重要、**P2** 可延後。

### 4.1 初始化與解鎖（INIT）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-INIT-01 | P0 | `init` 流程：設定主密碼（需輸入兩次，並顯示強度評估）、完成 Google OAuth 授權、決定 Drive 資料夾、建立 Vault。 |
| FR-INIT-02 | P0 | 主密碼最短 12 字元；工具以 zxcvbn 類演算法估算強度，低於門檻時警告但不強制拒絕。 |
| FR-INIT-03 | P0 | 支援在新裝置以 `init --existing` 連接既有 Vault：授權 Google、找到 Drive 資料夾、下載 Vault 標頭、驗證主密碼、同步。 |
| FR-INIT-04 | P0 | 建立 Vault 時產生復原金鑰（256-bit，以 Base32 分組顯示，共 52 字元），只顯示一次，並要求使用者確認已保存。使用者可選擇略過，略過時明確警告遺忘主密碼即無法復原。 |
| FR-INIT-05 | P0 | 解鎖：執行需要讀寫機密的指令時提示輸入主密碼；輸入錯誤回傳明確錯誤且不洩漏任何資料。 |
| FR-INIT-06 | P1 | 解鎖狀態維持：啟動本機 agent 程序在記憶體中保存 VK，逾時（預設 15 分鐘，可設定）或執行 `lock` 後清除，期間指令不需重複輸入主密碼。agent 的安全要求見 5.6。 |
| FR-INIT-07 | P0 | 變更主密碼：只重新包裝 VK 並寫入一次標頭，不碰任何資料檔；復原金鑰不受影響。流程見 5.1.5。 |
| FR-INIT-08 | P0 | 以復原金鑰解開 VK 並設定新主密碼；流程同 FR-INIT-07，只是 VK 來源不同。 |
| FR-INIT-10 | P0 | 全面輪替金鑰（懷疑 VK、主密碼或復原金鑰外洩時）：產生新世代 VK，所有檔案以新 DEK 重新加密。輪替前必須重新取得憑證並依外洩原因替換；採狀態機與明確提交點，可中斷續行，舊金鑰保留到所有裝置確認後才清除。流程見 5.1.6。 |
| FR-INIT-12 | P0 | 解開 VK 之後、採用標頭任何其他欄位（修訂鏈、世代狀態、命名規則、裝置清單）之前，必須先驗證整份標頭的 MAC（8.1）。MAC 驗證前只允許使用：格式版本、KDF 參數（經上限檢查）、VK 包裝、`vk_check`。 |
| FR-INIT-11 | P0 | 標頭中的 KDF 參數在執行前須通過硬性上限檢查（記憶體 ≤ 1 GiB、迭代 ≤ 16、並行 ≤ 16），超過即拒絕並回報 `KDF_PARAMS_UNSAFE`，不執行推導。 |
| FR-INIT-09 | P2 | 支援多個 Vault（例如 `--vault work`），各自對應不同 Drive 資料夾，可使用不同 Google 帳號。 |

### 4.2 Google Drive 儲存（DRIVE）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-DRIVE-01 | P0 | 以 OAuth 2.0（桌面應用程式流程、PKCE）取得 Google Drive 授權；權限範圍最小化，預設使用 `drive.file`（只能存取本工具建立的檔案）。詳見 7.1。 |
| FR-DRIVE-02 | P0 | 資料夾路徑由使用者自訂，預設為 My Drive 根目錄下的 `HolienSecretsManager`。初始化時以預設值預先填入，使用者可直接接受或修改。 |
| FR-DRIVE-03 | P0 | 資料夾路徑以 `/` 分隔，例如 `Backups/Secrets`；逐層檢查，不存在的層級自動建立。 |
| FR-DRIVE-04 | P0 | 指定路徑若已存在由本工具建立、包含有效 Vault 標頭的資料夾，則連接該 Vault；若資料夾存在但不含 Vault 標頭，詢問是否在其中建立新 Vault。 |
| FR-DRIVE-05 | P0 | 所有上傳到 Drive 的檔案在上傳前於本機加密（格式見第 8 章）；Item 檔名中代表該項目的部分為隨機識別碼，不含 Item 名稱或型別。 |
| FR-DRIVE-12 | P0 | Drive 上每一類檔案的名稱皆由使用者自訂，系統提供預設值（見 7.3）。可自訂的項目：Vault 標頭檔名、索引檔名、操作紀錄檔名、主機檔名、Item 子資料夾名稱、Item 檔名樣式（含 `{id}` 佔位符與副檔名）。 |
| FR-DRIVE-13 | P0 | 命名規則存於 Vault 標頭，其他裝置連接 Vault 時自動採用，不需重新輸入。 |
| FR-DRIVE-14 | P0 | 檔案辨識以 Drive 檔案的 `appProperties`（`hsm.kind`、`hsm.vault_id`、`hsm.file_id`）為準，檔名只是顯示用途。使用者在 Drive 網頁手動改名不會破壞 Vault；工具偵測到檔名與命名規則不符時只在狀態畫面提示，不自動改回。 |
| FR-DRIVE-15 | P0 | 檔名驗證：不得為空、不得含 `/`、長度不超過 200 字元、同一 Vault 內各類檔名不得相同、Item 檔名樣式必須含 `{id}`。不限制副檔名，允許使用者刻意用 `.txt`、`.jpg` 等名稱。 |
| FR-DRIVE-16 | P1 | 事後修改命名規則：於設定畫面修改後，工具更新 Vault 標頭並逐一以 `files.update` 重新命名 Drive 上的檔案；中途失敗可重試，重試時以 `appProperties` 找到檔案，不受已改名或未改名影響。 |
| FR-DRIVE-17 | P1 | 事後搬移資料夾：可指定新的資料夾路徑，工具以 `files.update` 變更資料夾的 parent，不重新上傳內容。 |
| FR-DRIVE-06 | P0 | OAuth refresh token 存於作業系統金鑰圈（macOS Keychain / Linux Secret Service / Windows Credential Manager）；無金鑰圈時以 VK 加密後存於本機設定目錄。 |
| FR-DRIVE-07 | P0 | 處理 Drive API 的速率限制與暫時性錯誤：指數退避重試（最多 5 次），超過後回報明確錯誤並保留本機變更待下次同步。 |
| FR-DRIVE-08 | P0 | 撤銷授權：`auth revoke` 向 Google 撤銷 token 並清除本機 token；不刪除 Drive 上的檔案。 |
| FR-DRIVE-09 | P1 | 顯示 Drive 資料夾狀態：Item 數、總大小、最後同步時間、Drive 剩餘空間。 |
| FR-DRIVE-10 | P0 | 不依賴 Google Drive 的檔案修訂歷史做資料保護（Google 不保證保留未設 `keepForever` 的舊修訂）。資料檔一律不就地覆寫，每次寫入建立新檔案，見 4.5 與 7.4。 |
| FR-DRIVE-11 | P2 | 儲存層抽象：定義 `Storage` 介面（list / get / put / delete / rename / revision），以便日後支援本機資料夾、S3 等後端。 |

### 4.3 帳密管理（CRED）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-CRED-01 | P0 | 建立帳密型 Item，欄位：名稱、帳號、密碼、網址、備註、標籤、TOTP 秘密（選填）。全部欄位皆為加密內容。 |
| FR-CRED-02 | P0 | 修改任一欄位產生新版本，舊版本保留，預設保留最近 10 個版本。 |
| FR-CRED-03 | P0 | 讀取密碼與讀取其他欄位（帳號、網址）為不同指令，避免列表時意外顯示密碼。 |
| FR-CRED-04 | P0 | 支援通用型 Item（任意 key/value 或單一文字區塊），用於連線字串、設定檔、備援碼等。 |
| FR-CRED-05 | P0 | 內建密碼產生器，可設定長度（預設 24）與字元集，可產生 passphrase 式密碼。 |
| FR-CRED-06 | P1 | 若 Item 含 TOTP 秘密，可直接產生目前的 6 位驗證碼。 |
| FR-CRED-07 | P1 | 可設定到期日，`status` 指令列出到期前 N 天（預設 14 天）的項目。 |
| FR-CRED-08 | P1 | 以名稱、標籤、網址模糊搜尋。 |
| FR-CRED-09 | P2 | 附件型 Item（任意檔案），單一附件上限預設 5 MB。 |

### 4.4 伺服器金鑰管理（KEY）

本模組為專案核心，涵蓋伺服器私鑰與公鑰的完整生命週期。

#### 4.4.1 產生與匯入

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-01 | P0 | 在本機產生金鑰對。支援演算法：`ed25519`（預設）、`rsa-4096`、`rsa-2048`、`ecdsa-p256`。 |
| FR-KEY-02 | P0 | 匯入既有私鑰（OpenSSH 格式、PEM / PKCS#1 / PKCS#8，含有 passphrase 的私鑰）。匯入時自動推導公鑰與指紋，並驗證與使用者提供的公鑰（若有）相符。 |
| FR-KEY-03 | P0 | 從 `~/.ssh` 掃描並批次匯入現有金鑰（`key import --scan`），列出候選後由使用者勾選。 |
| FR-KEY-04 | P0 | 匯入 TLS 憑證與對應私鑰；自動解析 Subject、SAN、到期日，驗證私鑰與憑證匹配。 |
| FR-KEY-05 | P0 | 金鑰對可設定用途（`purpose`）：`ssh`、`tls`、`signing`、`other`。 |
| FR-KEY-06 | P1 | 產生金鑰時可選擇為私鑰加上 passphrase；passphrase 另存為加密欄位，匯出時可選擇保留或移除。 |
| FR-KEY-07 | P1 | 支援僅儲存公鑰（無私鑰）的紀錄，用於登錄外部系統或他人提供的公鑰。 |

#### 4.4.2 取用與匯出

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-10 | P0 | 公鑰與指紋（SHA256 與 MD5 兩種格式）可直接顯示，不需額外確認。 |
| FR-KEY-11 | P0 | 私鑰匯出預設寫入檔案，以「建立時即為僅本人可讀」的方式建立（Unix `0600`、Windows 僅本人 ACL，見 5.3），絕不先寫明文再收緊權限；要輸出到 stdout 需明確加 `--stdout`，且偵測到 stdout 為終端時再次確認。預設只匯出 `active` 版本。 |
| FR-KEY-12 | P0 | 私鑰可以 OpenSSH、PEM（PKCS#1 / PKCS#8）格式匯出；格式轉換於記憶體內完成。 |
| FR-KEY-13 | P0 | 公鑰可以 OpenSSH `authorized_keys` 單行格式、PEM 格式輸出。 |
| FR-KEY-14 | P1 | `key install`：將私鑰與公鑰寫入 `~/.ssh/<name>` 與 `~/.ssh/<name>.pub`，並可選擇在 `~/.ssh/config` 追加對應 Host 區塊。第一版以匯出到使用者指定路徑（FR-KEY-11）為主。 |
| FR-KEY-15 | P1 | `key ssh-add`：直接將私鑰載入執行中的 ssh-agent，不寫入磁碟；可設定存活時間。 |
| FR-KEY-16 | P1 | 以指紋反查金鑰（用於從伺服器 `authorized_keys` 盤點）。 |
| FR-KEY-17 | P2 | 支援 PuTTY `.ppk` 格式匯出。 |

#### 4.4.3 生命週期

狀態是**每個金鑰版本各自一個**，不是整個 Item 一個。Item 層級只有衍生的摘要（有任一 `active` 版本即視為 active）。

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-20 | P0 | 每個金鑰版本有獨立狀態：`active`、`retired`（已被新版本取代，仍可讀取）、`revoked`（已撤銷，預設不可匯出）。狀態只能單向前進：active → retired → revoked，或 active → revoked。 |
| FR-KEY-21 | P0 | 輪替（rotate）：以相同演算法與設定產生新版本並設為 `active`，原 active 版本自動轉為 `retired`。`key show` 列出所有版本、各自狀態與指紋。 |
| FR-KEY-22 | P0 | 撤銷（revoke）：可指定版本或全部版本，需填寫原因。撤銷後該版本私鑰預設不可匯出、不可安裝、不出現在 `authorized-keys` 輸出；需加 `--force` 並再次確認才可匯出。 |
| FR-KEY-26 | P0 | 匯出、安裝、載入 agent 預設只針對 `active` 版本；要操作 `retired` 版本必須明確指定 `--version N` 並顯示警告。 |
| FR-KEY-27 | P0 | **撤銷是本機標記，不代表伺服器已移除授權。** 介面與文件必須明確區分「本工具已標記撤銷」與「伺服器端已移除」；後者由主機綁定的部署狀態記錄（FR-KEY-37）。 |
| FR-KEY-23 | P1 | 可設定金鑰到期日；TLS 憑證自動以憑證到期日為準。`status` 指令列出 30 / 14 / 7 天內到期項目。 |
| FR-KEY-24 | P1 | 可設定輪替政策（例如每 90 天），`status` 指令列出逾期未輪替的金鑰；不自動執行輪替。 |
| FR-KEY-25 | P1 | 刪除金鑰採軟刪除，進入垃圾桶，保留期（預設 30 天）後由 `gc` 指令永久清除。 |

#### 4.4.4 主機綁定

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-30 | P0 | 建立 Host 紀錄：名稱、位址（IP 或 FQDN）、SSH 連接埠、環境（`prod`/`staging`/`dev`/`other`）、備註、標籤。 |
| FR-KEY-31 | P0 | 綁定以「Host × 金鑰版本 × 伺服器端帳號」為單位。一把金鑰的不同版本可分別綁定；綁定時記錄伺服器端帳號（例如 `root`、`deploy`）與用途。 |
| FR-KEY-32 | P0 | 可列出某台 Host 所有綁定，及某把金鑰（含各版本）綁定的所有 Host。 |
| FR-KEY-33 | P0 | `host authorized-keys`：只輸出該 Host 綁定中、版本為 `active` 的公鑰；若存在 `retired` 或 `revoked` 版本仍標記為已安裝，輸出時以註解與警告列出，提醒使用者到伺服器移除。 |
| FR-KEY-37 | P0 | 每個綁定有部署狀態：`declared`（已登記，尚未確認安裝）、`installed`（使用者確認已放到伺服器）、`removal_pending`（該版本已 retired 或 revoked，但伺服器尚未移除）、`removed`（使用者確認已從伺服器移除）。狀態由使用者手動更新；輪替或撤銷時工具自動把受影響的 `installed` 綁定改為 `removal_pending`。 |
| FR-KEY-38 | P0 | `status` 列出所有 `removal_pending` 綁定，顯示主機、帳號、金鑰版本與指紋，作為「伺服器端待處理清單」。 |
| FR-KEY-34 | P1 | 可記錄 Host 的 SSH host key 指紋；`key install` 寫入 `~/.ssh/config` 時一併產生 `known_hosts` 行。 |
| FR-KEY-35 | P2 | 透過 SSH 連線到 Host 讀取 `authorized_keys`，與紀錄比對並回報差異（唯讀），並可據此把 `removal_pending` 自動更新為 `removed`。 |
| FR-KEY-36 | P2 | 透過 SSH 將公鑰推送到 Host 的 `authorized_keys`（需二次確認）。 |

### 4.5 同步與離線（SYNC）

設計原則：**先保存，再判斷**。每次寫入都先成為 Drive 上一個獨立、不會被覆寫的修訂（rev），衝突與分支是在修訂鏈上事後偵測，不靠「先查再寫」的時間窗。

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-SYNC-01 | P0 | Google Drive 為資料的唯一真實來源（source of truth）；本機保有完整的加密快取，內容與 Drive 上的密文相同。 |
| FR-SYNC-02 | P0 | 寫入操作：先寫本機快取並標記 `pending`，再上傳；上傳成功後清除標記。任何時刻中斷都不會造成已確認寫入的資料遺失。 |
| FR-SYNC-03 | P0 | 讀取操作：預設先同步索引（輕量），再讀取本機快取；`--offline` 時完全不連網。 |
| FR-SYNC-04 | P0 | 每個 Item 為獨立的邏輯檔案（`file_id`），衝突範圍限縮在單一 Item。 |
| FR-SYNC-05 | P0 | **不可覆寫的雜湊鏈修訂**：每次寫入 index、hosts、log、Item 或標頭，都在 Drive 建立一個新檔案（新的 Drive 檔案 ID），其受驗證的標頭含 `rev`（隨機 UUID）、`parent_rev` 與 `parent_hash`（父修訂整個檔案的 SHA-256）。工具從不呼叫 `files.update` 修改資料檔內容。 |
| FR-SYNC-06 | P0 | 分支偵測：同步時以 appProperties **只列出候選**，所有尚未驗證的候選一律下載驗證；head 只在**已驗證修訂**的集合上計算：一個已驗證修訂失去 head 身分，唯一的方式是存在一個已驗證的子修訂以它為父。未驗證或驗證失敗的候選不能取消任何修訂的 head 身分（7.4）。 |
| FR-SYNC-17 | P0 | 驗證規則：head 以內容驗證（解密）；祖先以關係驗證（雜湊）：沿 `parent_hash` 與 trailer 中的合併父雜湊走到本機信任點，途中每個節點都要下載並比對其完整 bytes 的雜湊，**不需要解密、不需要舊世代 VK**；信任點的實際檔案雜湊必須等於本機記錄；同一 `rev` 對應到不同內容即 `HASH_MISMATCH`；引用不存在的父修訂即 `MISSING_PARENT`，與 head 數量無關、獨立檢查。 |
| FR-SYNC-07 | P0 | 分支處理依檔案類型：索引由全部 Item 重建後自動合併；操作紀錄取聯集自動合併；Item 與 hosts 的分支若解密後內容完全相同則自動合併，否則標記 `conflict` 交由使用者處理，**不以時間戳自動決定**。 |
| FR-SYNC-08 | P0 | 涉及撤銷、刪除、金鑰輪替、狀態變更的分支一律由使用者處理；解決時產生一個新修訂，trailer 列出所有被合併的 head（rev 與雜湊），使它們都成為祖先。 |
| FR-SYNC-09 | P0 | 衝突介面逐欄位並排顯示兩個 head 的差異（機密值遮罩，可展開），使用者可選擇其一或逐欄位挑選。未解決前該 Item 可檢視（同時顯示所有 head 並加警示），但不可寫入。 |
| FR-SYNC-18 | P0 | 衝突未解決時，**敏感操作一律拒絕**：匯出私鑰、安裝、載入 agent、輸出 `authorized-keys`、複製密碼。若使用者要在解決前操作，必須明確選定某一個分支（介面顯示該分支的狀態，例如「此分支已撤銷」），且操作紀錄註明所選分支。不得因「較新」而自動選擇 active 分支。 |
| FR-SYNC-10 | P0 | 回滾防護：每台裝置在 `sync_state.json` 記錄每個 `file_id`（含標頭）的信任點（rev、整個檔案的 SHA-256、金鑰世代）。同步時若**沒有任何** head 能沿已驗證雜湊鏈走到信任點，或信任點已不存在於遠端，視為疑似回滾，回報 `ROLLBACK_SUSPECTED` 並停止寫入，需使用者確認後才接受新鏈並重設信任點。與信任點有共同祖先的其他 head 是合法的並行分支，走衝突處理，不是回滾（7.4）。 |
| FR-SYNC-20 | P0 | 遠端存在性：head 與回滾判定只使用「本次 list 確認存在於 Drive」的修訂；本機累積的驗證登錄只用於免重複下載與離線復原。遠端缺少本機已知的修訂時回報 `REMOTE_DELETED` 或 `ROLLBACK_SUSPECTED`，並提供從本機快取重新上傳。 |
| FR-SYNC-11 | P0 | 回滾防護的已知限制：全新裝置第一次同步沒有獨立的信任紀錄，無法辨識整份資料是否已被換回舊版本。文件須寫明；緩解方式為 GUI 顯示索引摘要雜湊與最後修訂時間，使用者可與另一台裝置或備份檔人工比對。 |
| FR-SYNC-12 | P0 | **第一版不清理任何修訂。** 祖先驗證需要完整雜湊鏈，而各裝置的信任點只存在本機，無法得知離線裝置仍信任哪一個修訂；刪除任何中間修訂都可能使其無法重新連接。`gc` 在第一版只處理本機快取與垃圾桶中 Item 的內容清除，不刪除 Drive 上的修訂。 |
| FR-SYNC-19 | P1 | 歷史清理（Phase 2 設計）：以「經驗證的檢查點」區分「刪除歷史內容」與「刪除歷史關係」。檢查點是一個受驗證的修訂，記錄所有 file_id 當時的 head rev 與雜湊；裝置若信任某個檢查點（或其後代），即可接受檢查點之前的修訂被刪除。裝置的信任點早於最舊檢查點時，退回 FR-SYNC-10 的人工確認。 |
| FR-SYNC-13 | P0 | 索引檔可由全部 Item 的 head 重建（`sync --rebuild-index`），索引損毀或分支不會造成資料遺失。 |
| FR-SYNC-14 | P0 | 每次寫入前讀取標頭的 head，確認本機所持 active 金鑰（key_id）仍是標頭指定的 active；若標頭已換 active 或輪替中，先更新再寫，否則拒絕寫入（`REKEY_IN_PROGRESS` / `STALE_GENERATION`）。 |
| FR-SYNC-21 | P0 | 跨世代父子規則（5.1.6）：子修訂金鑰世代低於父修訂者一律拒絕並隔離；高於父修訂者只允許作為輪替快照內 head 的遷移修訂；外洩輪替期間快照之外出現的舊世代修訂只隔離、不自動遷移、不進入衝突處理，由使用者檢視後重新提交或丟棄。 |
| FR-SYNC-22 | P0 | 並行輪替：兩台裝置同時啟動輪替形成標頭分支時，合併標頭保留雙方全部金鑰與修訂，被取代的 op 之金鑰轉為 retired 並列入允許遷移；合併前不清除任何金鑰（5.1.6）。 |
| FR-SYNC-15 | P1 | 顯示同步狀態：pending 上傳數、最後同步時間、衝突數、各檔案修訂數。 |
| FR-SYNC-16 | P2 | 背景同步：agent 程序定期（預設 5 分鐘）同步索引並上傳 pending 項目。 |

### 4.6 備份與匯入匯出（BACKUP）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-BAK-01 | P0 | `export --encrypted`：將整個 Vault 打包為單一**可獨立解鎖**的加密檔案（格式見 8.7）。外層明文保存格式版本、KDF 參數、salt 與 VK 的兩份包裝（主密碼、復原金鑰），內層為全部資料；外層資訊納入完整性保護。 |
| FR-BAK-05 | P0 | 備份驗收條件：在**全新電腦、沒有原本快取、無法登入原 Google 帳號**的情況下，僅憑備份檔加主密碼（或復原金鑰）即可還原為可用的 Vault，可還原到新的 Drive 資料夾或本機資料夾。此條件為自動化測試。 |
| FR-BAK-06 | P0 | 備份檔內容為所有檔案的 head 修訂與各 Item 保留的版本；`import` 還原時以新的 rev 鏈重新寫入，不保留原修訂歷史。 |
| FR-BAK-07 | P1 | 匯出時可選擇是否包含操作紀錄與垃圾桶中的項目。 |
| FR-BAK-02 | P0 | `export --plain`：匯出為明文 JSON，需二次確認並顯示警告；輸出檔以僅本人可讀方式建立（5.3）。 |
| FR-BAK-03 | P1 | 從其他密碼管理器的匯出檔匯入（Bitwarden JSON / CSV、1Password CSV、KeePass XML、Firefox 與 Edge CSV）。 |
| FR-BAK-04 | P1 | `verify`：下載所有 Item 並驗證解密與完整性，回報損毀項目。 |

#### 4.6.1 從 Chrome 匯入（CHROME）

兩條途徑：**CSV 匯出檔**（可靠、跨平台，P0）與**直接讀取 Chrome 設定檔**（免手動匯出，但受作業系統與 Chrome 版本限制，P1）。

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-CHROME-01 | P0 | 匯入 Chrome 密碼管理器匯出的 CSV（欄位 `name,url,username,password,note`）。欄位順序不固定時以標題列對應；缺少 `url` 或 `password` 的列跳過並列入報告。 |
| FR-CHROME-02 | P0 | 匯入前顯示預覽：總筆數、將新增、將合併、將跳過的筆數，以及每一筆的網址與帳號（密碼遮罩）。使用者可逐筆取消勾選。 |
| FR-CHROME-03 | P0 | 欄位對應：`name` → Item 名稱（空白時以網址主機名稱 + 帳號組成）、`url` → 網址、`username` → 帳號、`password` → 密碼、`note` → 備註；所有匯入項目自動加上 `chrome-import` 標籤與匯入日期標籤。 |
| FR-CHROME-04 | P0 | 重複判斷：同一正規化網址（去 scheme、去尾端 `/`、小寫主機名稱）且同一帳號視為同一項目。密碼相同 → 跳過；密碼不同 → 預設為既有 Item 新增一個版本（可改為跳過或另建項目）。 |
| FR-CHROME-05 | P0 | 匯入完成後顯示報告並寫入操作紀錄；報告不含密碼。提示使用者刪除 CSV 檔，並提供「安全刪除」按鈕（覆寫後刪除，不保證在 SSD 上有效，文件須說明）。 |
| FR-CHROME-06 | P1 | 直接讀取 Chrome 設定檔：自動偵測各平台預設路徑（下表），列出所有 Profile 供選擇，複製 `Login Data` 到暫存目錄後讀取，避免與執行中的 Chrome 衝突。 |
| FR-CHROME-07 | P1 | 解密各平台的 Chrome 密碼加密：macOS 透過 Keychain 讀取「Chrome Safe Storage」金鑰（系統會跳出授權提示）；Linux 透過 Secret Service / KWallet 取得金鑰，退回 `peanuts` 預設金鑰；Windows 讀取 `Local State` 中以 DPAPI 保護的 AES 金鑰。無法取得金鑰時給出明確錯誤並引導改用 CSV 途徑。 |
| FR-CHROME-08 | P1 | Windows 上 Chrome 127 以後的 App-Bound Encryption（`v20` 前綴）無法由第三方程式解密；偵測到時明確告知並引導改用 CSV 途徑，不嘗試繞過。 |
| FR-CHROME-09 | P1 | 直接讀取途徑的預覽、欄位對應、重複判斷與報告規則同 CSV 途徑。 |
| FR-CHROME-10 | P2 | 支援同為 Chromium 核心的 Edge、Brave、Chromium 設定檔路徑。 |

**Chrome 設定檔預設路徑**

| 平台 | 路徑 |
|------|------|
| macOS | `~/Library/Application Support/Google/Chrome/<Profile>/Login Data` |
| Linux | `~/.config/google-chrome/<Profile>/Login Data` |
| Windows | `%LOCALAPPDATA%\Google\Chrome\User Data\<Profile>\Login Data` |

`<Profile>` 為 `Default`、`Profile 1`、`Profile 2` 等；顯示名稱從同目錄的 `Preferences` 檔讀取。

**Chrome CSV 匯出步驟（寫入使用文件）**：Chrome 選單 → 密碼與自動填入 → Google 密碼管理工具 → 設定 → 匯出密碼。

### 4.7 本機操作紀錄（LOG）

個人工具不需要稽核，但保留操作紀錄便於回溯「我什麼時候改了什麼」。

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-LOG-01 | P0 | 本機記錄每次 Item 建立、修改、刪除、私鑰匯出、金鑰輪替、撤銷、同步衝突，含時間、裝置、操作、目標、結果。 |
| FR-LOG-02 | P0 | 操作紀錄本身以與其他檔案相同的方式加密存於本機並同步到 Drive，使各裝置可看到彼此的紀錄；分支時取聯集合併。 |
| FR-LOG-03 | P0 | 紀錄中絕不包含機密值。 |
| FR-LOG-04 | P1 | `log` 指令可依時間、操作類型、Item 篩選。 |

### 4.8 介面（UI）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-UI-01 | P0 | 桌面圖形介面覆蓋所有 P0 功能（畫面規格見 9.1）。 |
| FR-UI-02 | P0 | 圖形介面：機密值預設遮罩，點擊「顯示」才解密顯示；複製密碼或私鑰到剪貼簿後預設 30 秒清除。 |
| FR-UI-03 | P0 | 圖形介面：視窗失焦超過設定時間（預設 5 分鐘）或系統進入休眠時自動鎖定，回到主密碼畫面。 |
| FR-UI-04 | P0 | 圖形介面：支援拖放檔案（CSV、私鑰、憑證）到視窗直接進入對應匯入流程。 |
| FR-UI-05 | P1 | CLI 覆蓋金鑰相關與同步相關的 P0 功能（規格見 9.2），供終端機工作流程與腳本使用；與圖形介面共用同一核心與同一本機快取。 |
| FR-UI-06 | P1 | CLI 互動式提示在非 TTY 環境自動改為錯誤回報，不會卡住。 |
| FR-UI-07 | P1 | 系統匣常駐：快速搜尋、複製、鎖定；關閉視窗時縮到系統匣。 |
| FR-UI-08 | P2 | 全域快捷鍵呼叫快速搜尋視窗。 |

---

## 5. 安全設計

### 5.1 加密架構

```
  主密碼（使用者記憶）                      復原金鑰（隨機 256-bit，預設啟用）
         │                                         │
         ▼ Argon2id（salt 存於標頭）                 ▼ HKDF-SHA256
        PK（Password Key）                        RK_key
         │                                         │
         │ wrap（AES-256-GCM）                      │ wrap（AES-256-GCM）
         └────────────────┐       ┌────────────────┘
                          ▼       ▼
                 VK（Vault Key，隨機 256-bit，平常不變；有世代號）
                          │
                          │ wrap / unwrap（AES-256-GCM）
          ┌───────────────┼───────────────┬───────────────┐
          ▼               ▼               ▼               ▼
      DEK(index)      DEK(item A)      DEK(item B)      DEK(log)     ← 每個檔案修訂一把，隨機 256-bit
          │               │               │               │
          ▼ AES-256-GCM   ▼               ▼               ▼
      index 修訂       item A 修訂      item B 修訂       log 修訂     ← 上傳到 Google Drive 的檔案
```

三層結構的理由：

- **主密碼與復原金鑰是平行的**，各自包裝同一把 VK。變更主密碼只重做 PK 那一份包裝，復原金鑰完全不受影響；反之亦然。
- **VK 平常不變**，所以改密碼不需要碰任何資料檔，是一次標頭寫入就完成的原子操作。
- **只有「全面輪替」會換 VK**（產生新世代），那是懷疑金鑰外洩時的獨立操作，代價是全部檔案重新加密，設計上接受中斷並可續行。

共通規則：

- 每個檔案修訂一把隨機 DEK（256-bit CSPRNG），以 VK 用 AES-256-GCM 包裝後存於該檔案標頭。
- 內容以 AES-256-GCM 加密，96-bit 隨機 nonce，每次寫入重新產生。
- AAD 為**完整的檔案標頭**（8.2 定義的 227 bytes），包含格式版本、檔案種類、vault_id、file_id、rev、parent_rev、parent_hash、key_id、長度欄位、合併父數量與 trailer 雜湊；備份檔另串接外層雜湊（8.7）。改動標頭任一 byte 都會使驗證失敗。
- 標頭的 `vk_check` 區段存一段以 VK 加密的固定內容，解鎖時先解此段判斷主密碼或復原金鑰是否正確，不需下載任何資料。
- Vault 標頭是明文 JSON，由 VK 衍生的 HK 計算 HMAC-SHA256 保護整份內容（8.1）。沒有 VK 就無法偽造或修改標頭的修訂鏈、世代狀態、命名規則與裝置清單。

#### 5.1.1 金鑰推導與解鎖

```
輸入：master_password（UTF-8，先做 NFKC 正規化）、header（見 8.1）

0. 參數檢查（執行推導之前）：
     header.wrappings.password.kdf 的 memory_kib ≤ 1048576、iterations ≤ 16、parallelism ≤ 16、
     salt 長度 == 32；任一不符 → KDF_PARAMS_UNSAFE，不執行
1. PK = Argon2id(password=NFKC(master_password), salt, memory, iterations, parallel, out_len=32)
2. 對標頭 keys[] 中**每一把未清除的金鑰** k：
     VK_k = AES-256-GCM-Decrypt(key=PK, nonce=k.wrappings.password.nonce,
                               aad="hsm-vk-wrap-v1" || vault_id || k.key_id || k.gen,
                               ct=k.wrappings.password.wrapped_vk)
     任一失敗 → BAD_PASSWORD；PK 與已解出的 VK 全部 zeroize
     （所有未清除金鑰都以同一個主密碼、各自的 nonce 包裝；改密碼時一併重新包裝，見 5.1.5）
3. 對每一把 k 驗證：AES-256-GCM-Decrypt(key=VK_k, nonce=k.vk_check.nonce, aad=vault_id || k.key_id || k.gen,
                                      ct=k.vk_check.ciphertext) == "hsm-vk-check-v1"
     失敗 → CORRUPTED（標頭損毀或被竄改）
   得到金鑰集合 {key_id → VK}；top 與 active 由狀態機決定（5.1.6）
4. 標頭 MAC（FR-INIT-12）：header.mac.key_id 必須等於此標頭的 top（最高世代的未清除金鑰；同世代多把時取 key_id 位元組序最大者），否則 HEADER_MAC_INVALID
     HK = HKDF-SHA256(ikm=VK_top, salt=vault_id, info="hsm-header-mac-v1", len=32)
     HMAC-SHA256(HK, canonical(header 去掉 mac 欄位)) == header.mac.value，否則 HEADER_MAC_INVALID
     通過後才可採用 rev / parent_rev / parent_hash / keys[].status / rekey_ops / naming / devices
5. 標頭信任點與轉換授權：sync_state 有標頭信任點 P 時，此標頭 H 必須能沿雜湊鏈驗證回到 P，
     且 P 到 H 的路徑上每一個相鄰轉換都在 5.1.6 的授權表內（5.1.6「路徑驗證」），否則 HEADER_TRANSITION_DENIED
6. PK zeroize；VK 保存在記憶體直到鎖定

以復原金鑰解鎖：
1. RK = Base32Decode(使用者輸入，忽略空白與大小寫)，長度須為 32 bytes
2. RK_key = HKDF-SHA256(ikm=RK, salt=wrappings.recovery.hkdf_salt, info="hsm-recovery-v1", len=32)
3. 對每一把未清除金鑰 k：VK_k = AES-256-GCM-Decrypt(key=RK_key, nonce=k.wrappings.recovery.nonce,
                            aad="hsm-vk-wrap-v1" || vault_id || k.key_id || k.gen,
                            ct=k.wrappings.recovery.wrapped_vk)
4. 驗證與後續同上；RK、RK_key zeroize
```

- NFKC 正規化避免同一密碼在不同作業系統輸入法下得到不同 bytes。
- 推導只在解鎖時做一次；之後 VK 保存在記憶體（GUI 程序內或 CLI agent），直到鎖定。
- 解鎖永遠解出整個金鑰集合（可能多把：active、輪替中的另一把、以及取消留下的 retired），不只兩把。

#### 5.1.2 加密一個檔案修訂（寫入 / 上傳）

適用於 index、log、hosts、每個 Item 與標頭以外的所有資料檔。以 Item 為例：

```
輸入：active 金鑰（key_id_a, VK_a, gen_a）、vault_id、file_id、parent_rev 與 parent_hash（本機信任的 head 的 rev 與
      整個檔案的 SHA-256；首次皆為全零）、kind、plaintext_json
前置：父修訂（若有）的金鑰世代 ≤ gen_a（5.1.6「跨世代父子規則」）；否則拒絕寫入

1. 序列化：plaintext = JSON(plaintext_json)
2. 壓縮：  compressed = zstd(plaintext)；len(compressed) > 64 MiB → 拒絕
3. 新修訂識別：rev = UUIDv4()
4. 產生 DEK：DEK = CSPRNG(32 bytes)
5. 包裝 DEK：dek_nonce = CSPRNG(12)
             wrapped_dek = AES-256-GCM-Encrypt(key=VK_a, nonce=dek_nonce,
                             aad="hsm-dek-wrap-v1" || vault_id || file_id || rev || key_id_a, pt=DEK)
6. 組 227-byte 標頭（8.2）：
             trailer = 合併父列表（每筆 rev 16 bytes + hash 32 bytes），一般寫入為空
             header = "HSM1" || format(1) || kind || vault_id || file_id || rev || parent_rev ||
                      parent_hash || key_id_a || dek_nonce || wrapped_dek || nonce || plaintext_len ||
                      merge_count || trailer_hash
             trailer_hash 一律 = SHA-256(trailer bytes)；trailer 為空時即 SHA-256("") =
               e3b0c442 98fc1c14 9afbf4c8 996fb924 27ae41e4 649b934c a495991b 7852b855（沒有全零特例）
             其中 nonce = CSPRNG(12)、plaintext_len = len(compressed)
7. 加密內容：ciphertext = AES-256-GCM-Encrypt(key=DEK, nonce=nonce, aad=header, pt=compressed)
8. file = header || ciphertext || trailer
9. 寫入本機快取（寫暫存檔 → fsync → rename），記錄 pending
10. 上傳到 Drive 為**新檔案**（files.create），appProperties 標示 kind / vault_id / file_id / rev /
    parent_rev / key_id（只供搜尋候選，不作為信任依據）；檔名依命名規則
11. 上傳成功 → 以 SHA-256(file) 更新 sync_state 的 head 與信任點；清除 pending
12. zeroize：DEK、plaintext、compressed
```

- 每次寫入都換新的 DEK、nonce 與 rev；同一 Item 的相鄰兩個修訂是完全不同的密文與不同的 Drive 檔案。
- 寫入前先執行 FR-SYNC-14 的世代檢查。

#### 5.1.3 解密一個檔案修訂（讀取 / 下載）

```
輸入：金鑰集合 {key_id → VK}、file bytes、預期的 vault_id 與 file_id

1. len(file) < 243 或 > 80 MiB → CORRUPTED
2. magic == "HSM1" 且 format == 1，否則 CORRUPTED / UNSUPPORTED_FORMAT
3. 解析 227-byte 標頭；header.vault_id == 預期、header.file_id == 預期，否則 CORRUPTED
4. plaintext_len ≤ 64 MiB 且 len(file) == 227 + plaintext_len + 16 + merge_count × 48，否則 CORRUPTED
   trailer = 最後 merge_count × 48 bytes；SHA-256(trailer) == trailer_hash，否則 CORRUPTED
   （此檢查在任何配置之前；長度欄位本身也在 AAD 內，第 6 步會再驗證）
5. 依 header.key_id 在金鑰集合中選擇 VK；集合中沒有（已清除或未知）→ STALE_GENERATION
   DEK = AES-256-GCM-Decrypt(key=VK, nonce=dek_nonce,
                              aad="hsm-dek-wrap-v1" || vault_id || file_id || rev || key_id, ct=wrapped_dek)
   失敗 → CORRUPTED
6. compressed = AES-256-GCM-Decrypt(key=DEK, nonce=nonce, aad=header（kind=5 時再串接外層雜湊，8.7）, ct=ciphertext)
   失敗 → CORRUPTED
7. plaintext = unzstd(compressed)，解壓輸出上限 64 MiB
8. 反序列化 JSON；JSON 內的 rev / parent_rev 必須與標頭一致
9. 此修訂現在是「內容已驗證」：其 rev、parent_rev、parent_hash、trailer 中的合併父、key_id（經標頭 keys[] 對應到世代）可用於建鏈（7.4）
10. zeroize：DEK、compressed、plaintext
```

- 任何一步失敗都不回傳部分內容。
- 解密完全在本機記憶體進行，明文不寫入磁碟（使用者明確要求的匯出除外）。

#### 5.1.4 各類檔案的加解密時機

| 檔案 | 解密時機 | 寫入新修訂的時機 |
|------|----------|------------------|
| 標頭 | 解鎖時讀取 KDF 參數、VK 包裝與 `vk_check`；本身大部分為明文 | 建立、變更主密碼、更新復原金鑰、修改命名規則、全面輪替的兩個提交點 |
| 索引 | 解鎖後第一次列表時，以及同步偵測到新 head 時 | 任何 Item 新增、改名、改標籤、狀態變更後整份重寫 |
| Item | 開啟該 Item 詳細頁或讀取其機密值時，只解密那一個檔案的 head | 該 Item 任何欄位變更後整份重寫（含所有保留版本） |
| hosts | 開啟主機頁面或綁定查詢時 | Host 或綁定變更後整份重寫 |
| log | 開啟紀錄頁面或同步合併時 | 每次操作後（可批次延後） |
| 備份檔 | `import` 時 | `export --encrypted` 時 |

#### 5.1.5 變更主密碼（含以復原金鑰重設）

對**金鑰集合中每一把未清除的 VK** 重做 PK 那一份包裝，一次標頭寫入完成，不碰任何資料檔。

```
1. 取得金鑰集合 {key_id → VK}：以舊主密碼（5.1.1）或復原金鑰解開（所有未清除金鑰一起解）
2. 新 salt = CSPRNG(32)；以新主密碼推導 PK_new（參數同標頭，或使用者選擇調高）
3. 對每一把 k：k.wrappings.password = { kdf（共用同一組參數與 salt）, nonce=CSPRNG(12),
      wrapped_vk = AES-256-GCM-Encrypt(key=PK_new, nonce, aad="hsm-vk-wrap-v1" || vault_id || k.key_id || k.gen, pt=VK_k) }
4. 組新標頭：只替換每把金鑰的 wrappings.password；wrappings.recovery、vk_check、keys[] 的狀態、naming 全部不變
5. 寫入新標頭修訂（parent_rev / parent_hash = 目前標頭 head；MAC 以 top 的 HK 計算），上傳
6. 成功 → 本機記錄新 head；失敗 → 什麼都沒變，舊主密碼仍有效
7. zeroize PK_new；所有 VK 不變、繼續使用
```

- 寫入規則：標頭中任一把未清除金鑰缺少 `wrappings.password`，或各金鑰的 password 包裝使用不同 salt，視為 CORRUPTED；改密碼必須一次涵蓋整個集合。
- 其他裝置下次解鎖時讀到新標頭 head，舊主密碼解包失敗，提示「主密碼已在其他裝置變更」。
- 若兩台裝置同時改密碼，標頭出現兩個 head（FR-SYNC-06）；工具拒絕一切寫入並依 5.1.6「標頭的合併」處理（兩份都能解開整個集合，不會遺失資料）。
- **此操作不會讓舊主密碼失去解開舊標頭修訂的能力**（舊標頭修訂第一版不刪除）。若懷疑舊主密碼已外洩，必須執行 5.1.6 的全面輪替，輪替後舊標頭修訂中包裝的舊金鑰只能解開輪替前已存在的內容。

#### 5.1.6 全面輪替金鑰（新世代）

懷疑 VK、主密碼或復原金鑰外洩時使用。產生新世代 VK，每個檔案以**新的 DEK 重新加密內容**（不只是重新包裝，因為舊 DEK 可能已隨舊 VK 外洩）。

**先說清楚它做不到什麼**：重新加密保險庫只能讓「之後」取得密文的人讀不到內容。已經被讀走的伺服器密碼、API Token、私鑰不會因此失效，必須到對應的服務或伺服器輪替。工具在啟動輪替時列出自使用者指定的「可疑時間」以來操作紀錄中所有被讀取、匯出的 Item，作為服務端輪替清單（FR-KEY-38 的待處理清單一併顯示）。

**憑證處理（輪替前）**

程式在解鎖後已清除 PK 與 RK_key，而且從不持有復原金鑰，因此輪替前必須重新取得憑證，並依外洩原因決定替換：

| 使用者選擇的原因 | 主密碼 | 復原金鑰 |
|------|------|------|
| 主密碼外洩 | **必須**輸入舊主密碼並設定新主密碼 | 輸入現有復原金鑰以保留，或產生新的 |
| 復原金鑰外洩 | 輸入現有主密碼（可選擇順便更換） | **必須**產生新的，不接受輸入舊的 |
| 不確定 / 兩者皆可能 / 記憶體被讀取 | **必須**設定新主密碼 | **必須**產生新的 |
| 只是定期輪替 | 輸入現有主密碼 | 輸入現有復原金鑰以保留，或產生新的 |

新世代 VK 只以**此時剛取得或剛建立**的 PK 與 RK_key 包裝；程式不得沿用任何在輪替開始前推導的金鑰。新復原金鑰必須在提交點 A 之前完成顯示與使用者確認保存。

**事故範圍的正確說法**：攻擊者若已下載密文並取得受影響世代的 VK 或憑證，可以離線解密，本工具不會有任何讀取紀錄。因此服務端輪替清單必須以「受影響世代能解開的所有內容」為範圍，也就是輪替當時 Vault 內的全部 Item（含垃圾桶與歷史版本）。操作紀錄只用來排序優先處理順序，不用來縮小範圍。

**金鑰識別、世代編號與操作 ID**：每把 VK 有全域唯一 `key_id`（UUID v4）；資料修訂標頭以 key_id 指明金鑰，**世代號不是識別**。標頭有 `max_gen`（歷來最大世代，只增不減）。每次輪替的目標世代一律為 `max_gen + 1`，取消後也不回收；每次輪替有唯一 `op_id`（UUID）、`from_key`、`to_key`（key_id）、`to_gen`、`reason`、`direction`、`snapshot`（見下）。`keys[]` 的條目永不移除，清除金鑰時只把 `status` 改為 `purged` 並刪除其包裝。並行輪替可能產生兩把世代號相同、key_id 不同的金鑰（見「並行輪替與標頭的合併」），這是允許的，解密時一律以 key_id 選鑰。

**提交點 A 的快照**：啟動輪替的裝置在寫入 migrating 標頭之前，先完成一次完整同步，然後把當下**每個邏輯檔案的 head** 記入 `rekey_ops[].snapshot`：`{ index: {rev, hash}, hosts: {rev, hash}, log: {rev, hash} }`。索引內容（8.4）本身列出每個 Item 的 `head_rev` 與 `head_hash`，因此索引的 {rev, hash} 即間接認證了所有 Item 當時的 head。快照受標頭 MAC 保護。

**跨世代父子規則**（適用於所有資料修訂，與輪替原因無關）：

1. 子修訂的金鑰世代 **低於** 父修訂（含合併父）的金鑰世代 → `GENERATION_REGRESSION`，該修訂永遠不能成為 head、不能被遷移、不能進入衝突處理；列入隔離區供使用者檢視。這直接擋下「以舊金鑰製作修訂、parent 指向新世代 head」的攻擊。
2. 子修訂的金鑰世代 **高於** 父修訂 → 這是遷移修訂。允許的條件：父修訂是該檔案在對應輪替 op 快照中的 head，或父修訂是快照 head 在同一世代內的已驗證後代且該後代在 op 的「允許遷移」清單內（見下）。不符者 → `UNSNAPSHOTTED_MIGRATION`，隔離。
3. 同世代父子 → 一般寫入；世代相同但 key_id 不同（並行輪替的兩把金鑰）視為不同世代處理：兩者之間不允許直接父子關係，必須經合併後由 active 金鑰寫出合併修訂。

**快照之外的舊世代修訂（延遲上傳、或攻擊者以舊金鑰寫入）**：

| 輪替原因 | 處理 |
|------|------|
| `routine` | 允許延遲遷移：以 active 金鑰重新加密為新修訂（parent 指向它），並把它登記到 op 的「允許遷移」清單（標頭寫入，受 MAC 保護）。它與快照 head 的遷移結果會形成分支，進入一般衝突處理，使用者看得到。 |
| 非 `routine`（主密碼、復原金鑰、不確定） | **不自動遷移、不進入衝突處理**。登記為 `quarantined`（標頭 `rekey_ops[].quarantine[]`，含 rev、hash、file_id、發現時間），狀態畫面以「外洩輪替期間出現的未信任舊世代寫入」列出。使用者可檢視其解密內容（舊金鑰仍在集合中時），並選擇「以本裝置重新提交」（視為本裝置的新寫入，以 active 金鑰加密、parent 指向目前 head）或「丟棄」。合法的離線 pending 不走這條路：持有本機可信副本的裝置在取得新金鑰後，自己以 active 金鑰重新提交（見「延遲上傳與舊裝置重新加入」），不會在 Drive 上留下舊世代修訂。 |

隔離與允許遷移清單都記在標頭、受 MAC 保護，因此每台裝置做出相同判斷。舊金鑰被清除後，隔離修訂無法再檢視內容，只能丟棄。

**狀態機**

狀態由「active 世代 a」與「未清除的非 active 世代集合」描述。取消過的輪替會留下一個 `retired` 世代，它不會在取消時清除，而是等**下一次前進的輪替 settle 時**連同其他較低世代一起清除。這樣每個轉換的 MAC 世代都是該標頭中最高的未清除世代，不會出現「要用剛清除的世代簽」的矛盾。

```
stable(a; R)            未清除 = {a: active} ∪ {r: retired | r ∈ R}；R 可為空。m = max_gen + 1
   │ 提交點 A（op 開始，direction = forward）
   ▼
migrating(a→m; R)       {a: retiring, m: active} ∪ R
   │ 所有 head 皆為 m              │ 使用者取消（僅 reason == routine）→ op.direction = revert
   ▼                              ▼
settled(m; R ∪ {a})     reverting(m→a; R)        {a: active, m: reverting} ∪ R
   {a: retired, m: active} ∪ R     │ 所有 head 皆為 a
   │ 所有裝置確認 op(forward)       ▼
   ▼                              stable(a; R ∪ {m})      {a: active, m: retired} ∪ R    ← m 保留，不清除
stable(m; ∅)
   R ∪ {a} 全部標記 purged
```

- 任何標頭中「最高世代的未清除金鑰」記為 `top`（同世代多把時取 key_id 位元組序最大者）。在 stable(a; R) 中 top = max(a, R)；取消後 top = m > a。狀態機中的 a、m 都是金鑰（key_id），世代號只用來比較。
- 正常情況 R 為空；只有「取消過且尚未再做一次前進輪替」時 R 非空。R 中的世代仍可解密（包裝保留、隨密碼變更重新包裝），不用於寫入。

**轉換授權表**（每個轉換的子標頭 MAC 必須使用的世代；不符即 HEADER_TRANSITION_DENIED）

| 父標頭狀態 | 允許的子標頭狀態 | 子標頭 MAC 世代 | 說明 |
|------|------|------|------|
| stable(a; R) | stable(a; R)（同狀態更新：改密碼、復原金鑰、命名、裝置） | top = max(a, R) | |
| stable(a; R) | migrating(a→m; R)，m = 父的 max_gen + 1 | m | m 成為新的 top |
| migrating(a→m; R) | migrating(a→m; R)（同狀態更新、遷移進度） | m | |
| migrating(a→m; R) | settled(m; R ∪ {a}) | m | 所有 head 皆為 m |
| migrating(a→m; R) | reverting(m→a; R)，且該 op 的 reason == routine | m | |
| settled(m; R') | settled(m; R')（同狀態更新、交接確認） | m | |
| settled(m; R') | stable(m; ∅)，R' 全部轉為 purged | m | 需所有裝置的 forward 確認 |
| reverting(m→a; R) | reverting(m→a; R)（同狀態更新） | m | |
| reverting(m→a; R) | stable(a; R ∪ {m}) | m | 所有 head 皆為 a；m 保留為 retired，不需交接確認 |

- 規則一：任何標頭的 `mac.key_id` 必須等於其 top。表中每一列的「子標頭 MAC 世代」都等於子標頭的 top，兩者永不衝突。
- 規則二：只有表中列出的轉換合法；`max_gen` 只能不變或 +1（且只在 stable → migrating 時 +1）；`keys[]` 條目只能新增或改狀態，不能移除；清除只發生在 settled → stable 這一個轉換，且只清除低於 m 的世代。
- 規則三：reason 非 `routine` 的輪替（主密碼、復原金鑰、不確定）**沒有** reverting 轉換，只能前進到 stable(m; ∅)。
- 推論：沿任何合法路徑，top 與 max_gen 都單調不減。持有舊世代 VK_g 的攻擊者，在裝置信任任何 top > g 的標頭之後，無法簽出任何合法的子標頭（全部需要 VK_top）。他能做的只剩在 top == g 的狀態下做同狀態更新，而那正是輪替要離開的狀態。
- 標頭包裝的 AAD 含世代號，舊世代的 VK 包裝密文無法被重新標記為新世代。

**並行輪替與標頭的合併**

兩台裝置可能各自讀到 `max_gen = 1`，同時啟動輪替，各自建立世代 2 的金鑰 K甲、K乙（key_id 不同），並各自寫出 migrating 標頭，形成標頭分支。處理原則：**在合併完成前，雙方的金鑰包裝與資料修訂全部保留，任何一把都不清除。**

```
1. 同步發現標頭有兩個 head H甲、H乙（7.4）；一切寫入停止，狀態畫面要求處理
2. 使用者選擇「主要」head（例如 H甲）。工具分別以 H甲、H乙 的包裝解出各自的金鑰集合；
   若兩邊的主密碼不同（同時改了密碼），分別提示輸入
3. 組合併標頭 H合：
     keys[]        = 兩邊 keys[] 以 key_id 取聯集；每把金鑰的 wrappings 以選定的主密碼與復原金鑰重新包裝
     rekey_ops[]   = 兩邊 ops 聯集；主要 head 的 op 繼續（direction 不變）；
                     另一個 op 標記 superseded_by = 主要 op_id，其 to_key（K乙）狀態改為 retired
                     （K乙 是由密碼持有者建立的合法金鑰，其修訂可信，之後由遷移以 active 金鑰重新加密）
     max_gen       = max(兩邊)；active = 主要 op 的 to_key；top = 最高世代中 key_id 位元組序最大者
     devices、quarantine、allowed_migrations = 聯集；naming = 主要 head 的
     parent_rev / parent_hash = 主要 head；merge_parents = [ {rev, hash} of 其他 head ]（受 MAC 保護）
     mac = HMAC(HK_top, canonical(H合 without mac))
4. 寫入 H合；之後 H甲、H乙 都不再是 head
5. 遷移：K乙 世代的 head 不在主要 op 的快照中，但 K乙 是合併標頭中已知的合法金鑰 → 自動列入主要 op 的允許遷移清單，
   由遷移以 active 金鑰重新加密；與 K甲 世代的修訂形成分支者進入一般衝突處理
```

- 兩把世代 2 的金鑰永久保留各自的 key_id；資料修訂以 key_id 選鑰，不會混淆。K乙 在下一次 settled → stable 時與其他低於 active 的金鑰一併清除。
- 若被取代的 op 是外洩輪替而主要 op 是 routine，合併後主要 op 的 reason 提升為被取代者的 reason（取較嚴格者），且不可取消。
- 標頭的 `merge_parents` 與 7.4 的資料檔 trailer 語意相同：列出的每個 head 都成為祖先，路徑驗證可沿任一父回溯。
- 同樣的合併格式也用於「兩台裝置同時改密碼」等非輪替的標頭分支。

**授權的錨點**：裝置驗證 head 的 MAC 時，所用的 VK_top 必須是**用本機的主密碼（或復原金鑰）從 head 自己的包裝解出來的**。攻擊者沒有 PK 就無法為任何 VK 做出可被解開的包裝，因此無法製造任何 top 世代不是由密碼持有者建立的標頭。轉換表限制的是「狀態能往哪裡走」，包裝限制的是「誰能建立新世代」，兩者合起來才構成授權；缺一不可。

**路徑驗證**（取代「head 相對於信任點必須是相鄰轉換」）：裝置的標頭信任點 P 到 head H 之間可能隔著多個標頭修訂（離線期間另一台完成整個輪替是正常情況）。驗證方式：

```
1. 以 7.4 的雜湊鏈由 H 回溯到 P，得到路徑 P = H_0 → H_1 → … → H_k = H（每一步 parent_hash 已驗證）
2. 對每一步 H_i → H_{i+1}：以明文狀態檢查轉換在授權表內，且 H_{i+1}.mac.key_id == H_{i+1}.top；
   合併標頭（有 merge_parents）視為同時從每個父做合法轉換，路徑可沿任一父回溯
3. MAC 驗證：
   H（head）：必須驗證，金鑰 VK_top(H) 由本機密碼從 H 的包裝解出
   中間標頭 H_i：若其 mac.key_id 在 H 中仍未清除 → 以同一把 VK 驗證；
                 若已清除 → 無法驗證 MAC，接受雜湊鏈與明文轉換檢查的結果
4. 任一步失敗 → HEADER_TRANSITION_DENIED（轉換）或 HEADER_MAC_INVALID（MAC）
```

- 中間標頭的 MAC 無法驗證時為何安全：路徑上每個標頭的完整 bytes 都由其子的 `parent_hash` 認證，最終由 H 的 MAC 認證；攻擊者若要插入或改動中間標頭，必須同時改動 H，而 H 的 MAC 需要 VK_top(H)，攻擊者沒有。中間標頭的 MAC 驗證只是多一層防禦，不是授權的依據。
- 新裝置沒有 P 時只做第 2 步的內部一致性與第 3 步的 head 驗證（FR-SYNC-11 的限制適用）。

- **寫入世代**永遠是 `active` 那一個；任何裝置寫入前先讀標頭 head 並驗證 MAC，只用 active 世代的 VK（FR-SYNC-14）。
- `retiring` / `reverting` / `retired` 世代的 VK 仍留在標頭中（以目前憑證包裝），任何解鎖的裝置都能解出，因此這些世代的修訂仍可讀。
- **遷移**（migrating 或 reverting 期間）：對每個 file_id，只遷移 **op 快照中記錄的 head**（或其在同世代內、已登記為允許遷移的後代）：解密後以 active 金鑰重新加密為新修訂（parent 指向該 head）。快照之外的舊金鑰 head 依上表處理。可中斷；重啟後重新列出，只處理仍未遷移者。遷移可由任何已解鎖的裝置分擔。reverting 的快照為進入 reverting 時另行記錄的 `revert_snapshot`。
- **從 migrating 到 settled**：確認所有 file_id 的 head 都是 m 且無分支後寫入標頭。此時 a 仍可讀，所以「確認」與「清除」之間的時間差不再有害。
- **延遲上傳與舊裝置重新加入**：任何時刻若出現非 active 金鑰的新 head，依「快照之外的舊世代修訂」表處理（routine 才允許延遲遷移；外洩輪替一律隔離）。離線裝置自己重新上線時的規則：先同步標頭 → 發現 active 世代與本機 pending 修訂的世代不同 → 以本機快取的舊標頭與舊憑證（若已更換則提示輸入舊主密碼）解出舊世代 VK，解密 pending 修訂，再以 active 世代重新加密後上傳。若舊世代已被清除（stable(m; ∅) 狀態）而本機沒有舊標頭快取，pending 修訂無法上傳，回報 `STALE_GENERATION_PENDING` 並讓使用者檢視其明文（仍可用舊憑證從本機快取解出）。
- **所有裝置確認（交接確認）**：確認紀錄綁定這一次輪替操作，存於標頭 `rekey_ops[].acks[]`：`{ device_id, direction, acked_header_rev, pending_cleared: true, at }`。裝置只有在以下全部成立時才寫入確認：已讀取並驗證含此 op、處於 `settled` 狀態的標頭；本機所有 pending 修訂都已以 active 世代重新加密並上傳成功；本機已切換寫入世代為 active（不再有舊世代寫入）。從 `settled(m; R')` 進入 `stable(m; ∅)` 的條件：`devices[]` 中每台裝置都有此 `op_id`、`direction == forward` 的確認。取消（reverting → stable）不清除任何世代，因此不需要交接確認；被取消的世代 m 之後會在下一次前進輪替的 settled → stable 時清除，而那一次的確認綁定的是那一次的 op，其「pending 已清」條件自然涵蓋任何殘留的 m 世代寫入。使用者可在設定畫面明確移除未回報的裝置（介面須警告該裝置上的 pending 修訂將無法上傳）。工具不以時間、也不以單一世代數字作為自動條件。
- **清除舊世代**：只在 settled → stable 發生；所有低於 m 的未清除世代（含先前取消留下的 retired 世代）條目改為 `purged`，其 VK 包裝從標頭移除。舊世代的修訂檔案本身在第一版不刪除（FR-SYNC-12），它們仍是雜湊鏈的一部分；因為祖先只以雜湊做關係驗證（7.4），不需要解密，清除舊 VK 不影響任何裝置（含全新裝置）的同步與驗證。若舊憑證外洩，攻擊者能解開的只有輪替前已存在的內容，與執行輪替前的狀態相同。
- **取消**：只能從 `migrating` 進入 `reverting`，且只有 reason 為 `routine` 的輪替可以取消；兩把金鑰都保留；反向遷移完成並確認所有 head 都是 a 之後，進入 `stable(a; R ∪ {m})`：m 以 `retired` 保留，**不清除**。取消後再次輪替，目標世代為 `max_gen + 1`（即 m+1），m 永不重用；該次輪替 settle 時 m 與 a 一併清除。任何時刻中斷都停留在可判讀的狀態。第一版不提供「取消後單獨清除 m」的操作。
- **憑證變更與輪替的順序**：若輪替時同時更換主密碼或復原金鑰，新憑證在提交點 A 的標頭中即對所有未清除世代生效（舊世代的 VK 也改以新憑證重新包裝）。**這不代表舊憑證失效**：保留在 Drive 上的舊標頭修訂仍含以舊憑證包裝的舊世代 VK，舊憑證仍可解開舊標頭與所有舊世代密文。新憑證達成的是：提交點 A 之後的所有新修訂、新世代 VK 與新標頭，舊憑證都無法解開。
- 遷移期間若標頭出現分支（例如另一台裝置同時改密碼），遷移暫停並要求先解決標頭衝突。

#### 5.1.7 復原金鑰

```
建立（init 或 regenerate）：
  RK        = CSPRNG(32 bytes)，以 Base32 顯示（52 字元，每 4 字一組）
  hkdf_salt = CSPRNG(32)
  RK_key    = HKDF-SHA256(ikm=RK, salt=hkdf_salt, info="hsm-recovery-v1", len=32)
  對金鑰集合中每一把未清除的 k：
    k.wrappings.recovery = { hkdf_salt（共用）, nonce=CSPRNG(12),
                             wrapped_vk = AES-256-GCM-Encrypt(key=RK_key, nonce,
                                            aad="hsm-vk-wrap-v1" || vault_id || k.key_id || k.gen, pt=VK_k) }
  寫入標頭修訂；RK 本身不存任何地方，顯示後 zeroize

重新產生復原金鑰：替換每一把未清除金鑰的 wrappings.recovery，一次標頭寫入；主密碼不受影響。
任一未清除金鑰缺少 recovery 包裝（而 recovery.enabled 為 true）視為 CORRUPTED。
使用：見 5.1.1 的復原金鑰解鎖，之後執行 5.1.5 設定新主密碼，並建議重新產生復原金鑰。
```

#### 5.1.8 演算法與參數總表

| 用途 | 演算法 | 參數與上限 |
|------|--------|------|
| 主密碼 → PK | Argon2id | 預設 256 MiB、3 次、4 並行；硬性上限 1 GiB、16 次、16 並行；32-byte salt；輸出 32 bytes |
| 復原金鑰 → RK_key | HKDF-SHA256 | 32-byte salt、info `hsm-recovery-v1` |
| VK → HK（標頭 MAC 金鑰） | HKDF-SHA256 | salt = vault_id、info `hsm-header-mac-v1` |
| 標頭完整性 | HMAC-SHA256 | 對 RFC 8785 正規化 JSON（去掉 `mac` 欄位）計算 |
| 修訂鏈 | SHA-256 | `parent_hash` 與合併父雜湊 = 父修訂整個檔案 bytes 的雜湊；祖先只以雜湊做關係驗證；信任點亦以整個檔案雜湊記錄 |
| 包裝 VK（PK、RK_key）、`vk_check` | AES-256-GCM | 12-byte 隨機 nonce、16-byte tag、AAD 含 vault_id、key_id 與世代；每把未清除金鑰各一份 |
| 包裝 DEK（VK） | AES-256-GCM | AAD 含 vault_id、file_id、rev、key_id |
| 檔案內容加密 | AES-256-GCM | 每修訂獨立 DEK、12-byte 隨機 nonce、AAD = 完整 227-byte 標頭（備份檔再串接外層雜湊） |
| 壓縮 | zstd | 加密前壓縮；壓縮後與解壓後皆上限 64 MiB |
| 隨機來源 | 作業系統 CSPRNG | `crypto/rand` 或等效 |
| 記憶體清除 | zeroize | 所有金鑰、明文緩衝區使用後覆寫 |

不使用自行設計的密碼學構造；所有原語皆為標準庫或經審計的套件實作。

### 5.2 Google Drive 上可見的資訊

| Google 可見 | Google 不可見 |
|-------------|---------------|
| 資料夾名稱與檔案名稱（皆使用者自訂或預設） | 任何 Item 名稱、帳號、密碼、私鑰、備註、標籤 |
| 檔案的 `appProperties`（`hsm.kind`、`hsm.vault_id`、`hsm.rev` 等，標示檔案用途與修訂鏈） | |
| 檔案數量與各檔案大小 | Item 型別（檔名為隨機識別碼，內容加密） |
| 檔案建立時間、修訂數量與頻率 | 主機名稱與位址 |
| Vault 標頭中的 KDF 參數、salt、VK 的包裝密文（非機密） | 主密碼、PK、VK、DEK、復原金鑰 |

檔案大小會透露粗略資訊（例如 RSA-4096 私鑰比 ed25519 大）。第一版接受此洩漏，第二階段可評估固定區塊填充（padding）。

自訂檔名可讓 Drive 上的檔案看起來不像密碼庫，但 `appProperties` 仍標示用途，且檔案內容開頭有 `HSM1` magic bytes；自訂檔名是便利與偏好功能，不是安全機制，文件須明確說明。

### 5.3 本機安全

1. VK、DEK 與明文機密僅在處理期間存在記憶體，使用後立即覆寫（zeroize）。
2. agent 程序（FR-INIT-06）以 `mlock` 或等效機制防止 VK 被換頁到磁碟；逾時或系統休眠時清除。
3. 本機快取只存密文，與 Drive 上內容相同；本機快取遭竊等同 Drive 檔案遭竊，仍受主密碼保護。
4. 任何日誌、錯誤訊息、崩潰報告不得包含機密值、主密碼、VK、OAuth token。
5. 寫入私鑰到磁碟時，先檢查目標目錄的權限（Unix：非本人以外可寫或可讀；Windows：ACL 含本人、SYSTEM、Administrators 以外的允許項目）不安全則拒絕並警告。

**檔案權限的平台差異**

| 平台 | 設定目錄 / 快取目錄 | 匯出的私鑰、明文匯出、暫存檔 | 驗證方式 |
|------|------|------|------|
| Linux / macOS | 建立時 `mkdir 0700`；檔案以 `O_CREAT \| O_EXCL`、mode `0600` 建立 | 同左；暫存檔在目標目錄內以 `O_EXCL` 建立後 rename | 寫入前 `stat` 確認 mode 與擁有者 |
| Windows | 建立時透過 `SECURITY_ATTRIBUTES` 指定 DACL：僅目前使用者 SID 完全控制，停用繼承；不依賴 `os.Chmod`（在 Windows 只影響唯讀屬性） | 同左，以 `CreateFile` 帶 DACL 建立 | 寫入前讀取 DACL，確認不含其他使用者或 `Everyone` / `Users` 的允許 ACE |

6. **絕不先寫入明文再收緊權限**：所有含明文機密的檔案都在建立那一刻就帶上最終權限。
7. agent 的通訊端點：Unix 為 `0700` 目錄內的 Unix domain socket，連線時以 `SO_PEERCRED` / `LOCAL_PEERCRED` 驗證對方 UID 等於自己；Windows 為具僅本人 DACL 的 named pipe，連線時以 `GetNamedPipeClientProcessId` 取得程序並比對其 token 的使用者 SID。詳見 5.6。

### 5.4 Google 授權安全

- 使用 OAuth 2.0 授權碼流程加 PKCE，redirect 至本機 loopback 位址。
- 預設權限範圍 `https://www.googleapis.com/auth/drive.file`：只能看到與操作本工具建立的檔案，無法讀取使用者其他 Drive 內容。
- Refresh token 存於作業系統金鑰圈；access token 只存在記憶體。
- 不使用 Google 帳號做任何身分驗證用途；Google 授權只決定「能否存取 Drive 檔案」，解密仍完全依賴主密碼。

### 5.5 威脅模型

| 威脅 | 防護措施 | 殘餘風險 |
|------|----------|----------|
| Google 帳號遭入侵，攻擊者取得 Drive 檔案 | 全部內容客戶端加密，Argon2id 抵抗離線暴力破解 | 弱主密碼可被破解；檔案大小洩漏粗略資訊 |
| 攻擊者刪除或竄改 Drive 檔案 | AES-GCM 驗證竄改；工具自己保留的不可覆寫修訂與本機快取可還原；`verify` 指令偵測 | 攻擊者可造成資料遺失（可用性），需靠獨立備份補救 |
| 攻擊者把 Drive 上的資料換回舊的合法密文（回滾） | 每台裝置記錄信任點雜湊，新 head 必須沿已驗證雜湊鏈走到信任點 | 全新裝置首次同步無法辨識回滾，需人工比對（FR-SYNC-11） |
| 攻擊者偽造中間修訂或 appProperties，讓新 head 看似信任點的後代 | appProperties 只用於搜尋；祖先關係只由雜湊鏈建立，head 由解密驗證 | 無 |
| 攻擊者新增無效的假子修訂，讓合法分支（例如撤銷）失去 head 身分而被忽略 | 只有已驗證的子修訂才能取消父的 head 身分；所有新候選一律下載驗證 | 無 |
| 攻擊者修改明文標頭的修訂鏈、世代狀態、命名規則 | 標頭 HMAC（HK 由 VK 推導），解鎖後先驗證 MAC 再採用任何欄位 | 無 |
| 持有舊世代 VK 的攻擊者簽出降級標頭（退回已外洩世代） | 狀態轉換授權表：含新世代的任何轉換都需新世代 VK；外洩輪替無取消轉換；世代編號不重用 | 無 |
| 持有舊世代 VK 的攻擊者寫入合法加密的舊世代修訂，期望被自動遷移成新世代資料 | 子世代低於父世代一律拒絕；遷移只處理輪替快照中的 head；外洩輪替中快照之外的舊世代修訂只隔離不遷移 | routine 輪替允許延遲遷移，但結果以分支呈現、需使用者處理 |
| 本機電腦遭竊 | 快取為密文；OAuth token 在金鑰圈；agent 逾時清除 VK | 電腦處於 unlocked 且未鎖定螢幕時無保護 |
| 本機惡意軟體 | 超出本工具防護範圍 | 可讀取記憶體中的 VK 與明文；列為已知限制 |
| 同一台電腦上的其他使用者帳號 | 所有檔案建立時即為僅本人可讀（含 Windows ACL）；agent 端點驗證呼叫者身分 | 管理員權限的帳號仍可讀取 |
| 惡意或損毀的檔案（長度欄位、KDF 參數被竄改） | 配置記憶體前檢查長度與檔案大小；執行 KDF 前檢查參數上限；完整標頭納入 AAD | 無 |
| 主密碼外洩但攻擊者只有舊密文 | 全面輪替產生新世代並重新加密；`gc` 清除舊世代與舊標頭修訂 | Google 伺服器端可能仍保留已刪除檔案一段時間 |
| 主密碼遺忘 | 復原金鑰 | 兩者皆遺失則資料永久不可讀 |
| 多裝置同時修改 | 每次寫入為獨立不可覆寫修訂，事後以修訂鏈偵測分支，雙方都保留 | 需使用者手動解決衝突 |
| 工具本身的 OAuth client 憑證外洩 | 桌面應用程式的 client secret 本就不視為機密；PKCE 防止授權碼攔截 | 他人可冒用 client 名稱釣魚，需在文件提醒使用者 |


### 5.6 GUI 與 agent 強化

**桌面 GUI（WebView）**

1. WebView 只載入隨程式打包的前端資源（自訂 scheme 或 localhost 內嵌伺服器綁定隨機埠與一次性 token），不載入任何遠端 URL。
2. Content-Security-Policy：`default-src 'self'; script-src 'self'; connect-src 'self'; img-src 'self' data:; object-src 'none'; frame-ancestors 'none'`。禁止 `eval` 與 inline script。
3. 使用者輸入的任何欄位（名稱、備註、網址、匯入的 CSV 內容、Chrome 資料）一律以文字節點渲染，絕不當作 HTML 插入。
4. 外部連結只允許 `http://` 與 `https://`，以作業系統預設瀏覽器開啟，開啟前顯示完整網址請使用者確認；其他協定一律拒絕。
5. 正式版停用開發者工具與右鍵檢視原始碼；前端與 Go 後端之間的呼叫介面只暴露明確列舉的方法。
6. 剪貼簿寫入只由使用者動作觸發；視窗內的機密值在遮罩狀態下不存在於 DOM。

**本機 agent（CLI 解鎖狀態維持）**

1. 端點與呼叫者驗證見 5.3 第 7 點；驗證失敗的連線立即關閉並記錄。
2. agent **從不輸出任何 VK**。提供的操作只有：`status`、`lock`、`unwrap_dek(file_header)`（依標頭中的 key_id 選鑰，回傳該檔案的 DEK）、`wrap_dek(file_header, dek)`（只允許 key_id 為目前 active 金鑰）、`header_mac(canonical_bytes)`（以 top 計算標頭 MAC）、`keys()`（回傳集合中每把金鑰的 key_id、世代、狀態，不含金鑰本身）。內容的加解密由呼叫端自行完成。
3. 每個操作需帶 Vault ID；agent 每個 Vault 保存**整個金鑰集合**（所有未清除金鑰，以 key_id 索引），並記錄哪一把是 active、哪一把是 top。標頭 head 變更時由呼叫端通知 agent 重新載入。
4. 逾時、系統休眠、使用者鎖定螢幕（可偵測時）、收到 `lock` 時清除所有 VK。
5. 對 `unwrap_dek` 做速率限制（預設每秒 50 次），超過即暫停並記錄，避免被用來大量解包。
6. agent 以目前使用者身分執行，不需也不可提權。

---

## 6. 系統架構

### 6.1 元件

```
┌───────────────────────────────────────────────────────┐
│                     使用者的電腦                        │
│                                                       │
│  ┌──────────┐    ┌──────────────┐                     │
│  │ 桌面 GUI  │───▶│ Core 程式庫   │                     │
│  └──────────┘    │ ├ Vault 管理  │    ┌──────────────┐ │
│  ┌──────────┐    │ ├ 加密層      │◀──▶│ agent（選用） │ │
│  │ CLI      │───▶│ ├ 金鑰引擎    │    │ 保存 VK     │ │
│  └──────────┘    │ ├ 同步引擎    │    └──────────────┘ │
│                  │ ├ 匯入引擎    │                     │
│                  │ └ 儲存層介面  │                     │
│                  └──────┬───────┘                     │
│            ┌────────────┼──────────────┐              │
│            ▼            ▼              ▼              │
│   ~/.config/hsm/   ~/.cache/hsm/   OS 金鑰圈           │
│   設定、裝置 ID     加密快取、索引    OAuth token        │
└────────────┬──────────────────────────────────────────┘
             │ HTTPS（Google Drive API v3）
             ▼
   ┌──────────────────────────────┐
   │ Google Drive（名稱皆為預設值）│
   │ My Drive/HolienSecretsManager│
   │  ├ vault.hsm                 │
   │  ├ index.enc                 │
   │  ├ log.enc                   │
   │  ├ hosts.enc                 │
   │  └ items/<id>.enc ...        │
   └──────────────────────────────┘
```

- 本產品為單機桌面應用程式，沒有伺服器端。Core 程式庫包含所有邏輯，桌面 GUI 與 CLI 都只是薄殼，兩者可同時安裝並共用同一本機快取與設定。
- 匯入引擎處理 Chrome 與其他密碼管理器的資料來源，輸出統一的候選清單交給 Vault 管理層做重複判斷。
- 儲存層介面抽象 Drive 操作，測試時可以本機資料夾實作替代。
- agent 為選用程序，不存在時每個指令各自提示主密碼。

### 6.2 操作流程（以「匯出私鑰」為例，GUI 與 CLI 相同）

1. 介面層讀取設定，取得 Vault 的 Drive 資料夾 ID 與本機快取路徑。
2. 取得 VK：GUI 於解鎖後保存在程序記憶體；CLI 透過 agent 解包 DEK，agent 不存在或已逾時則提示主密碼，以標頭的 KDF 參數推導 PK 解開 VK，並以 `vk_check` 確認。
3. 同步索引：列出索引的修訂，若 head 未變則略過；否則下載新 head，解密，更新本機索引快取。離線模式略過此步。
4. 依名稱在索引中找到 Item ID，列出該 Item 的修訂確認 head，確認鏈包含本機信任點（FR-SYNC-10），本機快取沒有 head 則下載。
5. 解密 Item 檔案，取出 `active` 版本（或 `--version` 指定）的私鑰，依要求格式轉換。
6. 檢查該版本狀態非 `revoked`（或已 `--force`）；`retired` 時顯示警告。
7. 以僅本人可讀的方式建立目標檔案並寫入（5.3）。
8. 寫入本機操作紀錄並排程上傳 `log.enc`。
9. zeroize 記憶體中的私鑰與 DEK。

### 6.3 本機目錄

| 路徑 | 內容 | 權限 |
|------|------|------|
| `~/.config/hsm/config.yaml` | Vault 清單、各 Vault 的 Drive 資料夾 ID、預設 Vault、agent 逾時等設定 | 0600 |
| `~/.config/hsm/device_id` | 本裝置識別碼（UUID） | 0600 |
| `~/.cache/hsm/<vault_id>/` | 各邏輯檔案 head 修訂的快取，以及 `sync_state.json`（每個 `file_id` 的 head rev、信任點 rev 與 SHA-256、金鑰世代、pending 標記、rev 與 Drive 檔案 ID 對應）。本機快取一律使用固定內部名稱（`header`、`index`、`log`、`hosts`、`items/<id>`），與 Drive 上使用者自訂的檔名無關 | Unix 0700 / 0600；Windows 僅本人 ACL |
| OS 金鑰圈 | OAuth refresh token，鍵名 `hsm/<vault_id>/google` | 由系統管理 |

Windows 對應路徑為 `%APPDATA%\hsm\` 與 `%LOCALAPPDATA%\hsm\cache\`。

---

## 7. Google Drive 整合

### 7.1 OAuth client 與權限範圍

- 工具使用「桌面應用程式」類型的 OAuth client。此類 client 的 client secret 依 Google 文件不視為機密，可隨程式發布。
- 預設權限範圍：`drive.file`。此範圍下工具只能看到自己建立的檔案與資料夾。**影響**：若使用者指定的資料夾路徑已存在但由其他方式建立（例如使用者在網頁手動建立），工具看不到它，會建立一個同名的新資料夾。文件須明確說明此行為，並提供 `--scope full` 選項改用完整 `drive` 範圍（需重新授權）。
- OAuth client 的提供方式見待決議 Q-02：由專案內建，或由使用者自行在 Google Cloud Console 建立並提供 client ID。

### 7.2 資料夾解析與建立

```
輸入：folder_path（使用者自訂，預設 "HolienSecretsManager"）
1. 以 "/" 切割為 segments。
2. parent = "root"。
3. 對每個 segment：
   a. 查詢：name = segment AND mimeType = folder AND parent in parents AND trashed = false。
   b. 找到一個 → parent = 該資料夾 ID。
   c. 找到多個 → 回報錯誤，要求使用者在 Drive 整理後重試（不自行猜測）。
   d. 找不到 → 建立資料夾，parent = 新資料夾 ID。
4. 最終 parent 即 Vault 資料夾。
5. 以 query「parent = 該資料夾 AND appProperties has { key='hsm.kind' and value='header' }」列出所有標頭修訂（不依賴檔名），依 appProperties 的 hsm.vault_id 分組：
   - 多於一個 vault_id → 回報錯誤（資料夾內有多個 Vault），要求使用者整理。
   - 恰一個 vault_id → 這是既有 Vault。多個標頭修訂是正常情況（每次改密碼、改命名規則都會新增一個）。
     下載全部標頭修訂（都很小），解鎖後依 7.4 驗證 MAC 與雜湊鏈，找出 head：
       恰一個 head → 採用其命名規則與世代狀態；
       多個 head → HEADER_BRANCHED，要求使用者選擇。
     新裝置沒有標頭信任點，接受驗證通過的 head 並以它建立信任點（FR-SYNC-11 的限制在此適用）。
   - 找不到且資料夾為空 → 以使用者指定的命名規則建立新 Vault。
   - 找不到但資料夾非空 → 詢問使用者是否仍在此建立新 Vault。
6. 在資料夾上設定 appProperties：{ "hsm.vault_id": <uuid>, "hsm.format": "1" }，供日後快速辨識。
7. 連接既有 Vault 時另提供「以 vault_id 搜尋」途徑：query「appProperties has { key='hsm.kind' and value='header' }」列出此 Google 帳號下所有本工具建立的 Vault 標頭，使用者從清單選擇，不需記得資料夾路徑。
```

### 7.3 檔案配置與命名規則

所有名稱皆為**使用者自訂、系統提供預設值**。下表的「預設檔名」只是預設值，實際名稱由 Vault 標頭的 `naming` 區段決定（見 8.1）。

| 命名規則鍵 | 預設檔名 | `hsm.kind` | 內容 | 是否加密 | 更新時機 |
|------------|----------|------------|------|:--:|------|
| `header` | `vault.hsm` | `header` | Vault 標頭：格式版本、vault_id、各世代的 VK 包裝（主密碼、復原金鑰）、`vk_check`、命名規則、建立時間 | 部分（見 8.1） | 建立、變更主密碼、更新復原金鑰、修改命名規則、全面輪替提交點 |
| `index` | `index.enc` | `index` | 所有 Item 的摘要：id、名稱、型別、標籤、狀態、updated_at、目前版本號、到期日、綁定主機 | 是 | 任何 Item 新增、改名、改標籤、狀態變更 |
| `log` | `log.enc` | `log` | 操作紀錄（滾動保留最近 5,000 筆） | 是 | 每次操作後（可批次延後上傳） |
| `hosts` | `hosts.enc` | `hosts` | 所有 Host 紀錄與綁定關係 | 是 | Host 變更時 |
| `items_folder` | `items` | `items_folder` | 存放 Item 檔案的子資料夾 | 不適用 | 建立時 |
| `item_pattern` | `{id}.enc` | `item` | 單一 Item 的全部版本；`{id}` 替換為該 Item 的 UUID v4 | 是 | 該 Item 變更時 |

- `{id}` 為 UUID v4，與 Item 內容無關。`item_pattern` 可為任意含 `{id}` 的樣式，例如 `IMG_{id}.jpg`、`{id}`（無副檔名）。所有命名規則鍵都可額外使用 `{rev}` 佔位符（修訂 UUID 前 8 碼）；不使用時，同一邏輯檔案的多個修訂在 Drive 上會是多個同名檔案（Drive 允許同名），工具以 `appProperties` 區分。
- 每個邏輯檔案在 Drive 上是「一個或多個修訂檔案」。head 的判定見 7.4，不是「最新修改時間」。
- `items_folder` 設為空字串時，Item 檔案直接放在 Vault 資料夾內，不建子資料夾。
- 工具讀寫檔案時一律先以 `appProperties`（`hsm.kind`、`hsm.vault_id`、`hsm.file_id`、`hsm.rev`、`hsm.parent_rev`、`hsm.gen`，合併修訂另有 `hsm.merge_parents`）查詢，並快取在 `sync_state.json`；檔名只在建立與重新命名時使用。
- 所有檔案以 `multipart` 或 `resumable` 上傳；單檔小於 5 MB 用 multipart。
- 修改命名規則（FR-DRIVE-16）流程：更新標頭 `naming` 並上傳 → 逐一 `files.update(name=...)` → 全部成功後更新本機快取的對應；任一步失敗時狀態畫面顯示「命名規則更新未完成」，可重試。

### 7.4 修訂鏈、head 判定與衝突

**寫入**：永遠 `files.create` 新檔案，受驗證的標頭含 `rev`（新 UUID）、`parent_rev` 與 `parent_hash`（本機信任 head 的 rev 與整個檔案的 SHA-256）；合併修訂另在明文 trailer 列出其他父的 rev 與雜湊。從不 `files.update` 資料檔內容，也從不依賴 Drive 的 revision 歷史。

**兩種驗證**

| 方式 | 需要 | 得到 |
|------|------|------|
| 內容驗證 | 該修訂世代的 VK | 解密成功，標頭（含 parent_hash、trailer_hash）經 AES-GCM AAD 驗證，內容可用 |
| 關係驗證 | 一個已驗證的子修訂 C 與本修訂 P 的完整 bytes | SHA-256(P bytes) == C 所宣告的父雜湊 → P 的**全部 bytes**（含其標頭中的 parent_rev、parent_hash、generation 與 trailer）由 C 間接認證；P 的內容不需解密、不需 P 世代的 VK |

因此只有 head 需要內容驗證；head 以下的整條祖先鏈都只需關係驗證。清除舊世代 VK 後，任何裝置（含全新裝置）仍能驗證完整歷史關係。

**信任的來源**：appProperties（`hsm.file_id`、`hsm.rev`、`hsm.parent_rev`、`hsm.gen`）只用來**找候選檔案與安排下載順序**，不參與任何判定。本機保有兩個分開的集合：

| 集合 | 內容 | 生命週期 |
|------|------|------|
| `verified_revs`（曾驗證的歷史） | 每個 file_id：rev → {hash, parent_rev, parent_hash, 合併父, generation, verified_by: content \| relation, drive_file_id} | 跨次同步累積，只增不減；用於免重複下載與離線復原 |
| `remote_present`（本次遠端存在） | 本次 list 回傳、且確認與登錄內容一致的 rev 集合 | 每次同步重建，不保留 |

`remote_present` 的建立：對 list 回傳的每個候選，若其 rev 已在 `verified_revs` 且 Drive 回報的 `sha256Checksum`（檔案 metadata 欄位）等於登錄的 hash → 視為存在；不等或 Drive 未提供 → 重新下載驗證。Drive 的校驗欄位只用來省下載，不作為完整性依據。**head 與回滾判定只在 `verified_revs ∩ remote_present` 上進行**；本機登錄中有、遠端沒有的修訂，不能讓遠端看起來仍然完整。

**同步一個邏輯檔案（file_id）**：

```
輸入：本機信任點 T = {rev_T, hash_T}（新裝置或首次同步時為空）；本機 verified_revs

1. 候選：list appProperties hsm.file_id = X（分頁取完）→ 候選集合 C
2. 下載：C 中每個「rev 不在 verified_revs 中」的候選都下載（候選只決定順序，不決定是否可跳過）。
   同一 rev 出現多個候選：全部下載；bytes 雜湊不同 → HASH_MISMATCH，此 file_id 標記需人工處理
3. 內容驗證：對每個下載的修訂嘗試 5.1.3。
   成功 → 登錄為 verified_by: content
   失敗（含世代已清除 STALE_GENERATION）→ 暫存為「待關係驗證」
4. 關係驗證（反覆直到沒有新進展）：
   對每個已驗證修訂 V 的每個父參照 {parent_rev, parent_hash}（含 trailer 合併父）：
     父 bytes 在手且 SHA-256 == parent_hash → 父登錄為 verified_by: relation（若尚未登錄）
     父 bytes 在手但雜湊不符 → HASH_MISMATCH
     父不在候選中也不在 verified_revs → MISSING_PARENT（獨立錯誤，與 head 數量無關）
5. 仍未驗證的候選：既無法解密、也沒有任何已驗證修訂以它為父 → 捨棄，記錄 UNVERIFIED_CANDIDATE
   （它可能是攻擊者的假檔案，也可能是離線裝置以已清除世代上傳的修訂；兩者都不能影響 head 判定）
6. 遠端存在集合：S = verified_revs ∩ remote_present（見上表）。
   C 為空且本機有此檔案的快取 → REMOTE_DELETED：該檔案在 Drive 已被刪除，標記 missing，提供「從本機快取重新上傳」（7.5）
   T 非空且 rev_T ∉ S → 信任點已從遠端消失 → ROLLBACK_SUSPECTED（遠端歷史被移除），停止寫入，待使用者確認；
     使用者確認後可選擇「以本機快取重新上傳信任分支」或「接受遠端現狀並重設信任點」
7. head 判定（只看 S）：
   head = S 中沒有任何 S 內的修訂以它為父（parent_rev 或合併父）者
   → 未驗證候選即使宣稱以某修訂為父，也不能取消該修訂的 head 身分
   → 本機登錄中有、遠端已不存在的修訂，不參與判定
8. 回滾與分支的區分（T 非空時）：
   D = 能沿 S 內已驗證的父參照走到 rev_T 的 head（信任點的後代，含 rev_T 自己是 head 的情況）
   D 為空 → 沒有任何 head 延續本機信任的歷史 → ROLLBACK_SUSPECTED，停止寫入，待使用者確認
   D 非空 → 本機信任的分支仍在遠端。其他不屬於 D 的 head：
     與 rev_T 有共同祖先（沿父參照可走到 rev_T 的某個祖先）→ **合法的兄弟分支**（另一台裝置以較早的
       parent 離線寫入），進入第 10 步的分支處理，不視為回滾
     與 rev_T 的鏈完全不相交（沒有共同祖先）→ FOREIGN_CHAIN，捨棄並警告
   |head| == 0 且 C 非空 → CORRUPTED_CHAIN
9. |head| == 1：採用；更新本機快取與信任點 T = {rev, hash}
   （head 必須是 verified_by: content；若唯一的 head 只有關係驗證，表示它的世代已清除而沒有後代 → STALE_GENERATION_HEAD，需人工處理）
10. |head| > 1：分支
     index：以所有 Item 的 head 重建索引，寫入合併修訂（trailer 列出所有 head 的 {rev, hash}）
     log：解密各 head，取紀錄聯集，寫入合併修訂
     item / hosts：內容完全相同 → 寫入合併修訂；否則標記 conflict，列入待處理，禁止寫入並套用 FR-SYNC-18
     header：禁止一切寫入，要求使用者選擇（兩份皆可解鎖）
     某個 head 無法內容驗證（例如世代已清除）→ 不能自動合併，列入待處理並說明
```

- 分支只會在「兩台裝置以相同 parent 各自寫入」時出現，和兩者寫入的先後或時鐘無關。
- 解決衝突的修訂把所有 head 列入 trailer（含雜湊），之後它們都不再是 head。
- 離線寫入累積多個本機修訂時，上線後依序上傳（每個的 parent 是前一個），只會形成一條鏈。
- 標頭也是邏輯檔案，同樣走此流程；差別是內容驗證方式為 MAC（8.1）而非 AES-GCM，head 另需通過 8.1 的轉換授權檢查，合併以 JSON 內受 MAC 保護的 `merge_parents` 表示，且舊標頭（以已清除金鑰簽 MAC）同樣只需關係驗證。
- 在第 3 步內容驗證之後、第 7 步 head 判定之前，對每個已驗證修訂套用 5.1.6 的跨世代父子規則：世代回退者（GENERATION_REGRESSION）與快照外的跨世代遷移（UNSNAPSHOTTED_MIGRATION）移出 S，列入隔離區；它們不能成為 head，也不能取消任何修訂的 head 身分。
- 成本：每次同步下載「verified_revs 之外」的所有新修訂，通常是零到數個小檔案；已登錄者只比對 Drive 回報的校驗值；第一次同步或全新裝置下載全部歷史（第一版不清理，見 FR-SYNC-12）。

**推演：正常離線衝突不是回滾（審閱案例）**

兩台裝置原本都信任 R。甲寫出 A（信任點變為 A）；乙離線寫出 B（parent 仍為 R）。甲同步：S = {R, A, B}，head = {A, B}；第 8 步 D = {A}（A 是信任點本身），非空；B 不在 D 中，但 B 的父 R 是 A 的祖先 → 兄弟分支 → 第 10 步進入衝突處理。沒有回滾警告。

**推演：遠端刪除被本機登錄掩蓋（審閱案例）**

本機登錄有 R、A，信任點 A；Drive 上 A 被刪除。list 只回傳 R → remote_present = {R}，S = {R}；第 6 步 rev_T = A ∉ S → ROLLBACK_SUSPECTED（遠端歷史被移除），並提供從本機快取重新上傳 A。若 Drive 上整個檔案被清空，C 為空 → REMOTE_DELETED。本機登錄中的 A 不會讓程式誤以為遠端仍有 head。

**推演：假子修訂隱藏撤銷分支（審閱案例）**

T 分出合法修訂 A（active）與 B（revoked）；攻擊者加入假修訂 F 宣稱父為 B。第 2 步下載 A、B、F；第 3 步 A、B 內容驗證成功，F 失敗；第 4 步 F 不是任何已驗證修訂的父，無進展；第 5 步捨棄 F；第 6 步在 {A, B} 上判定 head：A、B 都沒有已驗證的子 → 兩個 head → 分支 → conflict，撤銷分支不會消失。

### 7.5 API 配額與錯誤處理

| 狀況 | 處理 |
|------|------|
| HTTP 403 `rateLimitExceeded` / 429 | 指數退避：1s、2s、4s、8s、16s，最多 5 次 |
| HTTP 5xx | 同上 |
| HTTP 401 | 以 refresh token 更新 access token 後重試一次；仍失敗則要求重新授權 |
| list 為空而本機有快取（檔案被使用者刪除） | `REMOTE_DELETED`：索引中標記該 Item 為 `missing`，`status` 顯示並提供「從本機快取重新上傳」或「從備份還原」 |
| 網路不可用 | 立即改為離線模式，寫入操作保留 pending |
| Drive 空間不足 | 回報錯誤，保留 pending |

---

## 8. 資料格式

### 8.1 Vault 標頭 `vault.hsm`

JSON，明文可讀（不含機密），整份以 HMAC 保護。標頭也是一個邏輯檔案，有自己的雜湊鏈（`rev`、`parent_rev`、`parent_hash` 在 JSON 內並受 MAC 保護；appProperties 的副本只供搜尋）。

```json
{
  "format": 1,
  "vault_id": "0190a4c2-....",
  "rev": "....",
  "parent_rev": "....",
  "parent_hash": "<base64 32 bytes：父標頭修訂整個檔案的 SHA-256>",
  "created_at": "2026-10-28T08:00:00Z",
  "merge_parents": [ { "rev": "....", "hash": "<base64 32 bytes>" } ],
  "max_gen": 1,
  "rekey_ops": [
    {
      "op_id": "....", "from_key": "<key_id>", "to_key": "<key_id>", "to_gen": 2,
      "reason": "routine", "direction": "forward", "superseded_by": null,
      "started_at": "...", "started_header_rev": "....",
      "snapshot": { "index": { "rev": "....", "hash": "..." }, "hosts": { "rev": "....", "hash": "..." }, "log": { "rev": "....", "hash": "..." } },
      "revert_snapshot": null,
      "allowed_migrations": [ { "file_id": "....", "rev": "....", "hash": "..." } ],
      "quarantine": [ { "file_id": "....", "rev": "....", "hash": "...", "key_id": "...", "seen_at": "..." } ],
      "acks": [ { "device_id": "...", "direction": "forward", "acked_header_rev": "....", "pending_cleared": true, "at": "..." } ]
    }
  ],
  "keys": [
    {
      "key_id": "<uuid>",
      "gen": 1,
      "status": "active",
      "wrappings": {
        "password": {
          "kdf": { "algorithm": "argon2id", "memory_kib": 262144, "iterations": 3, "parallelism": 4, "salt": "<base64 32 bytes>" },
          "nonce": "<base64 12 bytes>",
          "wrapped_vk": "<base64 48 bytes>"
        },
        "recovery": {
          "enabled": true,
          "hkdf_salt": "<base64 32 bytes>",
          "nonce": "<base64 12 bytes>",
          "wrapped_vk": "<base64 48 bytes>"
        }
      },
      "vk_check": { "nonce": "<base64 12 bytes>", "ciphertext": "<base64>" }
    }
  ],
  "naming": {
    "header": "vault.hsm",
    "index": "index.enc",
    "log": "log.enc",
    "hosts": "hosts.enc",
    "items_folder": "items",
    "item_pattern": "{id}.enc"
  },
  "devices": [
    { "device_id": "...", "name": "macbook", "first_seen": "...", "last_seen_header_rev": "...." }
  ],
  "active_key": "<key_id>",
  "mac": { "key_id": "<uuid>", "value": "<base64 32 bytes>" }
}
```

- `keys[]` 以 `key_id` 識別，`gen` 只用於排序；`status` ∈ `active` / `retiring` / `retired` / `reverting` / `purged`；`purged` 條目沒有 `wrappings` 與 `vk_check`，但條目本身永不移除。`retired` 金鑰可能不只一把（每次取消或並行輪替合併留下一把），全部在下一次 settled → stable 時清除。
- `merge_parents` 只在合併標頭中非空，受 MAC 保護，語意同資料檔 trailer（7.4、5.1.6「並行輪替與標頭的合併」）。
- `rekey_ops` 上例為示意；`single` 狀態下最後一個 op 已完成。`acks` 的語意見 5.1.6。

**完整性**

- `mac.value = HMAC-SHA256(HK_gen, canonical(header without "mac"))`，其中 `HK_gen = HKDF-SHA256(ikm=VK_gen, salt=vault_id, info="hsm-header-mac-v1", len=32)`，`canonical` 為 RFC 8785（JSON Canonicalization Scheme）序列化。
- `mac.key_id` 必須等於此標頭的 top（最高世代的未清除金鑰，同世代多把時取 key_id 位元組序最大者）；讀取時用該金鑰驗證（任何解鎖的裝置都能以密碼包裝解出整個集合）。
- **轉換授權**：從本機標頭信任點到 head 的路徑上，每一個相鄰轉換都必須在 5.1.6 的授權表內，每個標頭的 MAC 世代等於其 top，`max_gen` 與 top 單調不減，`keys[]` 只增不減；任一不符 → HEADER_TRANSITION_DENIED。head 的 MAC 金鑰必須由本機密碼從 head 自己的包裝解出（5.1.6「授權的錨點」）。新裝置沒有信任點時只檢查內部一致性，並以該 head 建立信任點（FR-SYNC-11 的限制在此適用）。
- 舊標頭修訂（以已清除世代簽 MAC）無法再做 MAC 驗證，但可由子標頭的 `parent_hash` 做關係驗證（7.4）。
- 讀取順序（FR-INIT-12）：解析 JSON（大小上限 1 MiB）→ 檢查 `format` 與每個 `kdf` 的參數上限 → 以密碼或復原金鑰解開整個金鑰集合 → `vk_check` → **驗證 MAC** → 此後才可採用 `rev`、`parent_rev`、`parent_hash`、`keys[].status`、`naming`、`devices`。MAC 失敗 → HEADER_MAC_INVALID，該修訂視同不存在。
- 標頭的雜湊鏈驗證同 7.4：新 head 必須沿 `parent_rev` / `parent_hash` 走到本機的標頭信任點。
- 未清除金鑰的合法組合只有 5.1.6 狀態機列出的四種形態（stable / migrating / settled / reverting，各自可附帶零或多把 `retired` 金鑰）；其他組合視為 CORRUPTED。
- `devices[].last_seen_header_rev` 由各裝置每次同步時更新（僅供顯示）；輪替的交接確認存於 `rekey_ops[].acks`（5.1.6），不以 `devices` 的欄位判定。
- 每把金鑰的 `wrappings.password` 與 `wrappings.recovery` 包裝的是該把 VK，AAD 為 `"hsm-vk-wrap-v1" || vault_id || key_id || gen`；所有未清除金鑰共用同一組 KDF 參數與 salt、同一個 `hkdf_salt`。
- 每把金鑰的 `vk_check.ciphertext` 為字串 `"hsm-vk-check-v1"` 以該 VK 加密的結果，AAD 為 `vault_id || key_id || gen`。
- 讀取標頭時先檢查 `format`、各 `kdf` 參數上限（FR-INIT-11）、base64 欄位長度，再做任何推導。
- `naming` 為此 Vault 的命名規則，值皆為使用者自訂，上例為預設值。
- `devices` 清單僅供顯示，不用於授權。

### 8.2 加密檔案格式（index、items、log、hosts 的每個修訂，以及備份檔內層）

這是**唯一**的位元組格式，5.1.2 與 5.1.3 的描述以此為準。所有整數為 big-endian，標頭固定 227 bytes，檔案結構為 `標頭 || 密文 || trailer`：

| 偏移 | 長度 | 欄位 | 說明 |
|------|------|------|------|
| 0 | 4 | magic | ASCII `HSM1` |
| 4 | 1 | format | 格式版本，目前為 1 |
| 5 | 1 | kind | 1 = index、2 = item、3 = log、4 = hosts、5 = backup 內層 |
| 6 | 16 | vault_id | UUID bytes |
| 22 | 16 | file_id | Item 的 UUID；非 Item 檔案為固定 UUID（index、log、hosts、backup 各一） |
| 38 | 16 | rev | 此修訂的 UUID |
| 54 | 16 | parent_rev | 上一個修訂的 UUID；首個修訂為全零 |
| 70 | 32 | parent_hash | 父修訂**整個檔案 bytes** 的 SHA-256；首個修訂為全零 |
| 102 | 16 | key_id | 包裝 DEK 所用 VK 的 key_id；世代由標頭 keys[] 對應 |
| 118 | 12 | dek_nonce | 包裝 DEK 用 |
| 130 | 48 | wrapped_dek | 32 bytes DEK + 16 bytes GCM tag |
| 178 | 12 | nonce | 內容加密用 |
| 190 | 4 | plaintext_len | 壓縮後明文長度（= ciphertext 長度 − 16） |
| 194 | 1 | merge_count | trailer 中合併父的數量（0–255），一般寫入為 0 |
| 195 | 32 | trailer_hash | SHA-256(trailer bytes)，**沒有特例**：merge_count = 0 時為 SHA-256 的空輸入值 `e3b0c442…b855` |
| 227 | plaintext_len + 16 | ciphertext | zstd 壓縮內容的 AES-256-GCM 密文 + 16 bytes tag |
| 227 + plaintext_len + 16 | merge_count × 48 | trailer | 明文；每筆 = 合併父 rev（16 bytes）+ 該父整個檔案的 SHA-256（32 bytes） |

- **AAD = 偏移 0 到 226 的完整標頭（227 bytes）**，包含 `parent_hash`、`plaintext_len`、`merge_count` 與 `trailer_hash`。kind = 5 時 AAD = 標頭 || SHA-256(備份外層 JSON bytes)（8.7）。
- trailer 為明文，目的是讓合併父的關係驗證**不需解密**（7.4）；它經 `trailer_hash` 綁入 AAD，對 head 而言受 AES-GCM 保護，對祖先而言受子修訂的 `parent_hash`（覆蓋整個檔案 bytes）保護。明文 JSON 內不再另存 `merge_parents`。
- 解析順序與硬性上限（5.1.3）：先檢查檔案總長度在 `[243, 80 MiB]`，再檢查 `plaintext_len ≤ 64 MiB` 且總長度 == 227 + plaintext_len + 16 + merge_count × 48，再比對 `trailer_hash`；全部通過才配置緩衝區與解密。長度欄位在驗證前只用來做這些比對，不用來配置與明文長度相關的記憶體。
- 內容為 JSON，經 zstd 壓縮後再加密（壓縮於加密前，個人使用情境不考慮壓縮側通道）。解壓縮輸出上限 64 MiB。

### 8.3 Item 內容（解密後 JSON）

```json
{
  "id": "0190a4c2-....",
  "rev": "....",
  "parent_rev": "....",
  "type": "key_pair",
  "name": "prod-web-01-deploy",
  "tags": ["prod", "web"],
  "expires_at": null,
  "rotation_days": 90,
  "created_at": "...",
  "updated_at": "...",
  "updated_by_device": "...",
  "deleted_at": null,
  "current_version": 2,
  "versions": [
    {
      "version": 2,
      "status": "active",
      "status_changed_at": "...",
      "revoked_reason": null,
      "created_at": "...",
      "created_by_device": "...",
      "note": "rotated",
      "fields": { "...型別專屬欄位..." }
    },
    { "version": 1, "status": "retired", "...": "..." }
  ]
}
```

- `rev`、`parent_rev` 必須與檔案標頭一致，解密後比對，不一致即 CORRUPTED；合併父只存於檔案 trailer（8.2）。
- 狀態在**每個版本**上：`active` / `retired` / `revoked`。帳密型 Item 的版本只有 `active`（目前）與 `retired`（歷史）。
- `conflict` 不存在檔案內；衝突是本機從修訂鏈推導的狀態，存於 `sync_state.json`。

**各型別 `fields`**

| type | fields |
|------|--------|
| `credential` | `username`, `password`, `url`, `notes`, `totp_secret?` |
| `generic` | `entries: { key: value }` 或 `content: string` |
| `key_pair` | `algorithm`, `purpose`, `comment`, `private_key_pem`（PKCS#8）, `private_key_passphrase?`, `public_key_openssh`, `public_key_pem`, `fingerprint_sha256`, `fingerprint_md5` |
| `tls_certificate` | `private_key_pem`, `certificate_pem`, `chain_pem?`, `subject`, `sans[]`, `issuer`, `not_before`, `not_after`, `serial`, `fingerprint_sha256` |

### 8.4 索引內容

```json
{
  "generated_at": "...",
  "items": [
    {
      "id": "...", "type": "key_pair", "name": "prod-web-01-deploy",
      "tags": ["prod"], "status": "active", "updated_at": "...",
      "current_version": 2, "expires_at": null, "head_rev": "...", "head_hash": "<base64 32 bytes>",
      "summary": { "fingerprint_sha256": "SHA256:...", "algorithm": "ed25519" }
    }
  ]
}
```

- `head_rev` 與 `head_hash` 為寫入索引時各 Item head 的 rev 與整個檔案的 SHA-256；輪替快照以索引的 {rev, hash} 間接認證所有 Item 的 head（5.1.6）。
- `summary` 只放非機密、常用於列表的欄位（帳密型放 `username` 與 `url`；金鑰型放指紋與演算法）。
- 索引可由所有 Item 檔案重建。

### 8.5 Hosts 內容

```json
{
  "hosts": [
    {
      "id": "...", "name": "prod-web-01", "address": "203.0.113.10",
      "ssh_port": 22, "environment": "prod", "tags": [], "notes": "",
      "host_key_fingerprint": "SHA256:...",
      "bindings": [
        {
          "item_id": "...", "key_version": 2, "remote_user": "deploy", "usage": "部署",
          "deploy_state": "installed",
          "declared_at": "...", "installed_at": "...", "removal_pending_since": null, "removed_at": null
        }
      ],
      "created_at": "...", "updated_at": "..."
    }
  ]
}
```

### 8.6 操作紀錄內容

每筆：`{ "at", "device_id", "action", "item_id?", "item_name?", "result", "details?" }`。`action` 例如 `item.create`、`item.update`、`key.export_private`、`key.rotate`、`key.revoke`、`sync.conflict`、`sync.rollback_suspected`、`vault.password_change`、`vault.rekey_start`、`vault.rekey_commit`。

### 8.7 加密備份檔格式（可獨立解鎖）

備份檔必須在沒有任何其他資訊的情況下，只憑主密碼或復原金鑰就能解開。因此外層以明文保存解鎖所需的一切，內層才是資料。

```
偏移  長度        欄位
0     4           magic "HSMB"
4     1           format（1）
5     4           outer_len（big-endian；硬性上限 1 MiB，超過即拒絕，未檢查前不配置、不解析）
9     outer_len   outer_json（UTF-8 明文；解析時限制巢狀深度 ≤ 8）
9+L   ...         inner：一個 8.2 格式的加密檔案，kind = 5，file_id = 固定 UUID，
                  rev = 此備份的 UUID，parent_rev / parent_hash = 全零，merge_count = 0，trailer_hash = SHA-256("")，key_id = outer_json 中 keys 的 active 金鑰；
                  其 AAD = 227-byte 標頭 || SHA-256(outer_json bytes)（8.2、5.1.3 已納入此規則）
```

`outer_json`：

```json
{
  "format": 1,
  "vault_id": "...",
  "exported_at": "...",
  "exported_by_device": "...",
  "keys": [ { "key_id": "...", "gen": 1, "status": "active",
              "wrappings": { "password": { "kdf": {...}, "nonce": "...", "wrapped_vk": "..." },
                             "recovery": { "enabled": true, "hkdf_salt": "...", "nonce": "...", "wrapped_vk": "..." } },
              "vk_check": { "nonce": "...", "ciphertext": "..." } } ],
  "active_key": "...",
  "naming": { ... },
  "contents": { "items": 123, "hosts": 4, "log": true, "trash": false }
}
```

- `keys` 直接複製自匯出當時標頭中所有未清除的金鑰（備份內層只以 active 金鑰加密，但保留整個集合以便還原後仍能讀取未遷移的歷史），因此備份用的是**同一組金鑰、同一個主密碼與復原金鑰**；匯出時也可選擇以獨立的備份密碼另行包裝（`--backup-password`），此時每把金鑰的 `wrappings.password` 為以該密碼推導的 PK 包裝。
- **完整性**：inner 檔案的 AAD 除了自己的 227-byte 標頭外，再串接 `SHA-256(outer_json bytes)`。竄改外層任何 byte（例如改 KDF 參數、換成別的包裝）都會使 inner 驗證失敗。
- 還原時的順序：檔案總長度檢查 → `outer_len` 上限 → 解析外層 → KDF 參數上限 → 解開 VK → `vk_check` → inner 長度檢查 → 解密。
- inner 明文為 JSON：`{ "header_naming", "index", "hosts", "log?", "items": [ 每個 Item 的完整 JSON（8.3）], "trash?": [...] }`。
- 還原流程：解析外層 → 參數上限檢查 → 以主密碼或復原金鑰解開 VK → 驗證 `vk_check` → 解密 inner → 在目標位置建立新 Vault（新的 rev 鏈，同一 VK 或由使用者選擇產生新世代）。
- 驗收：FR-BAK-05 的「全新電腦、無快取、無 Google」情境為自動化測試。

---

## 9. 介面規格

### 9.1 桌面應用程式

#### 9.1.1 畫面清單

| 畫面 | 內容 | 優先 |
|------|------|:--:|
| 歡迎 / 初始化精靈 | 建立新 Vault 或連接既有 Vault；步驟：主密碼 → Google 授權 → Drive 資料夾與檔案名稱（預先填入預設值，可展開「進階」修改各檔名與 Item 檔名樣式，即時驗證並預覽實際檔名）→ 復原金鑰（顯示一次並要求確認）。連接既有 Vault 時可輸入資料夾路徑，或從「此帳號下所有 Vault」清單選擇 | P0 |
| 解鎖 | 輸入主密碼；提供「使用復原金鑰」入口 | P0 |
| 主視窗 | 左側分類（全部 / 帳密 / 金鑰 / 憑證 / 主機 / 垃圾桶）與標籤；中間列表（名稱、摘要、狀態、更新時間）；右側詳細面板；頂部搜尋框 | P0 |
| 帳密詳細 | 帳號、密碼（遮罩 + 顯示 + 複製）、網址（可開啟）、備註、標籤、到期日、版本列表、TOTP 碼（P1） | P0 |
| 金鑰詳細 | 演算法、指紋、公鑰（複製 / 另存）、狀態、版本列表、綁定主機；動作：匯出私鑰、安裝到 ~/.ssh、載入 ssh-agent（P1）、輪替、撤銷 | P0 |
| 憑證詳細 | Subject、SAN、到期日、憑證鏈；動作：匯出、更新 | P0 |
| 主機詳細 | 位址、環境、綁定金鑰清單（帳號、用途）；動作：綁定 / 解除、產生 authorized_keys | P0 |
| 新增 / 編輯對話框 | 依型別切換欄位；密碼產生器內嵌 | P0 |
| 金鑰產生對話框 | 名稱、演算法、用途、註解、passphrase | P0 |
| 匯入精靈 | 來源選擇（Chrome CSV / Chrome 設定檔 / 其他密碼管理器 / 私鑰檔 / ~/.ssh 掃描）→ 預覽與勾選 → 匯入報告 | P0（Chrome 設定檔與其他管理器為 P1） |
| 同步與狀態 | pending、衝突列表（逐欄位並排比較、選擇保留或逐欄挑選）、疑似回滾警示、隔離區（外洩輪替期間的未信任舊世代寫入：檢視、重新提交、丟棄）、標頭分支合併、到期與逾期輪替提醒、missing 項目、伺服器端待處理清單（removal_pending 綁定） | P0 |
| 設定 | 自動鎖定時間、剪貼簿清除秒數、Drive 資料夾資訊、Google 授權狀態與撤銷、變更主密碼、重新產生復原金鑰、語言、索引摘要雜湊與最後修訂時間（供跨裝置人工比對） | P0 |
| 設定 → 全面輪替 | 說明與主密碼變更的差異、外洩原因選擇與憑證替換、服務端輪替清單、啟動、狀態機目前狀態與進度（已遷移 / 總數）、續行、取消、裝置確認清單與移除裝置 | P0 |
| 設定 → Drive 命名 | 顯示目前資料夾路徑與各檔案名稱；修改資料夾路徑（搬移）與命名規則（重新命名所有檔案），顯示進度與失敗重試 | P1 |
| 操作紀錄 | 依時間、操作、Item 篩選 | P1 |
| 系統匣選單 | 快速搜尋、鎖定、開啟主視窗、結束 | P1 |

#### 9.1.2 行為規範

- 任何顯示或複製機密值的動作都寫入操作紀錄。
- 匯出私鑰一律以「另存新檔」對話框指定路徑；檔案依 5.3 在建立瞬間即為僅本人可讀，目標目錄權限不安全時**拒絕**並說明原因，不提供「仍要寫入」選項。
- 破壞性動作（撤銷、刪除、明文匯出、`--force` 匯出已撤銷私鑰）需二次確認，確認對話框需輸入 Item 名稱。
- 視窗內不得出現可被螢幕截圖工具自動擷取的明文機密，除非使用者主動點擊「顯示」。
- 未解決的衝突與逾期提醒以主視窗頂部橫幅提示，不彈出阻斷式對話框。

### 9.2 CLI

#### 9.2.1 基本

- 執行檔名稱：`hsm`（待決議 Q-03）。
- CLI 為 Phase 2 交付（第 12 章）；本節為完整目標規格。
- 全域選項：`--vault NAME`、`--offline`、`--json`、`--quiet`、`--config PATH`。
- 環境變數：`HSM_VAULT`、`HSM_PASSWORD_FILE`（從檔案讀主密碼，供腳本使用，文件須警告風險）。
- 輸出：預設人類可讀，`--json` 輸出 JSON 供腳本使用。
- Item 以名稱參照；名稱在同一 Vault 內唯一。支援 `--id` 以 ID 參照。

#### 9.2.2 指令

```
# 初始化與帳號
hsm init [--drive-folder PATH] [--existing | --existing-from-list]
         [--name-header NAME] [--name-index NAME] [--name-log NAME] [--name-hosts NAME]
         [--name-items-folder NAME] [--item-pattern PATTERN]     # 皆有預設值，見 7.3
         [--no-recovery-key] [--scope file|full]
hsm auth status
hsm auth revoke
hsm unlock                         # 啟動或喚醒 agent（P1）
hsm lock
hsm vault list
hsm vault info                     # 含資料夾路徑與目前命名規則
hsm vault rename-files [--name-header NAME] [...] [--item-pattern PATTERN]   # P1
hsm vault move --drive-folder PATH # P1
hsm vault change-password
hsm vault recovery regenerate
hsm vault recovery unlock          # 以復原金鑰解鎖並重設主密碼
hsm vault rekey --reason password|recovery|unknown|routine [--new-recovery-key]   # 全面輪替（新世代）
hsm vault rekey --status | --resume | --abort | --finalize   # 續行 / 取消（僅 routine 可，進入 reverting）/ 嘗試 settled → stable
hsm vault rekey --ack                # 本裝置寫入此輪替操作的交接確認（pending 已清、已切換寫入世代）
hsm vault devices list | remove <device_id>

# 同步與狀態
hsm sync [--rebuild-index]
hsm status                         # pending、衝突、到期、逾期輪替、missing
hsm resolve <name> --keep local|remote|<version>
hsm verify
hsm gc                             # 第一版只清理本機快取與垃圾桶 Item 的內容，不刪除 Drive 修訂

# 帳密與通用項目
hsm list [--type TYPE] [--tag TAG] [--status S]
hsm search <query>
hsm add <name> --type credential [--username U] [--generate [--length N] | --stdin | --password-file PATH] [--url URL] [--tag T ...]
                                   # 未給密碼來源時以隱藏輸入提示；沒有 --password VALUE 這種參數
hsm add <name> --type generic [--from-file PATH | --fields-json PATH | --field KEY ...]
                                   # --field KEY 只給欄位名稱，值以隱藏輸入提示
hsm show <name> [--version N]      # 顯示非機密欄位與版本列表
hsm get <name> [--field password] [--version N] [--copy | --stdout]
hsm totp <name> [--copy]
hsm edit <name> [--username U] [--generate | --stdin | --password-file PATH | --prompt-password] [--url URL] [--notes-file PATH] [--expires DATE]
hsm tag <name> [--add T] [--remove T]
hsm rename <name> <new-name>
hsm versions <name>
hsm rm <name>
hsm restore <name>                 # 從垃圾桶還原
hsm generate-password [--length N] [--no-symbols] [--passphrase [--words N]]

# 伺服器金鑰
hsm key generate <name> [--algorithm ed25519|rsa-4096|rsa-2048|ecdsa-p256] [--purpose ssh|tls|signing|other] [--comment TEXT] [--passphrase] [--tag T ...]
hsm key import <name> --private-key PATH [--public-key PATH] [--purpose ...]
hsm key import --scan [--dir ~/.ssh]
hsm key import-public <name> --public-key PATH
hsm key show <name>                # metadata、公鑰、指紋、版本、綁定主機
hsm key public <name> [--format openssh|pem] [--out PATH | --copy]
hsm key private <name> [--format openssh|pkcs8|pkcs1] [--out PATH | --stdout] [--version N] [--force] [--strip-passphrase]
                                   # 預設只匯出 active 版本；retired 需 --version 並警告；revoked 需 --force
hsm key install <name> [--dir ~/.ssh] [--ssh-config --host HOSTNAME]
hsm key ssh-add <name> [--lifetime 1h]
hsm key rotate <name> [--note TEXT]
hsm key revoke <name> [--version N | --all] --reason TEXT   # 本機標記；受影響的 installed 綁定變為 removal_pending
hsm key find --fingerprint SHA256:...

# TLS 憑證
hsm cert import <name> --cert PATH --key PATH [--chain PATH]
hsm cert show <name>
hsm cert export <name> [--cert-out PATH] [--key-out PATH] [--chain-out PATH]
hsm cert renew <name> --cert PATH --key PATH [--chain PATH]
hsm cert expiring [--within-days N]

# 主機
hsm host list [--env ENV]
hsm host add <name> --address ADDR [--port N] [--env ENV] [--host-key-fingerprint FP]
hsm host show <name>
hsm host edit <name> [...]
hsm host rm <name>
hsm host bind <host> <key> [--version N] --remote-user USER [--usage TEXT]
hsm host unbind <host> <key> [--version N] --remote-user USER
hsm host deploy-state <host> <key> --version N --remote-user USER --set declared|installed|removal_pending|removed
hsm host pending                   # 所有 removal_pending 綁定（伺服器端待處理清單）
hsm host authorized-keys <host> [--out PATH]   # 只含 active 版本，retired/revoked 仍 installed 者以註解警告

# 備份與匯入匯出
hsm export --encrypted --out PATH [--backup-password] [--include-log] [--include-trash]
hsm export --plain --out PATH      # 二次確認
hsm restore --from PATH [--to-drive-folder PATH | --to-local DIR]   # 可在全新電腦、無 Google 授權下執行
hsm import --from PATH [--format hsm|chrome-csv|bitwarden|1password|keepass|firefox-csv|edge-csv] [--on-duplicate version|skip|new] [--dry-run]
hsm import --chrome [--profile NAME] [--on-duplicate ...] [--dry-run]   # 直接讀取 Chrome 設定檔（P1）

# 紀錄
hsm log [--since T] [--action A] [--item NAME] [--limit N]
```

#### 9.2.3 行為規範

- `get`、`key private`、`cert export` 預設寫檔或複製到剪貼簿；輸出到 stdout 需明確 `--stdout`，且 stdout 為終端時再次確認。
- **沒有任何參數接受命令列明文機密**（密碼、passphrase、TOTP 秘密、欄位值）。機密只能來自隱藏輸入提示、`--stdin`、`--password-file PATH`（檔案須為僅本人可讀，否則拒絕）或 `--generate`。這避免機密進入 shell history、程序參數列表與行程監控工具。
- 所有需要主密碼的指令在非 TTY 且無 agent、無 `HSM_PASSWORD_FILE` 時，以結束碼 3 失敗。
- 任何會寫入含明文機密檔案的指令，都以 5.3 的「建立時即為僅本人可讀」方式建立檔案。
- 結束碼：0 成功、1 一般錯誤、2 參數錯誤、3 未解鎖或主密碼錯誤、4 Google 授權失效、5 網路或 Drive 錯誤、6 存在衝突需處理。

---

## 10. 非功能需求

| 類別 | 需求 |
|------|------|
| 效能 | 解鎖（Argon2id 推導）在一般筆電上 1–2 秒；已解鎖且索引已快取時，`get` 在 200 ms 內完成（不含 Drive 往返）；完整同步 500 個 Item 在 60 秒內完成。 |
| 容量 | 目標：單一 Vault 2,000 個 Item、每 Item 10 版本、操作紀錄 5,000 筆；單次 Chrome 匯入 2,000 筆在 30 秒內完成預覽。 |
| 可靠性 | 任何時刻程序被中斷（Ctrl-C、斷電、斷線）都不得造成已確認寫入的資料遺失；寫入快取採「寫暫存檔再 rename」；Drive 上的資料從不就地覆寫；全面輪替在任一步中斷後重啟皆可續行或明確回復。 |
| 相容性 | Linux x86_64 / arm64、macOS（Intel / Apple Silicon）、Windows 10+。桌面應用程式以各平台原生安裝檔發布（`.dmg`、`.msi`、`.AppImage` / `.deb`）。 |
| 互通性 | 匯出的 OpenSSH 金鑰需能被 `ssh-keygen -y`、`ssh-add` 正確讀取；PEM 需能被 `openssl` 讀取。 |
| 測試 | 加密層與檔案格式單元測試覆蓋率 ≥ 90%；金鑰匯入匯出與 OpenSSH / OpenSSL 做互通測試；下列**復原導向驗收測試**為 Phase 1a 的結束條件，比覆蓋率更優先：(a) 變更主密碼與全面輪替的每一個步驟之後強制中斷，重啟後 Vault 必須仍可用主密碼與復原金鑰開啟且資料完整；(b) 兩台模擬裝置離線各自修改同一 Item、同一 hosts、同時改密碼，上線後必須偵測分支且雙方資料皆保留；(c) 清空本機環境、不給 Google 授權，只用備份檔加主密碼或復原金鑰還原，所有 Item 與金鑰可讀；(d) 竄改檔案長度欄位、KDF 參數、parent_rev、標頭 MAC 保護的欄位與回滾整份資料，工具必須拒絕並給出對應錯誤碼；(e) 寫入私鑰與明文匯出時在 Windows 與 Unix 驗證檔案建立瞬間即為僅本人可讀；(f) 偽造中間修訂（合法 appProperties、偽造或替換的內容）使新 head 看似信任點的後代，必須被 HASH_MISMATCH 或 MISSING_PARENT 擋下；(g) 舊世代裝置在輪替進入 `settled` 之後才上傳延遲的修訂，資料必須被遷移且不遺失；(h) 取消輪替途中斷電，重啟後兩把金鑰皆可用且能續行或完成反向遷移；(i) 離線裝置在輪替完成並清除舊世代後重新連線，其 pending 修訂必須能以本機舊標頭快取與舊憑證解出並重新上傳；(j) 假子修訂隱藏撤銷分支：T 分出 active 與 revoked 兩個合法修訂，加入一個宣稱以 revoked 為父的無效檔案，同步後必須仍回報分支衝突；(k) 舊 VK 簽署降級標頭：以已清除或 retired 世代的 VK 製作 parent 指向目前信任標頭、top 較低的 stable 標頭，必須被 HEADER_TRANSITION_DENIED 或 HEADER_MAC_INVALID 擋下；(l) 完成輪替、清除舊世代、再改一次主密碼後，全新電腦只用最新主密碼同步，必須能驗證完整歷史並採用最新資料；(m) 前進輪替進入 settled 後，某台裝置尚未寫入此 op 的 forward 確認（即使它曾看過新世代，或對先前被取消的 op 寫過確認），必須無法進入 stable(m; ∅) 清除舊世代；(n) 取消後再次輪替，目標世代必須為歷來最大世代加一，且所有歷史修訂的世代號仍能唯一對應金鑰；(o) 一般非合併檔案與備份檔寫入後立即解密成功（含空 trailer 雜湊）；(p) 取消輪替完成後鎖定、重啟，並以全新裝置用目前主密碼解鎖與同步，標頭 MAC 與狀態皆驗證通過；(q) 離線裝置跨過另一台完成的完整輪替（stable → migrating → settled → stable）後重新同步，路徑驗證通過且資料可用；(r) 兩台裝置以相同 parent 各自寫入後，必須進入衝突處理而不是回滾警告；(s) 保留本機快取、在 Drive 刪除某個修訂或整個檔案，同步必須回報 REMOTE_DELETED 或 ROLLBACK_SUSPECTED 並能從本機快取重新上傳；(t) 外洩輪替進入 settled 後，以已外洩的舊 VK 製作通過 AES-GCM 驗證的修訂，分別把 parent 指向新世代 head（必須 GENERATION_REGRESSION）與指向舊世代修訂（必須隔離、不遷移、不出現在衝突處理），且隔離內容只能由使用者明確重新提交；(u) 兩台裝置同時啟動輪替並各自以不同金鑰寫入後，合併標頭必須保留兩把金鑰，兩邊資料都能解開，遷移與衝突處理完成後所有資料都在 active 金鑰下；(v) 連續取消兩次、改一次密碼、重新產生復原金鑰後，全新裝置以目前主密碼與以復原金鑰各自解鎖，必須解出整個金鑰集合並能同步與讀取所有歷史。同步引擎以本機資料夾儲存層做整合測試。 |
| 相依性 | 不依賴專案方任何線上服務；除 Google Drive API 外不連線其他主機（含不做自動更新檢查，除非使用者明確啟用）。 |
| 可觀測性 | `--verbose` 輸出 Drive API 呼叫摘要（方法、檔案 ID、狀態碼），永不輸出內容或 token。 |
| 國際化 | CLI 訊息繁體中文與英文雙語，預設依系統語系，`HSM_LANG` 可覆寫。 |
| 授權 | MIT。相依套件需為相容授權。 |

---

## 11. 建議技術選型

以下為建議，供第 13 章決議；未定案前不應開始實作。

| 層級 | 建議 | 理由 |
|------|------|------|
| 語言 | Go | 單一靜態二進位檔，三平台交叉編譯簡單；標準庫 `crypto` 完整；`golang.org/x/crypto/ssh` 直接處理 OpenSSH 格式與 ssh-agent 協定；`argon2`、`hkdf` 皆有官方套件。 |
| Google Drive | 官方 `google.golang.org/api/drive/v3` 與 `golang.org/x/oauth2` | 官方維護，支援 PKCE 與 loopback 流程。 |
| 金鑰圈 | `zalando/go-keyring` 或等效 | 跨三平台。 |
| 壓縮 | `klauspost/compress/zstd` | 純 Go 實作。 |
| 桌面 GUI | Wails v2（Go 後端 + 網頁技術前端，使用系統 WebView） | 與 Core 同語言、單一執行檔、安裝檔小於 Electron；三平台皆支援系統匣。 |
| 前端 | TypeScript + Svelte 或 React | 由團隊熟悉度決定。 |
| CLI 框架 | `spf13/cobra` | 慣例成熟，自動產生說明與補全。 |
| 剪貼簿 | Wails 內建 API（GUI）、`atotto/clipboard`（CLI） | 跨平台。 |
| Chrome 匯入 | `modernc.org/sqlite`（純 Go，讀 `Login Data`）、`golang.org/x/crypto/pbkdf2`（macOS / Linux 金鑰推導）、DPAPI 透過 `golang.org/x/sys/windows` | 不需 cgo。 |

替代方案：Rust + Tauri 在記憶體安全與 zeroize 上更嚴謹、安裝檔更小，但 Google API 與 ssh-agent 生態較薄；若團隊熟悉 Rust 可考慮。Electron 不建議，安裝檔過大且記憶體佔用高。

---

## 12. 開發階段規劃

第一版的核心問題是「電腦壞掉後，密碼與金鑰拿不拿得回來」。因此 Phase 1 先做金鑰結構、獨立備份還原、中斷復原與同步衝突，並以第 10 章的復原導向驗收測試作為結束條件；自動安裝 SSH、agent、便利功能之後再加。

| 階段 | 目標 | 主要交付 | 對應需求 |
|------|------|----------|----------|
| **Phase 0：確認** | 規格審閱、技術選型定案、建立 Google Cloud OAuth client、repo 骨架 | 本文件 1.0、CI 流程、專案結構、儲存層介面定義 | 第 13 章全部 |
| **Phase 1a：核心與復原** | Core 程式庫在本機資料夾儲存層與 Google Drive 上通過全部復原導向驗收測試 | 三層金鑰結構、標頭 MAC 與雜湊鏈、8.2 檔案格式與硬性上限、不可覆寫修訂與經驗證的分支偵測、回滾防護、衝突合併修訂、變更主密碼、復原金鑰、全面輪替狀態機（含取消、延遲上傳、舊裝置重新加入）、可獨立解鎖的備份與還原、Google 授權與資料夾建立、命名規則、帳密 / 通用 / 金鑰 / 憑證 / hosts 的資料模型與讀寫、金鑰產生 / 匯入 / 匯出（含每版本狀態與部署狀態）、Chrome CSV 匯入引擎、操作紀錄、Windows ACL 與 Unix 權限、驗收測試 (a)–(v) | 5.1、5.3、8、4.5、4.6 的 P0 |
| **Phase 1b：桌面介面** | 桌面應用程式在單一裝置可完整使用，第二裝置可連接同步 | 9.1.1 所有 P0 畫面、5.6 GUI 強化、自動鎖定、剪貼簿清除、拖放匯入、衝突並排處理、伺服器端待處理清單 | 其餘 P0 |
| **Phase 2：日常順手** | CLI、自動安裝、提醒、歷史清理設計 | CLI 與 agent（5.6）、`key install`、ssh-add、經驗證檢查點與歷史清理（FR-SYNC-19）、Chrome 設定檔直接讀取、其他密碼管理器匯入、TOTP、到期與輪替提醒、指紋反查、verify、垃圾桶、Drive 狀態、系統匣、操作紀錄畫面 | 所有 P1 |
| **Phase 3：擴充** | 進階整合 | 多 Vault、背景同步、Host 掃描與公鑰推送、附件型 Item、PuTTY 匯出、其他 Chromium 瀏覽器匯入、全域快捷鍵、其他儲存後端、檔案大小填充 | P2 |

每階段結束條件：對應需求的自動化測試全部通過、第 5 章安全檢查逐項確認、使用文件更新。Phase 1a 另外要求驗收測試 (a)–(v) 全數通過後才開始 Phase 1b。

---

## 13. 待決議事項

| 編號 | 問題 | 建議 | 影響 |
|------|------|------|------|
| Q-01 | 實作語言 | Go（見第 11 章） | 全部 |
| Q-02 | OAuth client 由誰提供：專案內建一組（需 Google 驗證流程，否則顯示「未經驗證的應用程式」警告且測試使用者上限 100 人），或由使用者自行建立 GCP 專案並提供 client ID | 第一版由使用者自行建立並以 `hsm init --client-id ... --client-secret ...` 提供；文件附步驟。若日後公開發布再申請驗證 | FR-DRIVE-01、7.1 |
| Q-03 | 執行檔名稱：`hsm` 易與硬體安全模組混淆 | 改為 `holisec` 或 `hsecret` | 第 9 章、6.3 目錄名 |
| Q-04 | 預設 OAuth 範圍 `drive.file` 造成「看不到使用者手動建立的同名資料夾」 | 維持 `drive.file` 為預設，提供 `--scope full` | 7.1、7.2 |
| Q-05 | Drive 為唯一真實來源，或本機為主、Drive 為備份 | Drive 為真實來源（本文件採此設計），以支援多裝置 | 4.5 |
| Q-06 | 介面優先順序：GUI 與 CLI 是否都在 Phase 1 | GUI 為 Phase 1，CLI 為 Phase 2（本文件採此安排）；若使用者日常以終端機操作金鑰為主，可對調 | 4.8、第 12 章 |
| Q-12 | Chrome 設定檔直接讀取在 macOS 會觸發 Keychain 授權提示、在新版 Windows Chrome 幾乎不可行，是否值得投入 | 維持 P1；CSV 途徑為主要支援途徑 | 4.6.1 |
| Q-13 | Chrome 匯入的重複項目預設處理方式 | 密碼不同時為既有 Item 新增版本 | FR-CHROME-04 |
| Q-07 | 復原金鑰是否預設啟用 | 預設啟用，可用 `--no-recovery-key` 略過 | FR-INIT-04 |
| Q-08 | Argon2id 參數（記憶體 256 MB 在低階裝置可能過慢） | 預設 256 MB，`init` 時可 `--kdf-memory` 調整，參數存於標頭 | 5.1 |
| Q-09 | 版本保留數量與垃圾桶保留期 | 10 版、30 天 | FR-CRED-02、FR-KEY-25 |
| Q-10 | 操作紀錄是否同步到 Drive，或只留本機 | 同步（可看到各裝置紀錄） | FR-LOG-02 |
| Q-11 | 是否需要支援 Google Workspace 帳號的共用雲端硬碟（Shared Drive） | 第一版不支援，只用 My Drive | 7.2 |
| Q-14 | 預設名稱：資料夾 `HolienSecretsManager`、檔案 `vault.hsm` / `index.enc` / `log.enc` / `hosts.enc` / `items/{id}.enc` 是否合適 | 採用上述預設；若 Q-03 改了執行檔名稱，副檔名 `.hsm` 一併更名 | 7.3 |
| Q-15 | 命名規則是否允許每台裝置各自不同（只存本機設定） | 否，存於 Vault 標頭、所有裝置一致，避免同一 Vault 在不同裝置產生不同檔名 | FR-DRIVE-13 |
| Q-16 | 第一版不清理修訂，Drive 空間成長是否可接受（估計：每次 Item 寫入數 KB，索引每次重寫 ≈ Item 數 × 100 bytes） | 可接受；Phase 2 以檢查點設計清理 | FR-SYNC-12/19 |
| Q-17 | 全新裝置首次同步的回滾辨識：是否要在標頭加入「索引摘要雜湊 + 簽章」等機制 | 第一版不做，以人工比對緩解（FR-SYNC-11）；列為 Phase 3 研究 | FR-SYNC-10/11 |
| Q-18 | 備份檔是否預設包含操作紀錄 | 預設不含（紀錄含 Item 名稱與時間，但不含機密值；由使用者決定） | FR-BAK-07 |
| Q-19 | 全面輪替是否也要重新產生復原金鑰 | 預設詢問；懷疑復原金鑰外洩時必須 | 5.1.6 |
| Q-20 | 取消輪替後被取消的世代以 retired 保留到下次前進輪替，是否需要「單獨清除」操作 | 第一版不提供；需要時再做一次前進輪替即可 | 5.1.6 |
| Q-21 | routine 輪替是否也應改為隔離（與外洩輪替相同），犧牲延遲遷移的便利換取一致性 | 維持區分；routine 的延遲遷移結果以分支呈現、使用者可見 | FR-SYNC-21 |

---

## 附錄 A：假設清單

本文件在下列假設下撰寫，任一假設不成立時需修訂對應章節：

1. 使用者為單一個人；不需要與他人共享任何 Item。
2. 使用者擁有個人 Google 帳號，且願意在 Google Cloud Console 建立 OAuth client（或接受專案內建 client 的未驗證警告）。
3. 「伺服器 private + public key」主要指 SSH 金鑰與 TLS 私鑰；PGP 與程式碼簽章金鑰可用 `purpose=signing` 保存，但不提供專屬功能。
4. 系統不負責簽發憑證，只保管。
5. 使用者的電腦可視為可信環境；本工具不防禦本機惡意軟體。
6. 使用者在多台裝置上使用同一 Google 帳號與同一主密碼。
7. 產品為單機桌面應用程式，第一版主要介面為圖形介面；CLI 為輔助。
8. 文件與介面以繁體中文為主、英文為輔。
9. 使用者的 Chrome 資料為個人所有，匯入只讀取本機設定檔或使用者自行匯出的 CSV，不接觸 Google 帳號的密碼同步服務。

## 附錄 B：錯誤與結束碼

| 結束碼 | 錯誤代碼 | 說明 |
|:--:|------|------|
| 1 | `INTERNAL` | 未預期錯誤 |
| 1 | `NOT_FOUND` | Item / Host 不存在 |
| 1 | `CONFLICT_NAME` | 名稱重複 |
| 1 | `KEY_REVOKED` | 金鑰已撤銷，需 `--force` |
| 1 | `INVALID_KEY_MATERIAL` | 私鑰 / 憑證無法解析或不匹配 |
| 1 | `CORRUPTED` | 檔案格式錯誤、長度不符、GCM 驗證失敗或標頭與內容不一致 |
| 1 | `CORRUPTED_CHAIN` | 所有候選 head 皆無法驗證，或鏈上有環 |
| 1 | `MISSING_PARENT` | 已驗證的修訂引用不存在的父修訂 |
| 1 | `HASH_MISMATCH` | 父修訂或信任點的實際雜湊與記錄不符，或同一 rev 有不同內容 |
| 1 | `HEADER_MAC_INVALID` | 標頭 MAC 驗證失敗，或 mac.key_id 不是 top |
| 1 | `HEADER_TRANSITION_DENIED` | 標頭狀態轉換不在授權表內，或 MAC 世代不符該轉換 |
| 1 | `UNVERIFIED_CANDIDATE` | 候選修訂既無法解密也沒有已驗證的子修訂，已捨棄（警告） |
| 6 | `STALE_GENERATION_HEAD` | 唯一的 head 只有關係驗證（其世代已清除且無後代），需人工處理 |
| 5 | `REMOTE_DELETED` | 本機有此檔案的快取，但 Drive 上已無任何修訂 |
| 1 | `FOREIGN_CHAIN` | head 的鏈與本機信任的鏈沒有共同祖先，已捨棄 |
| 6 | `CONFLICT_SENSITIVE_OP` | 衝突未解決，敏感操作被拒絕 |
| 6 | `STALE_GENERATION_PENDING` | 本機 pending 修訂的世代已被清除，無法上傳 |
| 1 | `UNSUPPORTED_FORMAT` | 檔案格式版本高於本程式支援 |
| 3 | `KDF_PARAMS_UNSAFE` | 標頭或備份檔的 KDF 參數超過硬性上限，拒絕執行 |
| 6 | `ROLLBACK_SUSPECTED` | Drive 上的修訂鏈不包含本機信任點 |
| 6 | `HEADER_BRANCHED` | 標頭有多個 head，需使用者選擇 |
| 6 | `REKEY_IN_PROGRESS` | 全面輪替進行中，需先續行或取消 |
| 6 | `STALE_GENERATION` | 修訂的 key_id 不在本機金鑰集合中（已清除或未知），或本機 active 金鑰已非標頭指定者 |
| 1 | `GENERATION_REGRESSION` | 子修訂的金鑰世代低於父修訂，已隔離 |
| 1 | `UNSNAPSHOTTED_MIGRATION` | 跨世代遷移修訂的父不在輪替快照或允許遷移清單中，已隔離 |
| 6 | `QUARANTINED_WRITES` | 外洩輪替期間出現未信任的舊世代修訂，待使用者處理 |
| 1 | `PERMISSION_UNSAFE` | 目標檔案或目錄無法以僅本人可讀方式建立 |
| 1 | `UNSAFE_PATH` | 目標目錄權限不安全 |
| 1 | `IMPORT_PARSE` | 匯入檔案格式無法解析 |
| 1 | `CHROME_PROFILE_NOT_FOUND` | 找不到 Chrome 設定檔 |
| 1 | `CHROME_KEY_UNAVAILABLE` | 無法取得 Chrome 密碼加密金鑰，請改用 CSV 途徑 |
| 1 | `CHROME_APP_BOUND` | Chrome 使用 App-Bound Encryption，無法直接讀取，請改用 CSV 途徑 |
| 2 | `INVALID_ARGS` | 參數錯誤 |
| 3 | `LOCKED` | 需要主密碼但無法取得 |
| 3 | `BAD_PASSWORD` | 主密碼或復原金鑰錯誤 |
| 3 | `NOT_INITIALIZED` | 尚未執行 `init` |
| 4 | `AUTH_REQUIRED` | Google 授權不存在或已失效 |
| 4 | `AUTH_SCOPE` | 目前授權範圍不足以執行此操作 |
| 5 | `DRIVE_ERROR` | Drive API 錯誤（重試後仍失敗） |
| 5 | `DRIVE_QUOTA` | Drive 空間不足 |
| 5 | `OFFLINE` | 需要網路但不可用，且無 `--offline` |
| 5 | `FOLDER_AMBIGUOUS` | 資料夾路徑對應到多個同名資料夾 |
| 5 | `MULTIPLE_VAULTS` | 資料夾內有多個不同 vault_id 的 Vault |
| 2 | `INVALID_NAMING` | 檔名或 Item 檔名樣式不符合 FR-DRIVE-15 規則 |
| 5 | `RENAME_INCOMPLETE` | 命名規則更新未全部完成，可重試 |
| 6 | `CONFLICT_PENDING` | 存在未解決的同步衝突 |
