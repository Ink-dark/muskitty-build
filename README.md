# muskitty-build

MusKitty 的构建/发布流水线仓库（本仓库不存源码）。

## 工作流一览

| 工作流 | 触发 | 作用 |
|--------|------|------|
| **Release Windows Installer** | 手动 | 构建 + Inno Setup 打包 + 创建 Release（含版本号自增） |
| **Build Artifacts** | 手动 | 纯构建：只产出并上传 exe 构建产物，不打包、不发布 |

### 兜底策略

- Release 流程中，**原始构建产物在 Inno 打包之前就先上传**为 artifact——Inno 或后续
  环节失败也能取回已构建的 exe。
- Inno/发布环节彻底出问题时，用 **Build Artifacts** 流程对任意 ref（默认 `main`）
  做一次干净构建并下载产物。

## Release Windows Installer（`.github/workflows/release-windows.yml`）

手动触发（Actions → *Release Windows Installer* → Run workflow），流程：

1. 检出 [Ink-dark/MusKitty](https://github.com/Ink-dark/MusKitty) 主仓库到 `src/MusKitty`，
   并用其 `fetch-crates.ps1` 克隆 11 个独立 crate 仓库到 `crates/`
2. 以上一个 Release tag 为基准按所选位（patch/minor/major）自增版本号
   （无历史 Release 时从 `v0.1.0` 开始）
3. `cargo build --release --workspace`（入口 `muskitty-chrome.exe`）
4. 下载 Inno 非官方简体中文语言包（`Languages\Unofficial\ChineseSimplified.isl`，
   失败时降级为仅英文向导，不阻断发布）
5. Inno Setup 打包为 `MusKitty-Setup-vX.Y.Z.exe`（双语向导 + 桌面快捷方式任务 +
   许可协议页），生成 SHA256 校验文件
6. 产物上传为 workflow artifact；非 dry-run 时创建 Release 并上传成品

### 输入

| 输入 | 说明 |
|------|------|
| `bump` | 版本自增位：`patch`（默认）/ `minor` / `major` |
| `prerelease` | 标记预发布（不抢占 latest） |
| `notes` | 附加发布说明；留空自动生成 changelog |
| `dry_run` | 只构建打包不建 Release，产物走 artifacts 下载 |

### 注意

- Inno Setup `AppId`（`{{8F3C1A55-…}`，`{{` 是 Inno 的字面 `{` 转义）**首发后勿改**，
  否则 Windows 视为另一个软件。
- 并发保护：`release-windows` 组同时只跑一个发布流程，避免抢同一个版本号。
