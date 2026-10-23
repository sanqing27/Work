# XKLAutoCheck 自检框架复用说明

> 目的：把「CATIA 侧自检命令 + Qt 侧界面」这套框架的实现方式沉淀下来，供后续 AI / 开发直接复用。
> 主要参考源码：`MDSToolkit\HWMDSFrm\XKLAutoCheckMdl.m\src\XKLAutoCheckCmd.cpp`（命令启动函数）与其头文件、Addin、公共依赖库。
> 本文档基于既有代码整理，未臆造接口；引用位置以 `文件:行号` 标注。

---

## 1. 框架定位与整体结构

这是一套 **CATIA CAA 命令（CATStateCommand）+ 隐藏 Win32 窗口 + Qt 界面进程** 的双进程协作框架：

- **CAA 侧**：一个自检命令（`XKLAutoCheckCmd`），被点击后瞬间执行 `BuildGraph()`，随后创建一个 **隐藏窗口** 常驻，通过窗口消息接收 Qt 侧指令，并把数据以 JSON 通过消息发回 Qt。
- **Qt 侧**：`MDS_UI_MAIN` 工程内的 `Qt_AutoCheck*` 系列窗口，负责展示节点树、检查项结果、高亮/关闭/复检等交互，并把用户操作发回 CAA。
- **进程间通信（IPC）**：Windows `WM_COPYDATA` / 自定义消息 + 命名共享内存（`Catia_MDS_Share`）；JSON 作为数据载体。

```
用户点击工具按钮
      │  (Addin 注册的 CmdHeader)
      ▼
XKLAutoCheckCmd 构造 → BuildGraph()（一次性，OneShot）
      │  参数 _ArgumentCmd = 1(执行自检) / 2(复检选项页)
      ▼
隐藏窗口 "AutoCheck_Catia" + WndProc 常驻
      │◄────────── WM_COPYDATA（Qt→CATIA，json 走共享内存）
      │───────────► MDS_Catia_Message::sendMsg("MDS_Qt_Message", json)（CATIA→Qt）
      ▼
SPQ 检查引擎逐条执行 → 结果回传 Qt → Qt 展示
```

---

## 2. 文件与模块分布

| 角色 | 路径 |
|---|---|
| 命令实现 | `MDSToolkit\HWMDSFrm\XKLAutoCheckMdl.m\src\XKLAutoCheckCmd.cpp` |
| 命令头文件 | `MDSToolkit\HWMDSFrm\PublicInterfaces\XKLAutoCheckCmd.h` |
| 模块导出宏 | `MDSToolkit\HWMDSFrm\PublicInterfaces\XKLAutoCheckMdl.h` |
| 命令注册（能表/工具栏） | `MDSToolkit\HWMDSPublicFrm\HWMDSAddinMdl.m\src\HWMDSAddin.cpp` |
| 消息通道 | `MDSToolkit\HWMDSPublicFrm\PublicInterfaces\MDS_Catia_Message.h` |
| 共享内存 | `MDSToolkit\HWMDSPublicFrm\PublicInterfaces\SharedMemory.h` |
| 检查项调度 | `MDSToolkit\HWMDSPublicFrm\PublicInterfaces\AutoCheckSPQManage.h` |
| 检查项实现（SPQ_*） | `MDSToolkit\HWMDSPublicFrm\PublicInterfaces\CatiaAutoCheckSPQCmd.h` |
| 几何/PLM 通用 API | `...\PublicInterfaces\CatiaAutoCheckSPQ_APICmd.h` |
| Qt 界面 | `MDS_UI_MAIN\MDS_UI_MAIN\Qt_AutoCheck*.{h,cpp,ui}` |

---

## 3. 命令的注册与启动

### 3.1 组件注册（CAA 工厂）
`XKLAutoCheckCmd.cpp:112`：

```cpp
CATCreateClassArg(XKLAutoCheckCmd, void*);
```

`XKLAutoCheckCmd` 继承 `CATStateCommand`，使用 `DeclareResource`、`ExportedByXKLAutoCheckMdl`（`XKLAutoCheckCmd.h:64`）。

### 3.2 能表命令头注册（入口）
`HWMDSAddin.cpp` 的 `HWMDSAddin::CreateCommands()` 中注册了两个命令头，分别对应两种模式：

