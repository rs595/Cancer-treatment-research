flowchart TD
    A["用戶揀文件<br/>客戶端檢查類型／大小（只為方便用戶）"] --> B["requestUpload<br/>requireUser + 解析 parent + 檢查權限"]
    B --> C{"新版本？"}
    C -- 係 --> C1{"expectedVersion<br/>同 documents.version 相符？"}
    C1 -- 唔相符 --> X1["返回 409"]
    C1 -- 相符 --> D
    C -- 新文件 --> D["建立 upload_intent<br/>綁定操作者／目標／版本／checksum／到期時間"]
    D --> E["返回預簽名 PUT URL（≤ 15 分鐘）"]
    E --> F["瀏覽器直接 PUT 到 MinIO<br/>寫入 quarantine key，顯示進度"]
    F --> G["confirmUpload<br/>重新檢查意圖、操作者、當前權限"]
    G --> G0{"意圖已經 confirmed？"}
    G0 -- 係（重試） --> R["返回現有版本（冪等）"]
    G0 -- 否 --> H{"HEAD 大小／MIME／checksum 相符<br/>+ 惡意軟件掃描通過？"}
    H -- 否 --> X2["意圖標記 failed<br/>唔發佈任何內容"]
    H -- 是 --> I["server-side copy<br/>到不可變嘅最終 key"]
    I --> J["單一交易：<br/>documents + document_versions<br/>+ document_links + logActivity"]
    J --> J1{"併發衝突？"}
    J1 -- 係 --> X1
    J1 -- 否 --> K["文件已發佈"]
    X2 -.-> CL["cleanup script<br/>只刪除過期、未確認嘅 quarantine 物件"]
    K --> L["requestDownload<br/>檢查所有受限 context（包括間接 context）"]
    L -- 無權限 --> X3["403／只顯示 stub"]
    L -- 有權限 --> M{"confidential／deal_sensitive？"}
    M -- 是 --> N["記錄 document.viewed"]
    N --> O["返回預簽名 GET URL（≤ 10 分鐘）"]
    M -- 否 --> O
