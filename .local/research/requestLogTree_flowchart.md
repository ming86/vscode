---
title: requestLogTree Data Flow & Architecture
---

flowchart TD
    A["👤 User sends Chat Message"] --> B["💬 Conversation creates ChatRequest"]
    B --> C["📝 IRequestLogger.addEntry called"]
    C --> D["🔧 Creates LoggedRequestInfo<br/>Assigns unique ID<br/>Stores in _entries array"]
    D --> E["📢 Fires onDidChangeRequests event"]
    E --> F["🌳 ChatRequestProvider notified<br/>Rebuilds tree structure"]
    F --> G["📊 Displays tree in Copilot Chat panel"]

    G --> H{User Action?}
    H -->|Clicks tree item| I["🔗 Opens URI: ccreq:id.copilotmd"]
    H -->|Right-clicks| J["💾 Export/Save options"]

    I --> K["📄 VS Code requests document content"]
    K --> L["🔍 Text Document Provider invoked"]
    L --> M["🎯 ChatRequestScheme.parseUri extracts ID"]
    M --> N["🔎 Finds entry in _entries array"]
    N --> O["✨ _renderRequestToMarkdown generates content"]
    O --> P["📋 Formats Metadata+Request+Response"]
    P --> Q["🖥️ Virtual document displayed in editor<br/>Shows as .copilotmd file"]

    J --> R["💾 exportLogItemCommand"]
    J --> S["💾 saveCurrentMarkdownCommand"]
    J --> T["📦 exportPromptArchiveCommand"]

    R --> U["💿 Save single item to disk"]
    S --> U
    T --> V["🗜️ Create tar.gz archive<br/>with multiple items"]

    U --> W["📁 File saved on disk"]
    V --> W

    style A fill:#e1f5ff
    style Q fill:#c8e6c9
    style W fill:#fff9c4
    style O fill:#f3e5f5
    style L fill:#f1f8e9

    classDef memory fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef disk fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    classDef process fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef ui fill:#e0f2f1,stroke:#00897b,stroke-width:2px

    class D memory
    class N memory
    class W disk
    class O process
    class Q ui
