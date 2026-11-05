---
name: caa-api-discovery
description: Use when a CAA API is missing from public headers but the native CATIA/3DEXPERIENCE feature clearly performs the operation, or when you need to discover internal/undocumented CAA services by scanning installed headers, CAADoc HTML, NLS resources, and DLL exports (dumpbin/undname/GetProcAddress). Triggers: "头文件里没有这个接口", "原生能删但CAA不能", "查DLL导出", "内部服务", "dumpbin", "GetProcAddress".
---

## 目的与边界
- 主场景（CAA）：公开接口不满足，需要反查 CATIA 原生功能实际调用的**内部/未导出 CAA 服务**。
- 辅场景：对任意 DLL 做导出符号/签名反查（作为佐证手段）。
- 与 `caa-developer` 分工：先用 `caa-developer` 查公开文档；本 skill 负责「公开没有 → 找内部实现」。
- 铁律：**只认 `dumpbin` 实际看到的导出符号**；文档提及 ≠ 可用；无导出不得臆造。

## 前置与环境
- CAA 安装根：`D:\DS\B424`
  - 头文件：`<Framework>\PublicInterfaces`、`<Framework>\ProtectedInterfaces`
  - 运行库：`win_b64\code\bin`（DLL）、`win_b64\code\lib`（.lib）、`win_b64\code\include`
  - 消息资源：`win_b64\resources\msgcatalog\*.CATNls|*.CATRsc`（含多语言子目录）
  - 官方文档：`CAADoc\win_b64.doc\...`、`CAADoc\*.edu`（示例源码）
- 工具：`dumpbin` / `undname`。先 `Get-Command`，找不到再用 VS 路径：
  ```powershell
  $vs = Get-ChildItem 'C:\Program Files*\Microsoft Visual Studio' -Recurse -Filter dumpbin.exe -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
  ```
- 编码：CAA 源码 GBK；Qt UTF-8；只读扫描用 `errors='ignore'`。

## 流程（按序）
### 0. 先确认公开接口确实没有
- 按 `caa-developer` 查公开文档与 PublicInterfaces 头。
- 公开无、原生有 → 进入反查，并明确“这不是公开 API”。

### 1. 扫头文件（Public + Protected）
```python
import os
root=r'D:\DS\B424'
needles=['DeleteVPMDocument','PLMVPMDocDeleteServices']
for dp,dn,fn in os.walk(root):
    if ('PublicInterfaces' not in dp) and ('ProtectedInterfaces' not in dp): continue
    for f in fn:
        if not f.lower().endswith(('.h','.hpp')): continue
        p=os.path.join(dp,f); d=open(p,encoding='utf-8',errors='ignore').read()
        for i,l in enumerate(d.splitlines(),1):
            if any(n in l for n in needles): print(p,i,l.strip())
```
- 结论要区分：真没有 vs 同名类在其它 framework。

### 2. 扫 CAADoc HTML（文档可能提到未导出 API）
- 在 `CAADoc` 下检索关键类/方法名，读命中的 `.htm` 文本。
- 例：文档写了 `PLMDocumentServices::DeleteDocument`，头文件与 DLL 都没有 → 记“文档有、实现未暴露”。

### 3. 扫 NLS 资源（从 UI 提示反推功能与模块）
- 在 `win_b64\resources\msgcatalog` 搜同时含 `delet` 与 `database` 的英文行（排除本地化子目录）。
- 例：`OrphanPCDocDeleteConfirmation = "...permanently delete it from the database?"` → “孤儿文档删除”流程。
- 例：`CATMfgNCDocsPanelWidget.DeleteConfirmationMsg` → 同类流程旁证。

### 4. 定位实现 DLL
- 先按模块/关键词在 `win_b64\code\bin` 找 DLL。
- 再用符号字节扫描缩小范围（快）：
```python
import os
root=r'D:\DS\B424\win_b64\code\bin'; needle=b'PLMVPMDocDeleteServices'
for f in os.listdir(root):
    if not f.lower().endswith(('.dll','.exe')): continue
    if needle in open(os.path.join(root,f),'rb').read(): print(f)
```

