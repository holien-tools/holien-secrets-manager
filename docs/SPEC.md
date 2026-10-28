# Holien Secrets Manager 規格文件

| 項目 | 內容 |
|------|------|
| 文件版本 | 0.4.0（草稿） |
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
| Master Password（主密碼） | 使用者記憶的唯一密碼，用於推導 KEK。遺失即無法解密資料。 |
| Recovery Key（復原金鑰） | 建立 Vault 時產生的隨機金鑰，可在遺忘主密碼時解鎖 Vault。選用。 |
| KEK | Key Encryption Key，由主密碼推導的 256-bit 金鑰，用於包裝 DEK。 |
| DEK | Data Encryption Key，每個 Item 檔案一把的隨機 256-bit 金鑰，用於實際加密內容。 |
| Lock / Unlock | 本機尚未載入 KEK 的狀態為 locked；輸入主密碼推導出 KEK 後為 unlocked。 |
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
| FR-INIT-06 | P1 | 解鎖狀態維持：啟動本機 agent 程序在記憶體中保存 KEK，逾時（預設 15 分鐘，可設定）或執行 `lock` 後清除，期間指令不需重複輸入主密碼。 |
| FR-INIT-07 | P1 | 變更主密碼：重新推導 KEK、重新包裝所有 Item 的 DEK、更新 Vault 標頭並上傳。過程中任一步失敗須可回復到原狀態。 |
| FR-INIT-08 | P1 | 以復原金鑰解鎖並重設主密碼。 |
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
| FR-DRIVE-06 | P0 | OAuth refresh token 存於作業系統金鑰圈（macOS Keychain / Linux Secret Service / Windows Credential Manager）；無金鑰圈時以 KEK 加密後存於本機設定目錄。 |
| FR-DRIVE-07 | P0 | 處理 Drive API 的速率限制與暫時性錯誤：指數退避重試（最多 5 次），超過後回報明確錯誤並保留本機變更待下次同步。 |
| FR-DRIVE-08 | P0 | 撤銷授權：`auth revoke` 向 Google 撤銷 token 並清除本機 token；不刪除 Drive 上的檔案。 |
| FR-DRIVE-09 | P1 | 顯示 Drive 資料夾狀態：Item 數、總大小、最後同步時間、Drive 剩餘空間。 |
| FR-DRIVE-10 | P1 | 以 Drive 的檔案修訂（revision）作為額外安全網；工具不主動刪除修訂。 |
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
| FR-KEY-11 | P0 | 私鑰匯出預設寫入檔案，權限 `0600`；要輸出到 stdout 需明確加 `--stdout`，且偵測到 stdout 為終端時再次確認。 |
| FR-KEY-12 | P0 | 私鑰可以 OpenSSH、PEM（PKCS#1 / PKCS#8）格式匯出；格式轉換於記憶體內完成。 |
| FR-KEY-13 | P0 | 公鑰可以 OpenSSH `authorized_keys` 單行格式、PEM 格式輸出。 |
| FR-KEY-14 | P0 | `key install`：將私鑰與公鑰寫入 `~/.ssh/<name>` 與 `~/.ssh/<name>.pub`，並可選擇在 `~/.ssh/config` 追加對應 Host 區塊。 |
| FR-KEY-15 | P1 | `key ssh-add`：直接將私鑰載入執行中的 ssh-agent，不寫入磁碟；可設定存活時間。 |
| FR-KEY-16 | P1 | 以指紋反查金鑰（用於從伺服器 `authorized_keys` 盤點）。 |
| FR-KEY-17 | P2 | 支援 PuTTY `.ppk` 格式匯出。 |

