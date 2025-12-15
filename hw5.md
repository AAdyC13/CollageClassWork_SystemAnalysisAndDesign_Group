# UML 設計文件 (Class, Sequence & Activity Diagrams)

本文件包含系統類別結構設計,以及針對三個核心使用案例的詳細流程設計。

---

## 1. UML 類別圖 (Class Diagram)

此圖表展示系統核心類別及其互動關係,包含前端管理器、原生橋接層與資料庫存取層。

```mermaid
classDiagram
    class Runtime {
        +init()
        +start()
    }

    class ModulesManager {
        +moduleDict: Map
        +init(logger)
        +loadSystemModules()
        +enableModules(modIds)
    }

    class PageManager {
        +PageElements: Map
        +init(preLoadPages)
        +createPage(pageID, layoutID)
    }

    class Bridge {
        +callSync(method, args)
        +saveData(key, value)
    }

    class AndroidBridge {
        <<Interface>>
        +getSystemModulesList()
        +getTechs(json)
    }

    class ExpenseRepository {
        +allExpenses: Flow
        +insert(expense)
        +delete(expense)
    }

    class ExpenseDao {
        <<Interface>>
        +getAllExpenses()
        +insertExpense(expense)
    }

    class Expense {
        +id: Int
        +amount: Double
        +description: String
        +timestamp: Long
    }

    %% Relationships
    Runtime --> ModulesManager : Initializes
    Runtime --> PageManager : Initializes
    Runtime --> Bridge : Uses
    
    ModulesManager --> Bridge : Calls Native
    Bridge --> AndroidBridge : Interacts via WebView
    
    ExpenseRepository --> ExpenseDao : Uses
    ExpenseDao ..> Expense : Manages
```

---

## 2. 使用案例:系統啟動與模組清單載入

**情境:** 應用程式啟動時,Runtime 初始化並透過 ModulesManager 向 Android 原生層請求系統模組清單。

### 循序圖 (Sequence Diagram)

```mermaid
sequenceDiagram
    participant Runtime
    participant ModulesManager
    participant Bridge (JS)
    participant AndroidBridge (Kotlin)
    participant BackgroundWorker
    participant AssetManager

    Runtime->>ModulesManager: init(logger)
    ModulesManager->>Bridge (JS): callSync('getSystemModulesList')
    Bridge (JS)->>AndroidBridge (Kotlin): getSystemModulesList()
    AndroidBridge (Kotlin)->>BackgroundWorker: worker.systemModulesList
    BackgroundWorker->>AssetManager: list("www/systemModules")
    AssetManager-->>BackgroundWorker: folderNames
    loop 讀取每個模組資訊
        BackgroundWorker->>AssetManager: open(".../info.json")
        AssetManager-->>BackgroundWorker: JSON Content
    end
    BackgroundWorker-->>AndroidBridge (Kotlin): List<Map> modules
    AndroidBridge (Kotlin)-->>Bridge (JS): JSON String (Encoded)
    Bridge (JS)-->>ModulesManager: Module List Object
    ModulesManager->>ModulesManager: 建立 Module 實例並存入字典
```

### 活動圖 (Activity Diagram)

```mermaid
flowchart TD
    Start((開始))
    InitRuntime[Runtime 初始化]
    InitManagers[初始化各管理器 ModulesManager, PageManager...]
    ReqModules[請求系統模組清單]
    CallNative[呼叫 Android 原生介面]
    ReadAssets[讀取 Assets 資料夾]
    CheckInfo[檢查 info.json 是否存在]
    
    CheckInfo -- 存在 --> ParseInfo[解析模組資訊]
    CheckInfo -- 不存在 --> SkipModule[跳過該資料夾]
    
    ParseInfo --> AddToList[加入模組清單]
    AddToList --> LoopEnd{所有資料夾讀取完畢?}
    SkipModule --> LoopEnd
    
    LoopEnd -- 否 --> ReadAssets
    LoopEnd -- 是 --> ReturnList[回傳模組列表至前端]
    
    CreateInstances[前端建立 Module 實例]
    End((結束))

    Start --> InitRuntime
    InitRuntime --> InitManagers
    InitManagers --> ReqModules
    ReqModules --> CallNative
    CallNative --> ReadAssets
    ReadAssets --> CheckInfo
    ReturnList --> CreateInstances
    CreateInstances --> End
```

---

## 3. 使用案例:啟用模組與讀取技術配置

**情境:** ModulesManager 決定啟用特定模組後,需讀取該模組的 tech.json 以獲取元件與依賴設定。

### 循序圖 (Sequence Diagram)