```cpp
CATAfrCommandHeader::CATCreateCommandHeader("MDSAutoCheckHdr",        // 功能按钮标识
    "XKLAutoCheckMdl",                                                // 所在 module
    "XKLAutoCheckCmd", (void*)1,                                      // 自动检查模式
    ...
CATAfrCommandHeader::CATCreateCommandHeader("MDSAutoCheckReviewOptionSetHdr",
    "XKLAutoCheckMdl",
    "XKLAutoCheckCmd", (void*)2,                                      // 复检筛选/修改页面
```

`(void*)1 / (void*)2` 即构造函数的 `iArgument`，在 `XKLAutoCheckCmd.cpp:192` 被转成 `_ArgumentCmd = CATPtrToINT32(iArgument)`。

---

## 4. 构造函数做了什么（XKLAutoCheckCmd.cpp:152）

关键初始化顺序：

1. `MDS_Catia_Message::QueryAnd2SetupQt();` —— 初始化 Qt 通信。
2. 注册 Win32 窗口类 `kClassName = L"AutoCheck_Catia"`，创建 **100×100 隐藏窗口**，窗口过程为 `XKLAutoCheckCmd::WndProc`：
   ```cpp
   wc.lpfnWndProc = WndProc;
   wc.lpszClassName = kClassName;   // L"AutoCheck_Catia"
   hWnd_ = CreateWindowEx(..., WS_OVERLAPPEDWINDOW, ..., 100, 100, ...);
   ```
   该窗口是 **Qt→CATIA 消息的接收端点**。
3. `static CatiaAutoCheckSPQ_APICmd catiaAutoCheckSPQ_APICmd;`
4. `sql.InstanceServer();` 并 `std::thread t1(&XKLAutoCheckCmd::Userinitsql); t1.detach();` —— 后台线程预读自检配置。
5. `GetSecurityParameters("PLM1", ...)` 取当前用户 `userName`。
6. 初始化上报结构 `m_dat`（`MdsTracPointData`），`requestMethod="CATIA"`, `status="0"`, `responseStartTime`。

---

## 5. BuildGraph()：一次性分发（XKLAutoCheckCmd.cpp:206）

`BuildGraph()` 是命令的「启动函数」，执行完即 `RequestDelayedDestruction()` 退出命令态，隐藏窗口继续存活。

通用前置校验（任一失败即提示 + 上报 + 退出）：
1. `CheckVersionAvailable`（MDS 版本校验，当前被注释掉）。
2. `CATFrmEditor::GetCurrentEditor()` 为空 → 无模型提示。
3. 取 `GetUIActiveObject`，判断根对象是否为 `VPMReference` / `Piping_Rigid_Pipe`。
4. `CATSysCAALM::IsProductAuthorized("MES.prd")` 许可校验。

按 `_ArgumentCmd` 分流：

### 5.1 模式 2 —— 复检选项页（`XKLAutoCheckCmd.cpp:287`）
向 Qt 发 `AutoCheck_ReviewOptionSet` 创建窗口消息，然后上报并退出：
```cpp
sendJson["Head"]="AutoCheck_CheckNodeList";
sendJson["Feature"]="AutoCheck";
sendJson["Window"]="AutoCheck_ReviewOptionSet";
sendJson["Action"]="CreateWindow";
MDS_Catia_Message::sendMsg("MDS_Qt_Message", sendJson);
```

### 5.2 模式 1 —— 执行自检（`XKLAutoCheckCmd.cpp:307`）
1. 取根节点 `occ` 存 `_spRootOcc`，`GetRelatedReference` 取 `ref`。
2. 读根节点属性：`V_PartNumber / PLM_ExternalID / revision / V_Name / owner` → `rootEnterpriseNumber / rootPrdNumber / rootVerstion / rootVname / rootOwner`。
3. `GetFatherModel(...)` 取建模时参考版本 `m_strFatherVersion`。
4. `UpgradeDefinePermissionVer(...)` 校验业务线/权限，得到 `m_business`、`m_configHash`。
5. 命中业务线 → `ifOpenUpgradeDefine=true; Upgrade_Definition();`；否则 `BackFromUpgrade();`。

---

## 6. 自检主体流程

### 6.1 升级定义（Upgrade_Definition，XKLAutoCheckCmd.cpp:844）
查询表 `upgrade_definition_model_data`（`prd + version in (当前,参考) + if_open=1 + business_line`）：
- **无历史记录** → `GetAllDatas()` 收集模型数据，弹窗提示用户先建升级定义，发 Qt `UpgradeDefinition/CreateUpgradeDefinition`。
- **有当前版本记录** → 直接取 `id` 作为 `m_UpgradeDefineId`。
- **只有参考版本记录** → 用带参 SQL 把参考版本记录连同 `archive_type/product_type` 复制成当前版本记录（`insert ... if_open=1`），再查回 `id`。