#### 4.4.3 生命週期

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-20 | P0 | 金鑰狀態：`active`、`retired`（已輪替，仍可讀取）、`revoked`（已撤銷，預設不可匯出私鑰）。 |
| FR-KEY-21 | P0 | 輪替（rotate）：以相同演算法與設定產生新金鑰版本，舊版本自動轉為 `retired`。新舊並存，供使用者完成伺服器端更換；`key show` 列出所有版本與各自指紋。 |
| FR-KEY-22 | P0 | 撤銷（revoke）：需填寫原因；撤銷後私鑰預設不可匯出，需加 `--force` 並再次確認。 |
| FR-KEY-23 | P1 | 可設定金鑰到期日；TLS 憑證自動以憑證到期日為準。`status` 指令列出 30 / 14 / 7 天內到期項目。 |
| FR-KEY-24 | P1 | 可設定輪替政策（例如每 90 天），`status` 指令列出逾期未輪替的金鑰；不自動執行輪替。 |
| FR-KEY-25 | P1 | 刪除金鑰採軟刪除，進入垃圾桶，保留期（預設 30 天）後由 `gc` 指令永久清除。 |

#### 4.4.4 主機綁定

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-30 | P0 | 建立 Host 紀錄：名稱、位址（IP 或 FQDN）、SSH 連接埠、環境（`prod`/`staging`/`dev`/`other`）、備註、標籤。 |
| FR-KEY-31 | P0 | 一把金鑰可綁定多台 Host，一台 Host 可綁定多把金鑰；綁定時記錄伺服器端帳號（例如 `root`、`deploy`）與用途。 |
| FR-KEY-32 | P0 | 可列出某台 Host 所有綁定的金鑰，及某把金鑰綁定的所有 Host。 |
| FR-KEY-33 | P0 | `host authorized-keys`：輸出該 Host 所有 active 金鑰的 `authorized_keys` 內容。 |
| FR-KEY-34 | P1 | 可記錄 Host 的 SSH host key 指紋；`key install` 寫入 `~/.ssh/config` 時一併產生 `known_hosts` 行。 |
| FR-KEY-35 | P2 | 透過 SSH 連線到 Host 讀取 `authorized_keys`，與紀錄比對並回報差異（唯讀）。 |
| FR-KEY-36 | P2 | 透過 SSH 將公鑰推送到 Host 的 `authorized_keys`（需二次確認）。 |

### 4.5 同步與離線（SYNC）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-SYNC-01 | P0 | Google Drive 為資料的唯一真實來源（source of truth）；本機保有完整的加密快取，內容與 Drive 上的密文相同。 |
| FR-SYNC-02 | P0 | 寫入操作：先寫本機快取並標記 `pending`，再上傳；上傳成功後清除標記。任何時刻中斷都不會造成資料遺失。 |
| FR-SYNC-03 | P0 | 讀取操作：預設先同步索引（輕量），再讀取本機快取；`--offline` 時完全不連網。 |
| FR-SYNC-04 | P0 | 每個 Item 為獨立檔案，衝突範圍限縮在單一 Item。 |
| FR-SYNC-05 | P0 | 衝突偵測：上傳前比對 Drive 上該檔案的 `headRevisionId` 與本機記錄的已知修訂；不一致即為衝突。 |
| FR-SYNC-06 | P0 | 衝突處理：下載遠端版本，將遠端與本機的變更都保留為版本，以 `updated_at` 較新者為目前版本，並在 Item 上標記 `conflict`；`status` 指令列出待處理衝突，`resolve` 指令由使用者選擇保留哪個版本。 |
| FR-SYNC-07 | P0 | 索引檔（index）可由全部 Item 檔案重建（`sync --rebuild-index`），索引損毀不會造成資料遺失。 |
| FR-SYNC-08 | P1 | 顯示同步狀態：pending 上傳數、最後同步時間、衝突數。 |
| FR-SYNC-09 | P2 | 背景同步：agent 程序定期（預設 5 分鐘）同步索引並上傳 pending 項目。 |

