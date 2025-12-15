# 實體關係圖 (Entity Relationship Diagram)

模組化記帳系統採用混合式資料架構,結合 Android Room 核心資料庫與外掛模組動態擴充表。

---

## 1. ERD 架構圖

```mermaid
erDiagram
    %% --- 靜態核心層 (Android Room 管理) ---
    %% 這些是 App 安裝時就存在的固定資料表
    
    EXPENSES {
        int id PK "核心記帳 ID (流水號)"
        long timestamp "交易時間戳 (秒)"
        double amount "金額 (正=收, 負=支)"
        string description "交易備註"
    }

    PLUGINS {
        string pluginId PK "套件 ID (例如: com.user.invoice)"
        string displayName "套件顯示名稱"
        string version "版本號"
        string configJson "設定檔 (JSON)"
        boolean isEnabled "啟用狀態"
        string sourcePath "程式碼路徑"
    }

    %% --- 動態擴充層 (JS / SQL 動態管理) ---
    %% 這些表是 Plugin 運作時自己建立的,名稱不固定
    
    PLUGIN_DYNAMIC_TABLES {
        int _id PK "動態表主鍵"
        int expense_id FK "關聯帳目 ID (外鍵)"
        text custom_data "自定義資料 (如: 發票號, 圖片路徑)"
        text other_columns "其他欄位..."
    }

    %% --- 關係定義 ---

    %% 1. 管理關係:一個 Plugin 可以建立多張表
    PLUGINS ||--o{ PLUGIN_DYNAMIC_TABLES : "建立與管理 (Create)"

    %% 2. 擴充關係:核心帳目可以被多個動態表擴充
    %% (例如:一筆帳目可以同時有 '電子發票' 和 '打卡地點' 的資料)
    EXPENSES ||--o{ PLUGIN_DYNAMIC_TABLES : "擴充資訊 (Extend)"
```

---

## 2. 組合實體範例 (Composite Entities Examples)

本系統使用 `PLUGIN_DYNAMIC_TABLES` 作為通用的擴充載體,但在邏輯上可以自由定義各式組合實體完成所有邏輯，以下為範例內容:

### 1. 電子發票組合實體 (Invoice Entity)

**定義:** 將「核心帳目 (Expense)」與「發票模組 (Invoice Plugin)」結合的實體。

**功能:** 解決核心表欄位固定的限制,讓特定消費紀錄可以掛載發票號碼、賣方統編與載具資訊。

**結構:** 包含外鍵 `expense_id` (指向該筆消費) 與 JSON 資料 `custom_data` (存放發票細節)。

### 2. 圖片附件組合實體 (Receipt Attachment Entity)

**定義:** 核心帳目 + 圖片/相機模組資料。

**功能:** 可以紀錄實體收據或商品照片作為憑證供檢閱。

**結構:** expense_id: 關聯至核心消費紀錄。實作細節可能如下：
```json
custom_data: { "localPath": "/storage/.../img.jpg", "thumbnail": "base64..." }。
```

---