### 6.2 回到自检（BackFromUpgrade，XKLAutoCheckCmd.cpp:575）
1. 取工程权限 `projectAuts`（`MDSHedsConfig::getconfig("MDS000001")` + `MDSUtility::PreCheck_*`）。
2. **收集待检节点**：`CatiaAutoCheckSPQCmd::SPQ_GetCheckNode(_spRootOcc, ioVec, ioSet, maxHeap, withPrototypeAndReleasedNode)`，结果在静态 `ioVec`。
3. **规范性预校验**：无企业编号、无 `lifecyclestagekey`、零件 BOM 无下级 List 的节点 → 发 Qt `AutoCheck_NoPartNumber` 窗口。
4. 正常则把 `ioVec` 逐条转 JSON（`part_number/prd_number/title/version/status/save_time/node_num/PBOM_CODE/lifecyclestagekey/owner/is3DShape`），发 Qt `AutoCheck_CheckNodeList/CreateWindow`。
5. 上报日志 `PostLog(HTTP_POST, InitMdsTracPointData(m_dat), ref)`。

### 6.3 批量检查（Check → UserCheckMain_batch，XKLAutoCheckCmd.cpp:1346 / 1515）
- `Check(data)`：从 `data["Data"]` 读选中节点下标 → `selNodeIndex`，`stable_sort` 使 **二维图/明细在前**（`CheckNodeCmp`），再批量执行。
- `UserCheckMain_batch()`：
  - 发进度 `userSendMsgtoProgress(progressJs, 2)`。
  - `CheckContinueRun()` 支持取消（读 `d:/workerlog.txt` + 命名互斥体 `Global\JsonOldMutex`；取消时 `exit(0)`）。
  - 预加载层级：`SPQ_API_GetAutoCheckNodeMapInfo(_spRootOcc, "1", allLevelKeyInfo, allLevelOccInfo)`。
  - 逐节点 `UserCheckMain_one`，并记录节点/根节点耗时 `UpNodeCheckTime / UpRootCheckTime`。

### 6.4 单节点检查（UserCheckMain_one，XKLAutoCheckCmd.cpp:1599）
对每个节点：
1. `GetCheckProductAndModelType(node)` 得节点类型字符串（如「装配体」「钣金件」「零件二维图」等）。
2. `SPQ_API_GetNodeLevelInfo` 取层级信息到 `NodeLeverInfo`。
3. 建目录 `D:\MDS\<企业编号>`；读 `d:\MDS\MdsAutoCheckIDs.txt`（存在时仅执行其中检查项，调试用）。
4. **筛选检查项**：
   - `GetCheckIdFromUpgradeDefine(m_listsCheckconfig)`：按 `upgrade_definition_model_data` 的 `archive_type/product_type` + 业务线，匹配 `pro_auto_check_config`。
   - `GetCheckIdFromPartAttribute(strPartAtttribute, m_listsCheckconfig)`：再按加工属性（`PRODUCT_TYPE`）过滤。
5. 遍历 `sqlData`（`pro_auto_check_config` 预读结果），逐条过滤：`is_on_off`、`model_type`、检查项白名单。
6. **执行检查项**：`SPQManage.Usermain(sqlData[i][0]/*check_id*/, sqlData[i][4]/*adapter_name*/, node, js)`，其中 `js["PrdType"]`、`js["PrdNumber"]` 为上下文。
7. 结果发 Qt `AutoCheck_CheckNodeList/Check`（`CheckID` + `JsonData`），并落盘 `D:\MDS\<企业编号>\<check_id>.json`。
8. 节点完事发 `FinishOneNode`。记录检查项耗时 `UpCheckItemTime`。

### 6.5 复检（ReCheck，XKLAutoCheckCmd.cpp:2466）
与单节点检查类似，额外重新加载全局层级图信息，只针对 `data["Index"]` 指定节点，完成后 `ToSendFlashMessage(true, memoryjson)`。

---

## 7. 命令常驻窗口与指令分发表（WndProc，XKLAutoCheckCmd.cpp:368）

`WndProc` 处理两类消息：

- `WM_COPYDATA`：Qt→CATIA 指令。
- `WM_FLASH_SAVE_TIME`（`WM_USER+1`）：保存/刷新时间相关通知。