### 4.6 備份與匯入匯出（BACKUP）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-BAK-01 | P0 | `export --encrypted`：將整個 Vault（標頭、索引、所有 Item）打包為單一加密檔案（格式同第 8 章，以同一 KEK 加密），可直接以 `import` 還原到新的 Drive 資料夾。 |
| FR-BAK-02 | P0 | `export --plain`：匯出為明文 JSON，需二次確認並顯示警告；輸出檔權限 `0600`。 |
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
| FR-LOG-02 | P0 | 操作紀錄本身以 KEK 加密存於本機，並作為一個特殊 Item 同步到 Drive，使各裝置可看到彼此的紀錄。 |
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
  主密碼（使用者記憶）                 復原金鑰（隨機 256-bit，選用）
         │                                    │
         ▼ Argon2id（salt 存於 Vault 標頭）     ▼ HKDF
       KEK（256-bit，僅存在記憶體）  ◄── wrap ── RK-wrapped KEK（存於 Vault 標頭）
         │
         │ wrap / unwrap（AES-256-GCM）
   ┌─────┼──────────────┬──────────────────┐
   ▼     ▼              ▼                  ▼
 DEK(index)   DEK(item A)    DEK(item B)    DEK(log)     ← 每個檔案一把，隨機 256-bit
   │           │              │               │
   ▼ AES-256-GCM              ▼               ▼
 index.enc   items/<id>.enc  items/<id>.enc  log.enc     ← 上傳到 Google Drive 的檔案
```

- **KEK 推導**：Argon2id，預設參數記憶體 256 MB、迭代 3、並行 4；參數與 32-byte salt 存於 Vault 標頭，允許日後調高。
- **每個檔案一把 DEK**：256-bit CSPRNG 產生，以 KEK 用 AES-256-GCM 包裝後存於該檔案標頭。
- **內容加密**：AES-256-GCM，96-bit 隨機 nonce，每次寫入重新產生。
- **AAD**：包含檔案格式版本、Vault ID、檔案 ID、檔案類型，防止密文被搬移到其他檔案或其他 Vault。
- **KEK 驗證**：Vault 標頭存一段以 KEK 加密的固定內容，解鎖時先解此段以判斷主密碼是否正確，不需下載任何 Item。
- **復原金鑰**：若啟用，KEK 另以復原金鑰經 HKDF 推導的金鑰包裝後存於 Vault 標頭。變更主密碼不影響復原金鑰；重新產生復原金鑰則重新包裝。

### 5.2 Google Drive 上可見的資訊

| Google 可見 | Google 不可見 |
|-------------|---------------|
| 資料夾名稱與檔案名稱（皆使用者自訂或預設） | 任何 Item 名稱、帳號、密碼、私鑰、備註、標籤 |
| 檔案的 `appProperties`（`hsm.kind`、`hsm.vault_id` 等，標示檔案用途） | |
| 檔案數量與各檔案大小 | Item 型別（檔名為隨機識別碼，內容加密） |
| 檔案建立與修改時間 | 主機名稱與位址 |
| Vault 標頭中的 KDF 參數與 salt（非機密） | 主密碼、KEK、DEK、復原金鑰 |

檔案大小會透露粗略資訊（例如 RSA-4096 私鑰比 ed25519 大）。第一版接受此洩漏，第二階段可評估固定區塊填充（padding）。

自訂檔名可讓 Drive 上的檔案看起來不像密碼庫，但 `appProperties` 仍標示用途，且檔案內容開頭有 `HSM1` magic bytes；自訂檔名是便利與偏好功能，不是安全機制，文件須明確說明。

### 5.3 本機安全

1. KEK 與明文機密僅在處理期間存在記憶體，使用後立即覆寫（zeroize）。
2. agent 程序（FR-INIT-06）以 `mlock` 或等效機制防止 KEK 被換頁到磁碟；逾時或系統休眠時清除。
3. 本機快取只存密文，與 Drive 上內容相同；本機快取遭竊等同 Drive 檔案遭竊，仍受主密碼保護。
4. 本機設定目錄權限 `0700`，其中所有檔案 `0600`。
5. 任何日誌、錯誤訊息、崩潰報告不得包含機密值、主密碼、KEK、OAuth token。
6. 寫入私鑰到磁碟時，拒絕寫入到 world-readable 目錄並警告。

### 5.4 Google 授權安全

- 使用 OAuth 2.0 授權碼流程加 PKCE，redirect 至本機 loopback 位址。
- 預設權限範圍 `https://www.googleapis.com/auth/drive.file`：只能看到與操作本工具建立的檔案，無法讀取使用者其他 Drive 內容。
- Refresh token 存於作業系統金鑰圈；access token 只存在記憶體。
- 不使用 Google 帳號做任何身分驗證用途；Google 授權只決定「能否存取 Drive 檔案」，解密仍完全依賴主密碼。

