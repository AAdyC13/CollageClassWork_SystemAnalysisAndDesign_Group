# 實體關係圖 (Entity Relationship Diagram)

模組化記帳系統採用混合式資料架構，結合 Android Room 核心資料庫與外掛模組動態擴充表。

---

## 1. ERD 架構圖

```mermaid
    %% --- 外部實體 ---
    User[使用者]
    FileSystem[Android 檔案系統<br/>(Assets/Resources)]
    Database[SQLite/Room 資料庫<br/>(Expenses & Extensions)]

    %% --- 系統程序 ---
    subgraph WebView_Frontend [Web 前端環境]
        P1((P1. 系統初始化與<br/>模組管理))
        P2((P2. 記帳介面與<br/>資料輸入))
    end

    subgraph Android_Native [Android 原生環境]
        P3((P3. 原生橋接介面<br/>AndroidBridge))
        P4((P4. 背景工作處理<br/>BackgroundWorker))
        P5((P5. 資料庫存取<br/>Data Repository))
    end

    %% --- 資料流 ---
    
    %% 1. 初始化與模組載入流
    User -->|啟動應用| P1
    P1 -->|1. 請求模組清單与Schema| P3
    P3 -->|2. 轉發讀取請求| P4
    P4 -->|3. 讀取 info.json/tech.json| FileSystem
    FileSystem -->|4. 回傳 JSON 配置| P4
    P4 -->|5. 封裝模組資料| P3
    P3 -->|6. 回傳至前端| P1
    
    %% 2. 記帳資料流
    User -->|輸入金額與備註| P2
    P2 -->|7. 提交記帳資料 (Payload)| P3
    P3 -->|8. 請求寫入資料| P5
    
    %% 3. 資料庫交互
    P5 -->|9. 寫入核心帳目 (Expense)| Database
    P5 -->|10. 寫入擴充資料 (PluginData)| Database
    Database -->|11. 回傳寫入結果| P5
    P5 -->|12. 回傳成功狀態| P3
    P3 -->|13. 更新 UI| P2
    P2 -->|14. 顯示儲存結果| User

    %% --- 備註 ---
    %% P1 對應 ModulesManager.js, Runtime.js
    %% P2 對應 Recorder.js
    %% P3 對應 Interface.kt (AndroidBridge)
    %% P4 對應 Interface.kt (BackgroundWorker)
    %% P5 對應 ExpenseRepository.kt, ExpenseDao.kt

```
