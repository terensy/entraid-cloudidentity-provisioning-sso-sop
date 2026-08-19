# Entra ID 佈建使用者/群組至 Google Cloud Identity 並設定 SSO 部署與維護 SOP

> 透過 Microsoft Entra ID 的企業應用程式功能，將公司的使用者、群組自動同步（佈建）到 Google Cloud Identity，並設定 SAML 單一登入（SSO），讓使用者可以直接用 Entra ID 帳號密碼登入 Google Cloud Console 及其他 Google 服務。

本文件所有截圖、設定內容均整理自一次真實部署紀錄，並已對照 [Google Cloud 官方架構指南](https://docs.cloud.google.com/architecture/identity/federating-gcp-with-azure-ad-configuring-provisioning-and-single-sign-on) 與 [Google Admin 說明中心](https://knowledge.workspace.google.com/admin/apps/setting-up-sso) 校對正確性，並補充了原文件沒有涵蓋、但官方文件特別強調的注意事項。

---

## 目錄

1. [名詞對照](#名詞對照)
2. [整體架構](#整體架構)
3. [開始之前](#開始之前)
4. [系統架構參數表](#系統架構參數表)
5. [Part 1：設定使用者與群組自動佈建（Provisioning）](#part-1設定使用者與群組自動佈建provisioning)
6. [Part 2：設定 SAML 單一登入（SSO）](#part-2設定-saml-單一登入sso)
7. [Part 3：驗證登入](#part-3驗證登入)
8. [疑難排解](#疑難排解)
9. [參考資料](#參考資料)

---

## 名詞對照

| 名詞 | 說明 |
|---|---|
| **Google Cloud Identity**（本文件中與「Cloud Identity」同義） | Google 提供的身分與裝置管理平台，用來集中管理使用者帳號、群組，並作為登入 Google Cloud、Gemini Enterprise 等服務的身分來源。 |
| **佈建（Provisioning）** | 讓 Entra ID 自動把使用者、群組的資料「同步」寫入 Google Cloud Identity，包含新增、更新、刪除，不需要 IT 手動一個個建立帳號。技術上採用 **SCIM** 協定。 |
| **SSO（Single Sign-On，單一登入）** | 使用者用同一組帳密（這裡是 Entra ID 帳密），就能登入 Google Cloud Console 等服務，不需要另外設定 Google 密碼。技術上採用 **SAML 2.0** 協定。 |
| **企業應用程式（Enterprise Application）** | Entra ID 中代表「一個外部系統」的物件，本文件會建立兩個，分別對應「佈建」與「SSO」兩種不同用途。 |
| **屬性對應（Attribute Mapping）** | 定義 Entra ID 的欄位（如 `displayName`、`userPrincipalName`）要如何轉換、寫入 Google Cloud Identity 對應的欄位（如 `primaryEmail`）。 |
| **實體 ID（Entity ID）／ACS 網址** | SAML 協定中，Google（服務提供者，SP）用來識別自己、以及接收 Entra ID（身分提供者，IdP）驗證回應的網址。 |
| **NameID / 唯一使用者識別碼** | SAML 驗證回應中用來指出「這是哪一個使用者」的欄位，Google 會拿這個值去比對 Cloud Identity 中的使用者帳號。 |

---

## 整體架構

這份 SOP 會在 Entra ID 中建立**兩個獨立的企業應用程式**，分別負責不同的工作，不要把兩者搞混或合併成一個：

```mermaid
flowchart TB
    subgraph Entra["Microsoft Entra ID"]
        A["企業應用程式 A
Google Cloud (Provisioning)
負責同步帳號資料"]
        B["企業應用程式 B
Google Cloud (SSO)
負責處理登入驗證"]
    end

    subgraph GCI["Google Cloud Identity"]
        C[(使用者 / 群組資料)]
        D["SAML SSO 設定檔"]
    end

    A -- "SCIM 佈建
（單向：Entra → Google）" --> C
    B <-- "SAML 驗證
（使用者登入時雙向交握）" --> D
    D -. "登入時比對身分" .-> C

    style A fill:#0078D4,color:#fff
    style B fill:#0078D4,color:#fff
    style C fill:#4285F4,color:#fff
    style D fill:#34A853,color:#fff
```

- **企業應用程式 A（Provisioning）**：只負責把 Entra ID 的使用者、群組「寫進」Google Cloud Identity，跟登入驗證無關，因此設定上會刻意把「為使用者啟用登入」關閉。
- **企業應用程式 B（SSO）**：只負責使用者登入時的身分驗證，不會做任何帳號同步的動作。

---

## 開始之前

以下前置準備項目為官方文件明確要求或建議，原文件未特別條列，這裡整理供操作前確認：

### Google Cloud Identity 端

- 一個 Cloud Identity 組織，且已規劃好要使用的網域（Domain）。
- ⚠️ **不要用系統預設建立的 Super Admin 帳號**來做 Entra ID 的自動佈建。請另外建立一個專門用途的管理員帳號（本文件 Part 1 步驟 1 會示範），原因：
  - 佈建帳號的憑證會被存放在 Entra ID 系統中，權限範圍應盡量獨立、可控管。
  - 若之後 SSO 套用到整個組織，這個佈建帳號也會被強制改用 Entra ID 登入——但佈建功能需要的是「Google 密碼」授權，兩者會互相打架。**建議把這個佈建帳號放在獨立的機構單位（OU），並在 Part 2 設定 SSO 指派時，將這個 OU 排除在 SSO 範圍之外（指派為「無」）**，避免日後帳號被鎖在外面、無法重新授權。

### Microsoft Entra ID 端

- 一個具備 **全域管理員（Global Administrator）** 權限的帳號，才能新增企業應用程式、設定佈建與 SAML。

---

## 系統架構參數表

| 參數項目 | 範例值（本次部署） | 說明 |
|---|---|---|
| Google Cloud Identity 主要網域 | `demo.com` | 使用者、群組信箱所屬的網域 |
| Entra ID 租戶網域 | `demo.com`（與 Cloud Identity 相同） | 若與 Cloud Identity 網域不同，需額外設定屬性轉換，詳見 Part 1 步驟 5 |
| 佈建專用帳號 | `entra-id-provisioning@demo.com` | 專門用來授權 Entra ID 佈建功能的 Cloud Identity 帳號 |
| Provisioning 應用程式名稱 | `Google Cloud (Provisioning)` | Entra ID 中負責同步帳號的企業應用程式 |
| SSO 應用程式名稱 | `Google Cloud (SSO)` | Entra ID 中負責登入驗證的企業應用程式 |
| Google SAML 設定檔名稱 | `EntraID` | 在 Cloud Identity 建立的第三方 IdP SSO 設定檔 |

> 以上為範例表，請依照你自己的環境填入。

---

## Part 1：設定使用者與群組自動佈建（Provisioning）

### 步驟 1：建立專用的佈建帳號

在 Google Cloud Identity 管理控制台中，建立一個專門用來做佈建授權的帳號（不要使用預設 Super Admin）。

1. 進入「目錄」>「使用者」，點選「新增使用者」。

   ![新增使用者](images/01-cloudidentity-新增使用者.png)

2. 填入需要的資訊後，點選「新增使用者」。

   ![填入使用者資訊](images/02-cloudidentity-填入使用者資訊.png)

3. 回到使用者列表，找到剛剛建立的使用者並點選進入，在「管理員角色與權限」設定「超級管理員」權限。

   ![設定超級管理員](images/03-cloudidentity-設定超級管理員.png)

   > 建議依照〔開始之前〕的說明，把這個帳號安排在獨立的機構單位，方便日後在 SSO 設定中單獨排除。

### 步驟 2：在 Entra ID 建立負責佈建的 Google 應用程式

1. 回到 Entra 系統管理中心，左側欄點選「企業應用程式」，在上方功能列點選「新增應用程式」。

   ![新增應用程式](images/04-entra-新增應用程式.png)

2. 在應用程式資源庫中選擇「**Google Cloud Platform**」（顯示為 Google Cloud / G Suite Connector by Microsoft）。

   ![選擇 Google Cloud Platform](images/05-entra-選擇google-cloud-platform.png)

3. 自訂應用程式名稱（例如 `Google Cloud (Provisioning)`，方便日後和 SSO 應用程式區分），點選「建立」。

   ![命名 Provisioning 應用程式](images/06-entra-命名provisioning應用程式.png)

### 步驟 3：設定應用程式屬性

1. 回到「企業應用程式」，點選剛剛建立的應用程式，再點選「屬性」。

   ![屬性頁籤](images/07-entra-屬性頁籤.png)

2. 將「為使用者啟用登入?」以及「需要指派?」都調整為「**否**」，點擊下面「儲存」。

   ![啟用登入與需要指派設為否](images/08-entra-啟用登入與需要指派設為否.png)

   > 這個應用程式只負責同步帳號資料，不會有人透過它登入，所以兩個開關都關閉。

### 步驟 4：設定佈建連線

1. 點選左側「佈建」，進入畫面後點選「新設定」。

   ![點選佈建](images/09-entra-點選佈建.png)

   ![點選新設定](images/10-entra-點選新設定.png)

2. 若畫面顯示「只有從 Azure 入口網站才支援對應用程式進行授權」，點選提示中的連結，會導向 Azure 入口網站繼續設定。

   ![僅 Azure 入口網站支援授權提示](images/11-entra-僅azure入口網站支援授權提示.png)

3. 「佈建模式」選擇「**自動**」，展開「管理員認證」，點選「授權」。

   ![佈建模式自動並點選授權](images/12-entra-佈建模式自動並點選授權.png)

4. 使用〔步驟 1〕在 Cloud Identity 建立的專用帳號進行授權登入。

   ![使用佈建帳號登入授權](images/13-cloudidentity-使用佈建帳號登入授權.png)

5. 驗證完成後回到畫面，點選「測試連接」，連接沒問題會跳出成功通知。

   ![測試連接成功通知](images/14-entra-測試連接成功通知.png)

### 步驟 5：設定屬性對應（僅網域不同時需要）

> 若 Entra ID 的網域跟 Google Cloud Identity 的網域**相同**，這一步可以跳過，直接前往〔步驟 6〕。若不同，才需要以下設定，讓 Entra ID 知道該用哪個網域組成 Google 端的信箱。

1. 展開「對應」，點選「**Provision Microsoft Entra ID Users**」。

   ![對應 Provision Users](images/15-entra-對應-provision-users.png)

2. 找到 Google 屬性「**primaryEmail**」，點選右邊「編輯」。

   ![primaryEmail 編輯](images/16-entra-primaryemail編輯.png)

3. 修改「對應類型」為「**運算式**」，貼上以下運算式，並將 `GROUPS_DOMAIN` 替換成 Google Cloud Identity 使用的網域名稱：

   ```
   Join("@", NormalizeDiacritics(StripSpaces([displayName])), "GROUPS_DOMAIN")
   ```

   點選「確定」。

   ![primaryEmail 運算式設定](images/17-entra-primaryemail運算式設定.png)

4. 另外，「name.familyName」跟「name.givenName」要設定「若為 null，則為預設值」，填入底線 `_` 或其他自訂的預設值——因為部分帳號在 Entra ID 端可能沒有填寫姓氏或名字，若不設定預設值，佈建到這類帳號時會直接失敗。

   ![familyName/givenName 預設值](images/18-entra-familyname-givenname預設值.png)

5. 屬性都設定完成後，記得點選上方的「儲存」。

6. 回到「對應」，點選「**Provision Microsoft Entra ID Groups**」，進行群組對應內容更新。

   ![對應 Provision Groups](images/19-entra-對應-provision-groups.png)

7. 找到 Google 屬性「**email**」，點選「編輯」，同樣修改「對應類型」為「運算式」，貼上跟步驟 3 相同的運算式（`GROUPS_DOMAIN` 一樣要替換成實際網域），點選「確定」。

   ![Groups email 運算式設定](images/20-entra-groups-email運算式設定.png)

### 步驟 6：儲存並啟動佈建

1. 都設定完成後，回到佈建頁面，點選上方的「儲存」。

   ![佈建頁面儲存](images/21-entra-佈建頁面儲存.png)

2. 回到「Entra 系統管理中心」，點選「佈建」再到「概觀」，找到「開始佈建」並點選，跳出的確認視窗點選「確定」。

   ![開始佈建](images/22-entra-開始佈建.png)

3. 佈建沒問題的話，下方會顯示佈建的內容及狀態（例如同步了幾個使用者、幾個群組）。

   ![佈建週期狀態完成](images/23-entra-佈建週期狀態完成.png)

### 步驟 7：確認同步結果

回到 Google Cloud Identity 的「使用者」跟「群組」頁面，就可以看到剛剛同步過來的帳號與群組資訊。

![Cloud Identity 使用者已同步](images/24-cloudidentity-使用者已同步.png)

![Cloud Identity 群組已同步](images/25-cloudidentity-群組已同步.png)

> 補充：在 Entra ID 佈建功能的「佈建記錄」中，可以看到每一次佈建的詳細 log，如果佈建過程發生錯誤，也可以在這裡找到錯誤訊息，是排查問題的第一步。

![佈建記錄 log](images/26-entra-佈建記錄log.png)

---

## Part 2：設定 SAML 單一登入（SSO）

### 步驟 1：建立第二個 Google 應用程式（專門處理登入）

重複〔Part 1 步驟 2〕的做法，在 Entra ID Console 再建立一個新的 Google 應用程式，名稱可以取為「Google SSO」或「Google Cloud (SSO)」。

建立完成後，進入「屬性」頁面設定：

- 「為使用者啟用登入?」設為「**是**」。
- 「是否要向使用者顯示?」可依企業內部規定調整。
- 「需要指派?」依實際需求決定：只想讓特定人員或群組能透過 SSO 登入 Google Cloud，設為「是」；否則設為「否」。

![SSO 應用程式屬性設定](images/27-entra-sso應用程式屬性設定.png)

### 步驟 2：在 Google Cloud Identity 建立 SAML 設定檔

1. 回到 Google Cloud Identity Console，前往「安全性」>「驗證」>「使用第三方識別資訊提供者 (IDP) 的單一登入 (SSO) 服務」，點選「新增 SAML 設定檔」。

   ![新增 SAML 設定檔按鈕](images/28-cloudidentity-新增saml設定檔按鈕.png)

2. 為這個 SAML 設定檔取一個名稱，往下捲動到最下方點選「儲存」。

   ![新增 SAML 設定檔表單](images/29-cloudidentity-新增saml設定檔表單.png)

3. 儲存後，設定檔內會顯示「實體 ID」跟「ACS 網址」兩組網址，稍後會用在 Entra ID 端的設定。

   ![實體 ID 與 ACS 網址](images/30-cloudidentity-實體id與acs網址.png)

### 步驟 3：在 Entra ID 設定基本 SAML

1. 回到 Entra ID Console，在剛剛的「Google SSO」應用程式中點選「單一登入」，在「基本 SAML 設定」點選「編輯」。

   ![單一登入編輯按鈕](images/31-entra-單一登入編輯按鈕.png)

2. 依序填入：
   - **識別碼（實體識別碼）**：填入 Google Cloud Identity 的「實體 ID」。
   - **回覆 URL**：填入 Google Cloud Identity 的「ACS 網址」。
   - **登入 URL**：填入以下網址（`PRIMARY_DOMAIN` 換成 Google Cloud Identity 的網域）：
     ```
     https://www.google.com/a/PRIMARY_DOMAIN/ServiceLogin?continue=https://console.cloud.google.com/
     ```

   設定完成後點選上方「儲存」。

   > 若這三個欄位中已經有預設值（例如 Entra 系統自動帶入的舊資料），記得先移除再填入正確的值。

   ![基本 SAML 設定編輯](images/32-entra-基本saml設定編輯.png)

### 步驟 4：設定屬性與宣告

1. 點選「屬性與宣告」進入編輯頁面。

   ![屬性與宣告編輯連結](images/33-entra-屬性與宣告編輯連結.png)

2. 將「其他宣告」底下的預設內容都移除，「必要的宣告」只留下「唯一使用者識別碼 (名稱識別碼)」。確認畫面應如下所示：

   ![屬性與宣告確認畫面](images/34-entra-屬性與宣告確認畫面.png)

3. 進入「唯一使用者識別碼 (名稱識別碼)」編輯頁面。若 Entra ID 網域與 Google Cloud Identity 網域**不一致**，需要額外設定轉換規則：來源選擇「轉換」，轉換方式選 `Join()`，參數 1 填入來源屬性（例如 `user.userprincipalname`）、分隔符號填 `@`，參數 2 的「屬性名稱」填入 Google Cloud Identity 的網域。設定完成後點選相關的新增／儲存按鈕。

   ![唯一使用者識別碼轉換設定](images/35-entra-唯一使用者識別碼轉換設定.png)

### 步驟 5：下載憑證並取得安裝資訊

點選「SAML 憑證」區塊中「憑證 (Base64)」下載，並記下底下「安裝 Google Cloud (SSO)」區塊列出的三組網址（**登入 URL**、**Microsoft Entra 識別碼**、**登出 URL**），稍後在 Google Cloud Identity 設定中會用到。

![SAML 憑證下載與安裝資訊](images/36-entra-saml憑證下載與安裝資訊.png)

### 步驟 6：回到 Google Cloud Identity 完成 SSO 設定檔

回到剛剛建立的 SSO 設定檔，點選進入編輯畫面，依序填入：

- **IDP 實體 ID**：填入〔步驟 5〕紅色區塊中的「Microsoft Entra 識別碼」。
- **登入網頁網址**：填入〔步驟 5〕紅色區塊中的「登入 URL」。
- **登出網頁網址**：填入〔步驟 5〕紅色區塊中的「登出 URL」。
- **變更密碼網址**：填入 `https://account.activedirectory.windowsazure.com/changepassword.aspx`。
- 上傳〔步驟 5〕下載的 SSO 憑證。

完成後點選「儲存」。

![Cloud Identity IDP 詳細資料表單](images/37-cloudidentity-idp詳細資料表單.png)

### 步驟 7：指派 SSO 設定檔

1. 在「管理單一登入 (SSO) 設定檔指派作業」區塊，設定哪一個機構單位要套用哪一個 SSO 設定檔。若不指定，預設會是整個組織套用同一個 SSO 設定檔。

   ![SSO 設定檔指派作業](images/38-cloudidentity-sso設定檔指派作業.png)

   > 依照〔開始之前〕的建議，若佈建專用帳號有獨立的機構單位，記得在這裡把該機構單位的 SSO 設定檔指派為「無」，避免佈建帳號被強制導向 SSO 登入。

2. 指派設定頁面中，預設選擇「讓 Google 提醒他們輸入使用者名稱，然後將他們重新導向這個設定檔的 IdP 登入頁面」，維持這個選項即可。

   ![登入重新導向選項](images/39-cloudidentity-登入重新導向選項.png)

完成以上設定後，就可以透過 Entra ID SSO 登入 Google Cloud Console。**別忘了 Google Cloud 的 IAM 仍然要另外指派角色給使用者或群組**，SSO 只解決「登入驗證」，不會自動給予 GCP 資源的存取權限。

### 步驟 8（選用）：其他 Google 服務的網域專屬登入網址

如果除了 Google Cloud Console 之外，還要讓使用者透過同一組 SSO 登入其他 Google 服務（例如 Gemini Enterprise、Google 文件、Gmail 等），需要依照下表，把對應服務的網址設定成 Entra ID SSO 應用程式的「登入 URL」（可依需要建立多個 SSO 應用程式或另外設定重新導向規則），表中 `DOMAIN` 需替換為實際網域：

![各服務網域專屬網址對照表](images/40-cloudidentity-各服務網域專屬網址對照表.png)

---

## Part 3：驗證登入

都設定完成後，從一般 Google Cloud 登入介面輸入 email 帳號，就會自動導向 Microsoft 的登入驗證畫面。

![Microsoft 登入畫面](images/41-microsoft-登入畫面.png)

> ⚠️ **網域不一致時的登入注意事項**：如果 Google Cloud Identity 網域跟 Entra ID 網域不一樣，在 Google Cloud 登入畫面要輸入 **Cloud Identity 網域**的 email 帳號；轉導到 Microsoft 登入畫面後，則要輸入 **Entra ID 網域**的 email 帳號，才能成功轉導登入。

建議先找 2-3 位不同部門的同事實際測試登入、確認群組同步與權限都正確後，再正式公告給全公司使用。

---

## 疑難排解

### 「管理員認證」授權被存取權政策擋下

- **情境**：在 Part 1 步驟 4 點選「授權」，登入 Cloud Identity 帳號後被組織的存取權政策（Access Level / Context-Aware Access）擋下，無法完成授權。
- **排除方案**：依照 Google 官方文件說明，將 Google 官方指定的用戶端 ID 加入組織的信任應用程式清單：
  ```
  283861851054-1n5jlu9rm93njt6kh3k5k7pqv73q7d8d.apps.googleusercontent.com
  ```
  加入後重新進行授權即可。

### 佈建帳號登入被要求走 SSO，無法重新授權

- **情境**：Part 2 完成 SSO 設定並套用到全組織後，回頭要重新測試或修改 Part 1 的佈建連線時，登入畫面被導向 Entra ID SSO，但佈建功能需要的是原生 Google 密碼授權，導致卡住。
- **排除方案**：這正是〔開始之前〕與〔Part 2 步驟 7〕提醒的原因——回到 Cloud Identity 的「管理單一登入 (SSO) 設定檔指派作業」，找到佈建帳號所在的機構單位，將其 SSO 設定檔指派調整為「無」，即可恢復用原生密碼登入該帳號。

### 佈建執行後使用者或群組沒有出現在 Google Cloud Identity

- **情境**：點選「開始佈建」後，週期狀態顯示完成，但某些使用者或群組沒有同步過去。
- **排除方案**：
  1. 到 Entra ID「佈建記錄」查看該筆記錄的詳細錯誤訊息（通常是必填屬性缺漏，例如姓氏/名字為空但沒設定預設值）。
  2. 確認該使用者/群組是否有被指派給「Google Cloud (Provisioning)」應用程式（若〔Part 1 步驟 3〕的「需要指派」被誤設為「是」，未指派的使用者將不會被佈建）。

---

## 參考資料

- [Google Cloud：Federating Google Cloud with Azure Active Directory（配置佈建與單一登入的官方架構指南）](https://docs.cloud.google.com/architecture/identity/federating-gcp-with-azure-ad-configuring-provisioning-and-single-sign-on)
- [Google Admin 說明中心：設定第三方 IdP 的單一登入 (SSO) 服務](https://knowledge.workspace.google.com/admin/apps/setting-up-sso)
- [Microsoft Entra：Google Cloud / G Suite Connector 佈建教學](https://learn.microsoft.com/en-us/entra/identity/saas-apps/g-suite-provisioning-tutorial)

> 原文件第三個參考連結指向 Microsoft「內部部署 LDAP 連接器設定」文件，與本文件討論的 SaaS 應用程式佈建主題不符，已替換為正確對應的官方教學連結。

---

*本文件內容涉及公司內部真實網域、帳號等資訊，請妥善保管本 Repository 的存取權限，勿設為公開（Public）。*