处理 `WM_COPYDATA` 时：
1. 从共享内存取 JSON：
   ```cpp
   std::string memorystr = (char*)SharedMemory_Global::GetControl()->GetMemPtr();
   reader3.parse(memorystr, memoryjson);
   ```
2. 判断 `memoryjson["Function"] == "AutoCheck"`，按 `memoryjson["Cmd"]` 分发：

| Cmd | 处理函数 | 说明 |
|---|---|---|
| `BackFromUpgrade` | `BackFromUpgrade()` | 从升级定义返回；读取 `UpgradeDefineId` |
| `ClickBatchCheck` | `Check()` → `ToSendFlashMessage(false)` | 批量执行自检 |
| `PromptInfoSet` | 存 `m_productType` | 提示信息/产品类型 |
| `CloseQuestionNode` | `CloseNode()` | 关闭节点窗口 |
| `OpenQuestionNode` | `OpenNode()` | 打开节点 |
| `HighLight` | `HighLightQuestion()` | 高亮问题（含二维图 `CatiaApiCmd::HighLightDrw`） |
| `CloseCmd` | `CatiaApiCmd::ClearHighLight()` | 命令关闭时清高亮 |
| `Recalculate` | `ReCheck()` → `ToSendFlashMessage(true)` | 单项复检 |
| `ReviewOptionsSetting` | 存 `withPrototypeAndReleasedNode` | 复检选项 |
| `OneClickModify_ModifyModel` | `ModifyModels()` | 一键修改模型 |

`WM_FLASH_SAVE_TIME`：读共享内存中 `"true"/"false"` 决定 `RefreshCheckTime(true/false, memoryjson)`；注意此处会 `delete pCDS->lpData; delete pCDS;`。

---

## 8. 双向通信机制（重点，复用核心）

### 8.1 CATIA → Qt
统一走 `MDS_Catia_Message`：
```cpp
MDS_Catia_Message::sendMsg("MDS_Qt_Message", sendJson);            // 一般数据/创建窗口
MDS_Catia_Message::userSendMsgtoProgress(progressJs, 2);           // 进度条
```
JSON 通用约定字段：`Feature="AutoCheck"`、`Window`（目标窗口）、`Action`（CreateWindow/Check/FinishOneNode/ShowWindow/RefreshTable/InitBackAutoCheck 等）。

### 8.2 Qt → CATIA
- 端点窗口：类名 `L"AutoCheck_Catia"`，标题 `L"AutoCheck_Catia"`（`ToSendFlashMessage` 用 `FindWindow(nullptr, L"AutoCheck_Catia")` 查找）。
- 数据放 **命名共享内存** `Catia_MDS_Share`（`SharedMemory_Global`，容量常量 `MAP_SIZE`），`PutInData(json字符串, len)`。
- 通过 `WM_COPYDATA`（或 `WM_FLASH_SAVE_TIME`）通知；`COPYDATASTRUCT` 仅携带短标记字符串，真正的 JSON 在共享内存里，接收端 `GetMemPtr()` 直接读。

`ToSendFlashMessage`（`XKLAutoCheckCmd.cpp:2927`）示例：
```cpp
SharedMemory_Global::GetControl()->PutInData((char*)indata.toStyledString().c_str(), len+1);
COPYDATASTRUCT* copyData = new COPYDATASTRUCT;
copyData->cbData = (strlen(buffer)+1)*sizeof(TCHAR);
copyData->lpData = (void*)buffer;                 // "true"/"false"
::PostMessage(hWnd, WM_FLASH_SAVE_TIME, NULL, (LPARAM)copyData);
```
> 之所以用共享内存 + PostMessage 而非直接 `WM_COPYDATA` 大 payload，是因为直接 `SendMessage` 传大数据在 CATIA 关闭时可能异常（见 `ToSendFlashMessage` 注释）。

### 8.3 共享内存要点（SharedMemory.h）
- 名称固定 `L"Catia_MDS_Share"`；`Create` 失败则 `Open` 复用已有映射（首进程创建、后续进程打开）。
- 有 `Lock/Unlock`（命名互斥体 `<name>_Lock`）可选加锁。
- 最小映射大小 `min_mem_size = 500000`。

---

## 9. 关键数据结构