### 5. 列导出 + 反修饰
```powershell
& $dumpbin /exports 'D:\DS\B424\win_b64\code\bin\XXX.dll' | Select-String -Pattern 'Delete|Remove|Foo'
& $undname '?DeleteVPMDocument@PLMVPMDocDeleteServices@@SAJVCATListPtrCATIAdpPLMIdentificator@@@Z'
```
- 过滤编译器生成的 `??0/??1/??4/??_7`（构造/析构/赋值/vftable）。
- 读参数传递方式：`PEAV...`=指针、`V...`=按值、`AEAV...`=引用。

### 6. 向上找调用方（追封装）
```powershell
& $dumpbin /imports 'D:\DS\B424\win_b64\code\bin\CATImmDocCommon.dll' | Select-String 'DeleteVPMDocument'
```
- 被 import 的模块通常是上层封装，可能提供更简单的单个对象入口。

### 7. 判断可用性
- 有头文件 + `.lib` → 可正常链接。
- 无头文件/无 `.lib`、仅导出 → 只能 `LoadLibrary + GetProcAddress` 动态调用（内部/未公开，版本脆弱）。
- 有同名公开类不代表具备该能力；以导出符号为准。

### 8. 动态调用样例（无头文件时）
```cpp
HMODULE h = GetModuleHandleA("PLMDmtDocumentDCCP.dll");
bool loaded = false;
if (h == NULL) { h = LoadLibraryA("PLMDmtDocumentDCCP.dll"); loaded = (h != NULL); }
if (h == NULL) { Info() << "load failed, err=" << (int)GetLastError(); return E_FAIL; }

typedef HRESULT (*PFN_DeleteVPMDocument)(CATListPtrCATIAdpPLMIdentificator);
PFN_DeleteVPMDocument pfn = (PFN_DeleteVPMDocument)GetProcAddress(
    h, "?DeleteVPMDocument@PLMVPMDocDeleteServices@@SAJVCATListPtrCATIAdpPLMIdentificator@@@Z");
if (pfn == NULL) { Info() << "GetProcAddress failed, err=" << (int)GetLastError(); if(loaded) FreeLibrary(h); return E_FAIL; }

HRESULT hr = pfn(docIds);
if (loaded) { FreeLibrary(h); }
```

### 9. 输出结论表（必须）
| 符号 | DLL | 反修饰签名 | 头文件/库 | 可用方式 | 风险 |
|---|---|---|---|---|---|
| `PLMVPMDocDeleteServices::DeleteVPMDocument` | `PLMDmtDocumentDCCP.dll` | `static HRESULT DeleteVPMDocument(CATListPtrCATIAdpPLMIdentificator)` | 无/无 | 动态调用 | 内部、版本脆弱 |

## 案例：Document 永久删库
1. 公开 `PLMDocumentServices` 无删除 → CAADoc 提到 `DeleteDocument`。
2. 头文件全盘检索：无 `DeleteDocument` 声明。
3. `dumpbin /exports PLMDocumentInterfacesItf.dll`：无 `DeleteDocument`；仅 Attach/Detach/Create/Get/List/GetAttr/GetRevisions + `DeleteFileW`。
4. NLS 命中 `OrphanPCDocDeleteConfirmation` → 原生“解绑后变孤儿→询问永久删库”。
5. 字节扫 `PLMVPMDocDeleteServices` → 命中 `PLMDmtDocumentDCCP.dll`、`CATImmDocCommon.dll`。
6. `PLMDmtDocumentDCCP.dll` 导出 `DeleteVPMDocument(列表按值)` + `CATIPLMDeleteUI` 工厂。
7. `CATImmDocCommon.dll` 导出更高层：
   - `CATImmDocServices::DeleteOrphanPCDocument(CATIAdpPLMIdentificator*)`
   - `PLMVPMDocServices::DeleteVPMDocument(CATIAdpPLMIdentificator*)`
   且它 import 低层 `PLMVPMDocDeleteServices::DeleteVPMDocument`（即调用方）。
8. 结论方案：Detach → Save/传播 → `GetParents` 判孤儿 → 动态调用删除服务；共享文档只解绑。

## 硬性规则
- 先公开、后内部；无导出符号不得臆造。
- 内部方法集中封装、失败回退、打印 `GetLastError`/`HResultToString`。
- 明确标注“内部未公开 API”，评估版本升级风险。
- 文档有而导出无 → 结论写“文档描述存在，但当前版本未导出”，不得声称“可用”。
