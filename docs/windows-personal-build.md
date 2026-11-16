# Fork 的 Windows 手动构建

本流程生成 Windows 10/11 x64 的 release 安装包，无需作者的更新签名密钥或其他私有 Secrets，也不创建或发布 GitHub Release。安装包与程序未做 Authenticode 代码签名，不生成更新签名。Windows 可能提示发布者未知；本流程不修改系统防护设置。

## 在 GitHub Actions 构建与下载

1. 将本次改动提交并推送到自己的 Fork 的 `codex/callout-title-windows-build` 分支，该分支的推送会自动触发 **Windows Manual Build**，无需合并到 `main`。若以后要通过 **Run workflow** 手动选择分支，工作流还需存在于 Fork 的默认分支，待构建分支也须包含本流程及相关改动。
2. 打开 `https://github.com/li398779124/winer/actions`。如果 Fork 的 Actions 尚未启用，先启用工作流。
3. 在左侧选择 **Windows Manual Build**，打开刚才推送触发的运行。工作流进入默认分支后，也可点击 **Run workflow**，在 **Use workflow from** 选择需要编译的分支，再点击绿色的 **Run workflow**。
4. 打开这一次运行，等待 **Windows x64 unsigned installer** 成功。在运行详情页下方的 **Artifacts** 下载 **winer-windows-x64-unsigned**。下载需登录有访问权限的 GitHub 账号，产物保留 14 天。
5. 解压下载的 ZIP，运行其中的 **`winer_<版本>_x64-setup.exe`** 安装，然后从开始菜单启动 **winer**（安装后的主程序为 `winer.exe`）。ZIP 中的是安装包，不是便携版；无需把源码、Node.js 或 Rust 带到运行机器。

`BUILD-INFO.txt` 记录分支、准确提交和 Actions 运行地址。`SHA256SUMS` 记录安装包、许可证和说明文件的 SHA-256；可用 PowerShell 的 `Get-FileHash <安装包路径> -Algorithm SHA256` 核对下载完整性。

本流程编译的是运行时所选分支的提交，运行详情中的提交和 `BUILD-INFO.txt` 可用于核对。后续分支的新提交不会改变已经启动的构建。

## 安装包包含什么

- 复用项目 Tauri NSIS 安装方式，编译前通过 `beforeBuildCommand` 构建 React 前端和客户端插件。`custom-protocol` 将前端嵌入 release 程序，插件 JS 由 `build.rs` 嵌入；不启用 `qa` 或调试特性。
- 仓库的 `vendor/pengu-loader/core.dll` 已通过 `include_bytes!` 嵌入程序，按现有逻辑写入本机数据目录并加载；无需另外下载 Pengu Loader 的窗口程序或 .NET 依赖。本流程保留现有增强模块，不是“纯战绩模式”。
- 对仓库内 Pengu `core.dll` 的 PE 导入表检查显示：直接依赖 Windows 系统的 `KERNEL32.dll`、`USER32.dll`、`GDI32.dll`，延迟依赖客户端提供的 `libcef.dll`；无需随 winer 再打包一份客户端 CEF，但增强模块仍需要已安装的英雄联盟客户端。
- 安装包内含 WebView2 离线安装程序；缺失 WebView2 时由安装器处理，不要求用户手动搜寻 DLL。工作流通过目标专用的 `RUSTFLAGS` 静态链接主程序的 MSVC 运行库，安装器按 Tauri 的规则收集其他运行文件。离线 WebView2 会显著增大包体。
- ZIP 同时保留 winer 的 `LICENSE`、`THIRD_PARTY_NOTICES.md` 和本说明。安装包不是一个单独拷出的裸 `winer.exe`。

WebView2 的打包选项见 [Tauri Windows 安装文档](https://v2.tauri.app/distribute/windows-installer/#webview2-installation-options)。最终安装、启动、客户端连接及游戏内输入仍需在 Windows 真机验证。

## 更新与配置范围

手动构建额外合并 `app/src-tauri/tauri.personal.conf.json`：关闭 `createUpdaterArtifacts`，并将更新地址设为空。个人构建不会从上游下载更新覆盖定制内容；现有“检查更新”入口会提示未配置更新地址，后台检查失败保持安静。以后需要新版本时，重新运行本流程并手动安装。

个人构建沿用原应用标识与安装位置，**会覆盖同一用户已安装的 winer**，不是与官方版隔离安装。安装前退出正在运行的 winer。普通发布工作流仍使用原配置，不受这份覆盖配置影响。

如需在自己的 Windows 开发机执行同一构建，在仓库根目录安装 `package.json` 指定的 pnpm、Node.js 和 Rust 1.96.0/MSVC 工具链后运行：

```powershell
pnpm install --frozen-lockfile
$env:CARGO_TARGET_X86_64_PC_WINDOWS_MSVC_RUSTFLAGS = "-C target-feature=+crt-static"
pnpm tauri build --target x86_64-pc-windows-msvc --bundles nsis --config app/src-tauri/tauri.personal.conf.json --features custom-protocol --no-sign -- --locked
```

安装包位于 `target/x86_64-pc-windows-msvc/release/bundle/nsis/`。Mac 只需修改和推送源码，最终安装包由 Windows runner 生成。

标题开关、模板示例及双方发送限制见 [评级与喊话设置](callout-settings.md)。