| 结构 | 位置 | 用途 |
|---|---|---|
| `AutoCheckNode` | `CatiaAutoCheckSPQCmd.h:78` | 待检节点：`pNode / nodeType / strNodeNumber / strNodeEnterpriseNumber / strVerstion / strLifecyclestagekey / strOwner / iNodeNum / lSaveTime / parameters` |
| `AUTO_CHECK_TYPE`（枚举） | `CatiaAutoCheckSPQCmd.h:67` | 装配/钣金装配/钣金件/零件/装配二维图/装配明细/零件二维图/BOM/3DShape |
| `CheckConfig` | `XKLAutoCheckCmd.h:48` | 检查项配置：`checkid/businessline/archive_type/product_type/part_attribute` |
| `inSPQExtendedData` | `CatiaAutoCheckSPQCmd.h:92` | 检查项扩展数据：节点、`allASMOccVec`、`allFeatureVec`、`allHoleInfoVec` |
| `MdsTracPointData` | `MDS_Catia_Message.h:43` | 埋点/上报字段（title/businessType/method/status/errorMsg/响应时间等） |
| `UpNodeInfo` | `XKLAutoCheckCmd.h:41` | 上报节点耗时用 |

`XKLAutoCheckCmd` 大量使用 **static 成员** 承载跨回调共享状态（`ioVec / ioSet / selNodeIndex / sqlData / _spRootOcc / userName / root* / m_business / m_configHash / m_listsCheckconfig / m_UpgradeDefineId / withPrototypeAndReleasedNode` 等，见 `XKLAutoCheckCmd.h:369` 起）。

---

## 10. 检查项调度（AutoCheckSPQManage）

`AutoCheckSPQManage::Usermain(checkId, allSPQ, node, allJson)`（`AutoCheckSPQManage.h:53`）：
- 内部维护 `std::map<std::string, SPQ_Func> SPQfuncMap`，`SPQ_Func` 签名：
  ```cpp
  typedef bool (*SPQ_Func)(Json::Value& orgJson, Json::Value& inPutSpq,
                           Json::Value& outList, inSPQExtendedData& inData);
  ```
- 真正的检查项实现是一批 `CatiaAutoCheckSPQCmd::SPQ_XXX(...)` 静态函数（`CatiaAutoCheckSPQCmd.h`），几何/PLM 通用算法在 `CatiaAutoCheckSPQ_APICmd::SPQ_API_XXX(...)`。
- `allRunData` 缓存同一次检查内已执行的 SPQ 结果，供后续检查项复用。

### 新增一个检查项的最小步骤
1. 在 `CatiaAutoCheckSPQCmd` 增加静态函数，签名固定为 `(Json::Value&, Json::Value&, Json::Value&, inSPQExtendedData&) -> bool`，返回 `true` 表示「符合要求」。
2. 在 `AutoCheckSPQManage` 的映射表里登记 `检查名 -> 函数指针`。
3. 数据库 `pro_auto_check_config` 增加一行：`check_id / adapter_name(SPQ名) / model_type / product_config / is_on_off / auto_check_object`。
4. 若需前端展示，在 Qt 侧对应窗口解析回传的 `JsonData`。

---

## 11. 数据库与配置接口

- **数据库**：`MDSDataBaseServer`（`sql` 全局对象，`InstanceServer()` 建立连接）。
  - 查询：`LinkDataTable(_variant_t(sql))` → `GetmRecordPtr()->RecordCount` → 遍历字段。
  - 参数化写入：`_CommandPtr + CreateParameter(adVarWChar, adParamInput, ...)`，`adCmdText` 执行；异常 `_com_error` 回滚。
  - 关键表：`pro_auto_check_config`、`upgrade_definition_model_data`、`pro_auto_check_root_part_time`、`pro_auto_check_part_time`、`pro_auto_check_item_time`、`base_config_info`。
- **配置**：`MDSHedsConfig::getconfig(CATUnicodeString toolname, map)`；`MDSUtility::PreCheck_*` 权限工具。
- **UI/消息**：`CAAPublicService::MessageBoxShow / GetObjectTypeName / OpenPLMComponent / GetEnvValue`。
- **许可**：`CATSysCAALM::IsProductAuthorized("MES.prd")`。

---

## 12. 日志与埋点

- 日志宏（`MDS_Catia_Message.h:125` 起）：`Debug() / Info() / Warn() / Error() / Critical()`，支持 `<<` 链式；带埋点的 `Info_OnlineLog(dat) / Warn_OnlineLog(dat) ...`。
- 埋点上报：填 `MdsTracPointData` → `MDS_Catia_Message::InitMdsTracPointData(m_dat)` → `MDS_Catia_Message::PostLog(HTTP_POST, str, ref)`。
- 本地调试日志：`d:/mdslog.txt`（检查项耗时）、`d:/workerlog.txt`（取消状态）。