```mermaid
sequenceDiagram
    participant ModulesManager
    participant Bridge (JS)
    participant AndroidBridge (Kotlin)
    participant AssetManager

    ModulesManager->>ModulesManager: enableModules(modIds)
    ModulesManager->>Bridge (JS): callSync('getTechs', fileNames)
    Bridge (JS)->>AndroidBridge (Kotlin): getTechs(jsonString)
    
    loop 針對每個模組名稱
        AndroidBridge (Kotlin)->>AssetManager: open(".../tech.json")
        alt 檔案存在
            AssetManager-->>AndroidBridge (Kotlin): File Content
        else 檔案不存在
            AssetManager-->>AndroidBridge (Kotlin): Exception (Log Warning)
        end
    end

    AndroidBridge (Kotlin)-->>Bridge (JS): TechMap JSON String
    Bridge (JS)-->>ModulesManager: Tech Data Object
    ModulesManager->>ModulesManager: loadTech(data)
    ModulesManager->>ModulesManager: emit('MM:Module_enabled')
```

### 活動圖 (Activity Diagram)

```mermaid
flowchart TD
    Start((開始))
    IdentifyMods[確定需啟用的模組 ID 集合]
    RequestTech[向 Android 端請求 Tech 設定]
    IterateNames[遍歷模組資料夾名稱]
    
    ReadTechFile[讀取 tech.json]
    FileExists{檔案存在?}
    
    FileExists -- 是 --> ParseJson[解析 JSON 內容]
    FileExists -- 否 --> LogWarn[記錄警告並回傳空物件]
    
    ParseJson --> AddToMap[加入 Tech Map]
    LogWarn --> AddToMap
    
    AddToMap --> LoopCheck{所有模組處理完畢?}
    LoopCheck -- 否 --> IterateNames
    LoopCheck -- 是 --> ReturnData[回傳 Tech Map 至前端]
    
    LoadTech[前端載入 Tech 資料]
    UpdateStatus[更新模組狀態為 TechLoaded]
    EmitEvent[發送 Module_enabled 事件]
    End((結束))

    Start --> IdentifyMods
    IdentifyMods --> RequestTech
    RequestTech --> IterateNames
    IterateNames --> ReadTechFile
    ReadTechFile --> FileExists
    LoopCheck --> ReturnData
    ReturnData --> LoadTech
    LoadTech --> UpdateStatus
    UpdateStatus --> EmitEvent
    EmitEvent --> End
```

---

## 4. 使用案例:新增一筆記帳紀錄

**情境:** 使用者在 Recorder 介面輸入金額與資訊,並按下儲存按鈕。

### 循序圖 (Sequence Diagram)

```mermaid
sequenceDiagram
    participant User
    participant Recorder (UI)
    participant Calculator (Component)
    participant Bridge (JS)
    participant ExpenseDao (Kotlin)

    User->>Recorder (UI): 點擊「金額」欄位
    Recorder (UI)->>Calculator (Component): init(fieldId)
    User->>Calculator (Component): 輸入數字與運算
    Calculator (Component)-->>Recorder (UI): 回傳計算結果
    User->>Recorder (UI): 選擇類別/輸入備註
    User->>Recorder (UI): 點擊「儲存」按鈕
    Recorder (UI)->>Recorder (UI): saveRecord() 資料驗證
    Recorder (UI)->>Bridge (JS): callSync('saveData', expenseData)
    Bridge (JS)->>ExpenseDao (Kotlin): insertExpense(expense)
    ExpenseDao (Kotlin)-->>Bridge (JS): Success
    Bridge (JS)-->>Recorder (UI): Success
    Recorder (UI)->>Recorder (UI): navigate('pages/home.html')
```

### 活動圖 (Activity Diagram)

```mermaid
flowchart TD
    Start((開始))
    OpenRecorder[開啟記帳頁面]
    SelectType[選擇交易類型 支出/收入/轉帳]
    LoadSubInterface[載入對應子介面]
    
    InputAmount[點擊輸入金額]
    UseCalc[使用計算機元件]
    FillDetails[填寫類別、帳戶、備註]
    
    ClickSave[點擊儲存按鈕]
    Validate{資料驗證?}
    
    Validate -- 通過 --> CallSaveAPI[呼叫儲存 API]
    Validate -- 失敗 --> ShowError[顯示錯誤提示]
    ShowError --> InputAmount
    
    CallSaveAPI --> DBInsert[寫入資料庫]
    DBInsert --> NavigateHome[返回首頁]
    End((結束))

    Start --> OpenRecorder
    OpenRecorder --> SelectType
    SelectType --> LoadSubInterface
    LoadSubInterface --> InputAmount
    InputAmount --> UseCalc
    UseCalc --> FillDetails
    FillDetails --> ClickSave
    ClickSave --> Validate
    NavigateHome --> End
```

---