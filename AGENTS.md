# mds_favor 仓库约定（AGENTS.md）

本仓库是 CATIA CAA（`MDSToolkit`）与 Qt 前端（`MDS_UI_MAIN`）的双进程工具集。

## 编码约定

- CAA 侧源码（`MDSToolkit/**`，含中文注释）：**GBK** 保存与读取，切勿按 UTF-8 误判为乱码。
- Qt 侧源码/界面（`MDS_UI_MAIN/**`，`*.ui`）：**UTF-8**。
- 编辑任何文件时保持其原有编码不变，除非明确要求更改。
- 仓库内 GBK 文件用 ripgrep 可能报 `Invalid ripgrep match output`，改用 PowerShell 按 936 码页读取。

## 命名与代码规范

- 见个人全局 `~/.config/opencode/AGENTS.md`（CAA 函数大驼峰、Qt 小驼峰、成员 `m_`、常量全大写等）。
- CAA 接口类型优先 `_var`，不可用智能指针时用 `DECLARE_COM_GUARD()`（`HWMDSPublicFrm/PublicInterfaces/ComRAIIUtils.h`）。
- `HRESULT hr` 失败分支必须打日志再 continue/return。

## 自检（XKLAutoCheck）框架速览

- 架构：CAA 一次性命令（`CATStateCommand`）+ 隐藏 Win32 常驻窗口 + Qt 双进程，跨进程用「窗口消息 + 共享内存」通信。
- 详细复用说明见 `MDSToolkit/docs/XKLAutoCheck_Framework.md`（含 Qt 侧对接，第 16 节）。

### 关键常量（两端必须一致）

| 用途 | 值 | 定义处 |
|---|---|---|
| CATIA→Qt 目标窗口标题 | `MDS_Qt_Message` | Qt 主窗口 |
| CATIA 隐藏命令窗口 | 类 `AutoCheck_CatiaCmd` / 标题 `AutoCheck_Catia` | `XKLAutoCheckCmd.cpp:114/167` |
| 进程通信窗口 | 类/标题 `MDS_Catia_Message_Cmd` | `MDS_Catia_Message.cpp:427` |
| 进度条窗口 | `MDS_Progress_Message` | `MDS_Catia_Message.cpp` |
| 命名共享内存 | `Catia_MDS_Share` | `SharedMemory.h` |

### 通信约定

- CATIA→Qt：`MDS_Catia_Message::sendMsg("MDS_Qt_Message", json)`。
- Qt→CATIA：`MDS_Qt_Message::userSendMsg("AutoCheck_Catia", json)`，json 带 `Function="AutoCheck"` + `Cmd`。
- 大数据一律写共享内存 `Catia_MDS_Share`，`COPYDATASTRUCT.lpData` 只带短标记。

### 命令注册入口

- `HWMDSAddin::CreateCommands()` 用 `(void*)1/2` 区分模式（1=执行自检，2=复检选项页）。
- 检查项统一收敛到 `AutoCheckSPQManage::Usermain` → `CatiaAutoCheckSPQCmd::SPQ_XXX`。

## 文档索引

- `MDSToolkit/docs/XKLAutoCheck_Framework.md`：自检框架全量复用说明（15 节框架 + 第 16 节 Qt 侧对接）。