---

## 13. 复刻本框架的清单（Checklist）

要新做一套「Qt 驱动的 CATIA 命令」，按以下顺序即可：

1. **建模块**：`XXXMdl.m` + `PublicInterfaces/XXXMdl.h`（导出宏）+ 头/实现。
2. **命令类**：继承 `CATStateCommand`，`DeclareResource`，`CATCreateClassArg(XXXCmd, void*)`。
3. **构造**：`QueryAnd2SetupQt()` → 注册并创建隐藏窗口（窗口过程 `WndProc`）→ 后台线程初始化数据。
4. **BuildGraph**：用 `CATPtrToINT32(iArgument)` 分流；做编辑器/对象/许可校验；发 Qt 创建窗口消息或执行主逻辑；`RequestDelayedDestruction()`。
5. **通信**：CATIA→Qt 用 `MDS_Catia_Message::sendMsg("MDS_Qt_Message", json)`；Qt→CATIA 用 `WM_COPYDATA` + 共享内存 `Catia_MDS_Share`。
6. **指令分发**：在 `WndProc` 里按 `Function/Cmd` 分发到静态处理函数。
7. **Addin 注册**：在 `HWMDSAddin::CreateCommands()` 加 `CATCreateCommandHeader(...)`，并在资源/工具栏 `SetAccessCommand`。
8. **Qt 侧**：在 `MDS_UI_MAIN` 建对应窗口，按 `Window/Action` 响应，用户操作回传 `Cmd`。

---

## 14. 注意事项 / 常见坑

- **编码**：CAA 源码（含中文注释）为 **GBK**；Qt 源码/`.ui` 为 **UTF-8**。跨进程 JSON 文本注意 `Utf8ToGbk / gb2312_to_utf8`（`MDS_Catia_Message.h:218`）。
- **窗口标识必须两端一致**：类名/标题 `AutoCheck_Catia`、共享内存名 `Catia_MDS_Share`。
- **命令不阻塞主流程**：真正的重活通过隐藏窗口消息驱动，命令本身 `OneShot` 即退。
- **静态成员即全局状态**：多实例/重入需谨慎（框架按单次流程设计）。
- **取消机制**：`UserCheckMain_batch` 里 `CheckContinueRun()` 为假时 `exit(0)`，属硬退出，慎改。
- **JSON 库**：`#include "../../../include/json/json.h"`，并 `#pragma comment(lib, "...\\jsoncpp.lib")`（`XKLAutoCheckCmd.cpp:111`），路径为硬编码，移植时需调整。
- **COM 变量管理**：CAA 接口 `_var` 与 `DECLARE_COM_GUARD(...)`（`ComRAIIUtils.h`）配合使用，避免泄漏；`GetRelatedReference` 返回裸指针时判空后再用。
- **SQL 失败即打日志再 continue/return**，与项目规范一致。

---

## 15. 附录：关键调用速查

```cpp
// 取根节点
CATFrmEditor* Frm = CATFrmEditor::GetCurrentEditor();
CATPathElement pe = Frm->GetUIActiveObject();
pRootProduct->QueryInterface(IID_CATIPLMNavOccurrence, (void**)&piRootOcc);
piRootOcc->GetRelatedReference(piNavReference);

// 读属性
CATCkeObjectAttrReadServices::GetValueAsString(ref, "PLM_ExternalID", str);

// 收集待检节点
CatiaAutoCheckSPQCmd::SPQ_GetCheckNode(rootOcc, ioVec, ioSet, maxHeap, withPrototypeAndReleased);

// 执行检查项
SPQManage.Usermain(checkId, adapterName, node, js);

// 回传 Qt
MDS_Catia_Message::sendMsg("MDS_Qt_Message", sendJson);
```

---

## 16. Qt 侧对接详解（MDS_UI_MAIN）

自检是 **CATIA 内嵌命令 + 独立 Qt 进程** 的双进程架构。CATIA 侧只负责取模型数据、执行需要 CAA 接口的几何/PLM 检查、高亮/打开节点；UI 展示、问题列表、Excel 导出、线程池调度都在 Qt 进程 `MDS_UI_MAIN.exe` 里完成。

### 16.1 进程生命周期