### 5.5 威脅模型

| 威脅 | 防護措施 | 殘餘風險 |
|------|----------|----------|
| Google 帳號遭入侵，攻擊者取得 Drive 檔案 | 全部內容客戶端加密，Argon2id 抵抗離線暴力破解 | 弱主密碼可被破解；檔案大小洩漏粗略資訊 |
| 攻擊者刪除或竄改 Drive 檔案 | AES-GCM 驗證竄改；Drive 修訂歷史與本機快取可還原；`verify` 指令偵測 | 攻擊者可造成資料遺失（可用性），需靠備份補救 |
| 本機電腦遭竊 | 快取為密文；OAuth token 在金鑰圈；agent 逾時清除 KEK | 電腦處於 unlocked 且未鎖定螢幕時無保護 |
| 本機惡意軟體 | 超出本工具防護範圍 | 可讀取記憶體中的 KEK 與明文；列為已知限制 |
| 主密碼遺忘 | 復原金鑰 | 兩者皆遺失則資料永久不可讀 |
| 多裝置同時修改 | 每 Item 獨立檔案、修訂比對、保留雙方版本 | 需使用者手動解決衝突 |
| 工具本身的 OAuth client 憑證外洩 | 桌面應用程式的 client secret 本就不視為機密；PKCE 防止授權碼攔截 | 他人可冒用 client 名稱釣魚，需在文件提醒使用者 |

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
│  │ CLI      │───▶│ ├ 金鑰引擎    │    │ 保存 KEK     │ │
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
2. 取得 KEK：GUI 於解鎖後保存在程序記憶體；CLI 向 agent 詢問，agent 不存在或已逾時則提示主密碼，以 Vault 標頭的 KDF 參數推導，並以標頭中的驗證段確認。
3. 同步索引：下載 `index.enc`（若 Drive 修訂未變則略過），解密，更新本機索引快取。離線模式略過此步。
4. 依名稱在索引中找到 Item ID，確認本機快取的該 Item 檔案修訂與索引一致，否則下載。
5. 解密 Item 檔案，取出目前版本的私鑰，依要求格式轉換。
6. 檢查狀態非 `revoked`（或已 `--force`）。
7. 寫入目標檔案（`0600`）。
8. 寫入本機操作紀錄並排程上傳 `log.enc`。
9. zeroize 記憶體中的私鑰與 DEK。

### 6.3 本機目錄

| 路徑 | 內容 | 權限 |
|------|------|------|
| `~/.config/hsm/config.yaml` | Vault 清單、各 Vault 的 Drive 資料夾 ID、預設 Vault、agent 逾時等設定 | 0600 |
| `~/.config/hsm/device_id` | 本裝置識別碼（UUID） | 0600 |
| `~/.cache/hsm/<vault_id>/` | 各 Drive 檔案的快取，以及 `sync_state.json`（各檔案已知修訂、pending 標記、Drive 檔案 ID 對應）。本機快取一律使用固定內部名稱（`header`、`index`、`log`、`hosts`、`items/<id>`），與 Drive 上使用者自訂的檔名無關 | 0700 / 0600 |
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
5. 以 query「parent = 該資料夾 AND appProperties has { key='hsm.kind' and value='header' }」尋找 Vault 標頭檔（不依賴檔名）：
   - 找到一個 → 讀取並驗證格式；此為既有 Vault，命名規則由標頭載入。
   - 找到多個 → 回報錯誤（資料夾內有多個 Vault 標頭）。
   - 找不到且資料夾為空 → 以使用者指定的命名規則建立新 Vault。
   - 找不到但資料夾非空 → 詢問使用者是否仍在此建立新 Vault。