- **启动**：CATIA 端 `MDS_Catia_Message` 构造时（`MDS_Catia_Message.cpp:427`）：
  1. `FindWindow(NULL, L"MDS_Catia_Message_Cmd")` 判重 —— 已存在则提示「不支持多个 Catia 同时打开」并置 `bHaveMdsTool=false` 直接返回；
  2. `system("start MDS_UI_MAIN.exe " + strQtLogPath)` 拉起 Qt 进程；
  3. `RegisterClassEx` + `CreateWindowEx` 建一个隐藏窗口，类名 `kClassName`、标题 `L"MDS_Catia_Message_Cmd"`。
- **退出**：析构（`MDS_Catia_Message.cpp:494`）若 `bHaveMdsTool`，发 `{Head:"Close_Qt_Exe"}` 给 `MDS_Qt_Message` 让 Qt 自杀；并 `taskkill /im QtProgressBar.exe /f`。
- 注意：CATIA 与 Qt 是**两个进程**，因此所有交互都退化为「窗口消息 + 共享内存」，不能用进程内指针/信号槽。

### 16.2 Qt 主窗口与消息接收

- Qt 端主窗口标题固定为 **`MDS_Qt_Message`**（即 CATIA `sendMsg("MDS_Qt_Message", ...)` 的目标）。
- 主窗口重写 `nativeEvent`，收到 `WM_COPYDATA` 后：
  1. 从命名共享内存 `Catia_MDS_Share`（`SharedMemory_Global::GetControl()->GetMemPtr()`）读出 JSON 文本（`COPYDATASTRUCT.lpData` 只带短标记，真正数据在共享内存）；
  2. 解析 JSON 顶层字段，按 `Head` / `Feature` 分发。自检走 `Feature == "AutoCheck"` → `MDS_UI_MAIN::AutoCheckCmd(memoryjson)`（`MDS_UI_MAIN.cpp:1430`）。
- 返回 Qt→CATIA 侧统一用 `MDS_Qt_Message::userSendMsg(窗口标题, json)`（见 16.4）。

### 16.3 AutoCheckCmd：Window × Action 分发表（`MDS_UI_MAIN.cpp:1430`）

`AutoCheckCmd` 只做两件事：取 `inData["Window"]` 与 `inData["Action"]`，然后分发。

| Window | Action | 处理 |
|---|---|---|
| `AutoCheck_NoPartNumber` | `CreateWindow` | `Create_QtShowAutoCheckNoPartNumber()` 无企业编码窗口 |
| `AutoCheck_ReviewOptionSet` | `CreateWindow` | `Create_Qt_AutoCheckOptionsSet()` 审核选项设置窗口 |
| `AutoCheck_Prompt` | `CreateWindow` | `Create_QtAutoCheckPrompt()` 升级定义/提示窗口 |
| `AutoCheck_CheckNodeList` | `CreateWindow` | `All_Data` 为空则弹提示；否则设 `AutoCheckMain::m_upgradeDefineId`、`checkQuestionInfo`，`Create_QtShowAutoCheckNodeList()` |
| `AutoCheck_CheckNodeList` | `ShowWindow` | `Qt_AutoCheckNodeListDlg->show()/activateWindow()` |
| `AutoCheck_CheckNodeList` | `Check` | `threadPool.addOneTask(CheckFunMap["Check"+CheckID], JsonData)` |
| `AutoCheck_CheckNodeList` | `FinishOneNode` | 等线程池 `IsAllTaskFinish()` → `AutoCheckMain::WriteDataFunc()` → `VecQuestion.clear()` → `FinishRowData()` |
| `AutoCheck_CheckNodeList` | `RefreshCheckTime` | `FinishAllNodeTime()` |
| `AutoCheck_CheckNodeList` | `InitBackAutoCheck` | 重建 `Qt_AutoCheckNodeList`（后台自检初始化） |
| `AutoCheck_CheckQuestionList` | `CreateWindow` | `Create_QtShowAutoCheckQuestionList()` 问题明细窗口 |
| `AutoCheck_CheckQuestionList` | `FinishOneNode` | 等线程池 → `WriteDataFunc` → `FinishShowInfo()` |
| `AutoCheck_OneClickModify` | `CreateWindow` | `Create_QtOneClickModifyList()` 一键修改列表 |
| `AutoCheck_OneClickModify` | `RefreshTable` | `Qt_OneClickModifyListDlg->RefreshQuesStatus()` |

要点：
- **窗口对象是全局单例指针**（`Qt_AutoCheckNodeListDlg`、`Qt_AutoCheckQuestionListDlg`、`Qt_OneClickModifyListDlg` 等），重复 `CreateWindow` 前一般先判空或 `deleteLater`。
- **检查项在 Qt 侧执行**：`Check` 把 `CheckFunMap[CheckFun]`（`CheckFun` = `void(*)(Json::Value)`，见 `AutoCheckMain.h:14`）丢进线程池，`CheckID` 对应 `AutoCheckMain::CheckV1_00001_XXXX` 系列静态函数；CATIA 侧只回传数据 JSON。
- `FinishOneNode` 会**阻塞等待线程池**（`while(!IsAllTaskFinish()) Sleep()`），再统一写库/刷新。

### 16.4 Qt → CATIA：userSendMsg 与 Cmd 表

Qt 端调用（`MDS_Qt_Message.cpp`）：
```cpp
MDS_Qt_Message::userSendMsg("AutoCheck_Catia", sendJson);      // 目标为 CAA 隐藏窗口
MDS_Qt_Message::userSendMsgtoProgress(progressJs);             // 进度条，目标窗口 MDS_Progress_Message
```
`sendJson` 固定带 `Function="AutoCheck"`，具体命令放 `Cmd`。CATIA 端在 `XKLAutoCheckCmd::WndProc`（`XKLAutoCheckCmd.cpp:396`，`strFunction=="AutoCheck"`）里按 `Cmd` 分发：

| Cmd | 发起窗口 | CATIA 处理 |
|---|---|---|
| `ClickBatchCheck` | Qt_AutoCheckNodeList | `Check()` 执行批量自检 → `ToSendFlashMessage(false)` |
| `PromptInfoSet` | QT_AutoCheckPrompt | 记录 `m_productType` |
| `CloseQuestionNode` | Qt_AutoCheckQuestionList | `CloseNode()` |
| `OpenQuestionNode` | Qt_AutoCheckNodeList | `OpenNode(OpenNode, OpenVerstion)` 打开节点 |
| `HighLight` | Qt_AutoCheckQuestionList | `HighLightQuestion()` 高亮 |
| `CloseCmd` | Qt_AutoCheckNodeList | `CatiaApiCmd::ClearHighLight()` 清理高亮 |
| `Recalculate` | Qt_AutoCheckQuestionList | `ReCheck()` 复检 → `ToSendFlashMessage(true)` |
| `OneClickModify_ModifyModel` | Qt_OneClickModifyList | `ModifyModels()` 一键修改模型 |
| `OneClickModifyGetModelInfo` | NodeList / QuestionList | 取模信息（辅助一键修改） |
| `ReviewOptionsSetting` | Qt_AutoCheckOptionsSet | 更新 `withPrototypeAndReleasedNode` |
| `BackFromUpgrade` | （升级窗口） | `BackFromUpgrade()` 携带 `UpgradeDefineId` 回自检 |

> 反向通知端点：CAA 隐藏窗口 **类名 `L"AutoCheck_CatiaCmd"`、标题 `L"AutoCheck_Catia"`**（`XKLAutoCheckCmd.cpp:114` / `:167`）。Qt 必须用**标题** `AutoCheck_Catia` 去 `FindWindow`。

### 16.5 谁在真正「管进程内状态」

- CATIA 进程内状态：`XKLAutoCheckCmd` 的 static 成员（`ioVec/ioSet/...`）。
- Qt 进程内状态：`AutoCheckMain` 的 static 成员（`VecQuestion`、`checkQuestionInfo`、`m_upgradeDefineId`、`m_strarchive_type` 等）。
- 二者**不共享内存对象**，只共享**JSON 文本**（经共享内存 `Catia_MDS_Share` 传输）。

### 16.6 复刻「Qt 侧」的最小清单

1. Qt 主窗口标题必须 = CATIA 端 `sendMsg` 的目标标题；重写 `nativeEvent` 处理 `WM_COPYDATA`，从共享内存读 JSON。
2. 按 `Head` / `Feature` 建一级分发表，`Feature=="AutoCheck"` → `AutoCheckCmd`。
3. `AutoCheckCmd` 内按 `Window` × `Action` 二次分发（见 16.3）。
4. 用户操作回调统一用 `MDS_Qt_Message::userSendMsg("<CAA窗口标题>", {Function:"AutoCheck", Cmd:"..."})`。
5. 耗时任务用线程池，`FinishOneNode` 前必须等待任务完成再写数据。
6. 进程退出：响应 `{Head:"Close_Qt_Exe"}` 自退出。