6. 在資料夾上設定 appProperties：{ "hsm.vault_id": <uuid>, "hsm.format": "1" }，供日後快速辨識。
7. 連接既有 Vault 時另提供「以 vault_id 搜尋」途徑：query「appProperties has { key='hsm.kind' and value='header' }」列出此 Google 帳號下所有本工具建立的 Vault 標頭，使用者從清單選擇，不需記得資料夾路徑。
```

### 7.3 檔案配置與命名規則

所有名稱皆為**使用者自訂、系統提供預設值**。下表的「預設檔名」只是預設值，實際名稱由 Vault 標頭的 `naming` 區段決定（見 8.1）。

| 命名規則鍵 | 預設檔名 | `hsm.kind` | 內容 | 是否加密 | 更新時機 |
|------------|----------|------------|------|:--:|------|
| `header` | `vault.hsm` | `header` | Vault 標頭：格式版本、vault_id、KDF 參數與 salt、KEK 驗證段、RK-wrapped KEK、命名規則、建立時間 | 部分（見 8.1） | 建立、變更主密碼、更新復原金鑰、修改命名規則 |
| `index` | `index.enc` | `index` | 所有 Item 的摘要：id、名稱、型別、標籤、狀態、updated_at、目前版本號、到期日、綁定主機 | 是 | 任何 Item 新增、改名、改標籤、狀態變更 |
| `log` | `log.enc` | `log` | 操作紀錄（滾動保留最近 5,000 筆） | 是 | 每次操作後（可批次延後上傳） |
| `hosts` | `hosts.enc` | `hosts` | 所有 Host 紀錄與綁定關係 | 是 | Host 變更時 |
| `items_folder` | `items` | `items_folder` | 存放 Item 檔案的子資料夾 | 不適用 | 建立時 |
| `item_pattern` | `{id}.enc` | `item` | 單一 Item 的全部版本；`{id}` 替換為該 Item 的 UUID v4 | 是 | 該 Item 變更時 |

- `{id}` 為 UUID v4，與 Item 內容無關。`item_pattern` 可為任意含 `{id}` 的樣式，例如 `IMG_{id}.jpg`、`{id}`（無副檔名）。
- `items_folder` 設為空字串時，Item 檔案直接放在 Vault 資料夾內，不建子資料夾。
- 工具讀寫檔案時一律先以 `appProperties`（`hsm.kind`、`hsm.vault_id`，Item 另有 `hsm.file_id`）查詢取得 Drive 檔案 ID，並快取在 `sync_state.json`；檔名只在建立與重新命名時使用。
- 所有檔案以 `multipart` 或 `resumable` 上傳；單檔小於 5 MB 用 multipart。
- 修改命名規則（FR-DRIVE-16）流程：更新標頭 `naming` 並上傳 → 逐一 `files.update(name=...)` → 全部成功後更新本機快取的對應；任一步失敗時狀態畫面顯示「命名規則更新未完成」，可重試。

### 7.4 修訂與衝突

- 每次 `files.get` 記錄 `headRevisionId`，存於 `sync_state.json`。
- 上傳前先 `files.get(fields=headRevisionId)` 比對；不一致即觸發 FR-SYNC-06 流程。此比對與上傳之間仍有競態視窗，接受此風險（個人使用情境下機率極低），且即使發生，Drive 修訂歷史仍保留被覆蓋的版本。
- 工具不呼叫 `revisions.delete`；使用者若要清理可在 Drive 網頁操作。

### 7.5 API 配額與錯誤處理

| 狀況 | 處理 |
|------|------|
| HTTP 403 `rateLimitExceeded` / 429 | 指數退避：1s、2s、4s、8s、16s，最多 5 次 |
| HTTP 5xx | 同上 |
| HTTP 401 | 以 refresh token 更新 access token 後重試一次；仍失敗則要求重新授權 |
| HTTP 404（檔案被使用者刪除） | 索引中標記該 Item 為 `missing`，`status` 顯示並建議從本機快取或 Drive 垃圾桶還原 |
| 網路不可用 | 立即改為離線模式，寫入操作保留 pending |
| Drive 空間不足 | 回報錯誤，保留 pending |

---

## 8. 資料格式

### 8.1 Vault 標頭 `vault.hsm`

JSON，明文可讀（不含機密）：

```json
{
  "format": 1,
  "vault_id": "0190a4c2-....",
  "created_at": "2026-10-28T08:00:00Z",
  "kdf": {
    "algorithm": "argon2id",
    "memory_kib": 262144,
    "iterations": 3,
    "parallelism": 4,
    "salt": "<base64 32 bytes>"
  },
  "kek_check": {
    "nonce": "<base64 12 bytes>",
    "ciphertext": "<base64>"
  },
  "recovery": {
    "enabled": true,
    "hkdf_salt": "<base64 32 bytes>",
    "nonce": "<base64 12 bytes>",
    "wrapped_kek": "<base64>"
  },
  "naming": {
    "header": "vault.hsm",
    "index": "index.enc",
    "log": "log.enc",
    "hosts": "hosts.enc",
    "items_folder": "items",
    "item_pattern": "{id}.enc"
  },
  "devices": [
    { "device_id": "...", "name": "macbook", "first_seen": "..." }
  ]
}
```

- `kek_check.ciphertext` 為字串 `"hsm-kek-check-v1"` 以 KEK 加密的結果，AAD 為 `vault_id`。
- `naming` 為此 Vault 的命名規則，值皆為使用者自訂，上例為預設值。標頭檔自己的名稱也記在其中，供重新命名時使用。
- `devices` 清單僅供顯示，不用於授權。

### 8.2 加密檔案格式（`index.enc`、`items/*.enc`、`log.enc`、`hosts.enc`、備份檔）

二進位格式，所有整數為 big-endian：

| 偏移 | 長度 | 欄位 | 說明 |
|------|------|------|------|
| 0 | 4 | magic | ASCII `HSM1` |
| 4 | 1 | kind | 1 = index、2 = item、3 = log、4 = hosts、5 = backup |
| 5 | 16 | vault_id | UUID bytes |
| 21 | 16 | file_id | item_id；非 Item 檔案為固定 UUID |
| 37 | 12 | dek_nonce | 包裝 DEK 用 |
| 49 | 48 | wrapped_dek | 32 bytes DEK + 16 bytes GCM tag |
| 97 | 12 | nonce | 內容加密用 |
| 109 | 4 | plaintext_len | 解密後長度，供預先配置 |
| 113 | n | ciphertext | 內容 + 16 bytes GCM tag |

- AAD = 偏移 0 到 108 的全部標頭 bytes。
- 內容為 JSON，經 zstd 壓縮後再加密（壓縮於加密前，個人使用情境不考慮壓縮側通道）。

### 8.3 Item 內容（解密後 JSON）

```json
{
  "id": "0190a4c2-....",
  "type": "key_pair",
  "name": "prod-web-01-deploy",
  "tags": ["prod", "web"],
  "status": "active",
  "revoked_reason": null,
  "expires_at": null,
  "rotation_days": 90,
  "created_at": "...",
  "updated_at": "...",
  "updated_by_device": "...",
  "conflict": false,
  "current_version": 2,
  "versions": [
    {
      "version": 2,
      "created_at": "...",
      "created_by_device": "...",
      "note": "rotated",
      "fields": { "...型別專屬欄位..." }
    },
    { "version": 1, "...": "..." }
  ]
}
```

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
      "current_version": 2, "expires_at": null, "conflict": false,
      "summary": { "fingerprint_sha256": "SHA256:...", "algorithm": "ed25519" }
    }
  ]
}
```

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
      "keys": [
        { "item_id": "...", "remote_user": "deploy", "usage": "部署", "bound_at": "..." }
      ],
      "created_at": "...", "updated_at": "..."
    }
  ]
}
```

### 8.6 操作紀錄內容

每筆：`{ "at", "device_id", "action", "item_id?", "item_name?", "result", "details?" }`。`action` 例如 `item.create`、`item.update`、`key.export_private`、`key.rotate`、`key.revoke`、`sync.conflict`、`vault.password_change`。

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
| 同步與狀態 | pending、衝突列表（逐筆比較、選擇保留）、到期與逾期輪替提醒、missing 項目 | P0 |
| 設定 | 自動鎖定時間、剪貼簿清除秒數、Drive 資料夾資訊、Google 授權狀態與撤銷、變更主密碼、重新產生復原金鑰、語言 | P0 |
| 設定 → Drive 命名 | 顯示目前資料夾路徑與各檔案名稱；修改資料夾路徑（搬移）與命名規則（重新命名所有檔案），顯示進度與失敗重試 | P1 |
| 操作紀錄 | 依時間、操作、Item 篩選 | P1 |
| 系統匣選單 | 快速搜尋、鎖定、開啟主視窗、結束 | P1 |

#### 9.1.2 行為規範

- 任何顯示或複製機密值的動作都寫入操作紀錄。
- 匯出私鑰一律以「另存新檔」對話框指定路徑，寫入後權限 `0600`；目標目錄為 world-readable 時警告。
- 破壞性動作（撤銷、刪除、明文匯出、`--force` 匯出已撤銷私鑰）需二次確認，確認對話框需輸入 Item 名稱。
- 視窗內不得出現可被螢幕截圖工具自動擷取的明文機密，除非使用者主動點擊「顯示」。
- 未解決的衝突與逾期提醒以主視窗頂部橫幅提示，不彈出阻斷式對話框。

### 9.2 CLI

#### 9.2.1 基本

- 執行檔名稱：`hsm`（待決議 Q-03）。
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

# 同步與狀態
hsm sync [--rebuild-index]
hsm status                         # pending、衝突、到期、逾期輪替、missing
hsm resolve <name> --keep local|remote|<version>
hsm verify
hsm gc [--older-than 30d]

# 帳密與通用項目
hsm list [--type TYPE] [--tag TAG] [--status S]
hsm search <query>
hsm add <name> --type credential [--username U] [--password P | --generate [--length N] | --stdin] [--url URL] [--tag T ...]
hsm add <name> --type generic [--from-file PATH | --field k=v ...]
hsm show <name> [--version N]      # 顯示非機密欄位與版本列表
hsm get <name> [--field password] [--version N] [--copy | --stdout]
hsm totp <name> [--copy]
hsm edit <name> [--username U] [--password P | --generate | --stdin] [--url URL] [--notes N] [--expires DATE]
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
hsm key install <name> [--dir ~/.ssh] [--ssh-config --host HOSTNAME]
hsm key ssh-add <name> [--lifetime 1h]
hsm key rotate <name> [--note TEXT]
hsm key revoke <name> --reason TEXT
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
hsm host bind <host> <key> --remote-user USER [--usage TEXT]
hsm host unbind <host> <key>
hsm host authorized-keys <host> [--out PATH]

# 備份與匯入匯出
hsm export --encrypted --out PATH
hsm export --plain --out PATH      # 二次確認
hsm import --from PATH [--format hsm|chrome-csv|bitwarden|1password|keepass|firefox-csv|edge-csv] [--on-duplicate version|skip|new] [--dry-run]
hsm import --chrome [--profile NAME] [--on-duplicate ...] [--dry-run]   # 直接讀取 Chrome 設定檔（P1）

# 紀錄
hsm log [--since T] [--action A] [--item NAME] [--limit N]
```

#### 9.2.3 行為規範

- `get`、`key private`、`cert export` 預設寫檔或複製到剪貼簿；輸出到 stdout 需明確 `--stdout`，且 stdout 為終端時再次確認。
- `--password P` 之類在命令列直接帶機密的參數，文件須警告會留在 shell history，建議用 `--stdin` 或 `--generate`。
- 所有需要主密碼的指令在非 TTY 且無 agent、無 `HSM_PASSWORD_FILE` 時，以結束碼 3 失敗。
- 結束碼：0 成功、1 一般錯誤、2 參數錯誤、3 未解鎖或主密碼錯誤、4 Google 授權失效、5 網路或 Drive 錯誤、6 存在衝突需處理。

---

## 10. 非功能需求

| 類別 | 需求 |
|------|------|
| 效能 | 解鎖（Argon2id 推導）在一般筆電上 1–2 秒；已解鎖且索引已快取時，`get` 在 200 ms 內完成（不含 Drive 往返）；完整同步 500 個 Item 在 60 秒內完成。 |
| 容量 | 目標：單一 Vault 2,000 個 Item、每 Item 10 版本、操作紀錄 5,000 筆；單次 Chrome 匯入 2,000 筆在 30 秒內完成預覽。 |
| 可靠性 | 任何時刻程序被中斷（Ctrl-C、斷電）都不得造成已確認寫入的資料遺失；寫入快取採「寫暫存檔再 rename」。 |
| 相容性 | Linux x86_64 / arm64、macOS（Intel / Apple Silicon）、Windows 10+。桌面應用程式以各平台原生安裝檔發布（`.dmg`、`.msi`、`.AppImage` / `.deb`）。 |
| 互通性 | 匯出的 OpenSSH 金鑰需能被 `ssh-keygen -y`、`ssh-add` 正確讀取；PEM 需能被 `openssl` 讀取。 |
| 測試 | 加密層與檔案格式單元測試覆蓋率 ≥ 90%；同步引擎以本機資料夾儲存層做整合測試，涵蓋衝突情境；金鑰匯入匯出與 OpenSSH / OpenSSL 做互通測試。 |
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

| 階段 | 目標 | 主要交付 | 對應需求 |
|------|------|----------|----------|
| **Phase 0：確認** | 規格審閱、技術選型定案、建立 Google Cloud OAuth client、repo 骨架 | 本文件 1.0、CI 流程、專案結構、儲存層介面定義 | 第 13 章全部 |
| **Phase 1：MVP** | 桌面應用程式在單一裝置可完整使用，第二裝置可連接同步 | Core：加密層與檔案格式、Vault 建立與解鎖、復原金鑰、Google 授權與資料夾建立、帳密與通用 Item、金鑰產生 / 匯入 / 掃描 / 匯出 / 安裝 / 輪替 / 撤銷、Host 綁定與 authorized_keys 輸出、同步與衝突、離線快取、加密備份匯出匯入、Chrome CSV 匯入、操作紀錄。GUI：9.1.1 所有 P0 畫面、自動鎖定、剪貼簿清除、拖放匯入 | 所有 P0 |
| **Phase 2：日常順手** | CLI、減少摩擦、增加安全提醒 | CLI 與 agent、Chrome 設定檔直接讀取、其他密碼管理器匯入、變更主密碼、ssh-add、TOTP、到期與輪替提醒、指紋反查、verify、軟刪除垃圾桶、Drive 狀態顯示、系統匣、操作紀錄畫面 | 所有 P1 |
| **Phase 3：擴充** | 進階整合 | 多 Vault、背景同步、Host 掃描與公鑰推送、附件型 Item、PuTTY 匯出、其他 Chromium 瀏覽器匯入、全域快捷鍵、其他儲存後端、檔案大小填充 | P2 |

每階段結束條件：對應需求的自動化測試全部通過、第 5 章安全檢查逐項確認、使用文件更新。

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
| 1 | `CORRUPTED` | 檔案格式錯誤或 GCM 驗證失敗 |
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
| 5 | `HEADER_AMBIGUOUS` | 資料夾內有多個 Vault 標頭 |
| 2 | `INVALID_NAMING` | 檔名或 Item 檔名樣式不符合 FR-DRIVE-15 規則 |
| 5 | `RENAME_INCOMPLETE` | 命名規則更新未全部完成，可重試 |
| 6 | `CONFLICT_PENDING` | 存在未解決的同步衝突 |
