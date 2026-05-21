# YooAsset 热更新流程分析

## 一、总体架构概览

YooAsset v3 的热更新体系基于 **状态机驱动 + 双文件系统协同 + 分层异步操作** 架构。核心由以下模块组成：

```
用户 API 层 (ResourcePackage)
    └── 文件系统中枢 (FileSystemHost)
          ├── 内置文件系统 (BuiltinFileSystem)    — 读取 StreamingAssets
          └── 沙盒文件系统 (SandboxFileSystem)    — 下载缓存 + 远端请求
                ├── 下载调度器 (DownloadScheduler)
                └── Bundle 缓存 (SandboxBundleCache)
```

### 三种主要运行模式

| 模式 | 文件系统 | 适用场景 |
|------|---------|---------|
| EditorSimulateMode | EditorFileSystem x1 | 编辑器调试，直接使用本地原始资源 |
| OfflinePlayMode | BuiltinFileSystem x1 | 纯离线，资源全部打包在 APP 内 |
| **HostPlayMode** | BuiltinFileSystem + **SandboxFileSystem** x2 | **热更新核心模式**，内置资源 + 远端下载 |

---

## 二、热更新完整流程（8 步状态机）

示例代码参见 `Assets/YooAsset/Samples~/Space Shooter/GameScript/Runtime/PatchLogic/PatchManager.cs`

```
FsmInitializePackage
  → FsmRequestPackageVersion
  → FsmUpdatePackageManifest
  → FsmCreateDownloader
  → FsmDownloadPackageFiles
  → FsmDownloadPackageOver
  → FsmClearCacheBundle
  → FsmStartGame
```

### 第 1 步：初始化资源包裹

**文件**: [FsmInitializePackage.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmInitializePackage.cs)

```csharp
// HostPlayMode 需要传入两套文件系统参数
var options = new HostPlayModeOptions
{
    BuiltinFileSystem = BuiltinFileSystemParameters,  // 内置资源（StreamingAssets）
    CacheFileSystem = CacheFileSystemParameters         // 缓存/远端资源（PersistentDataPath）
};
package.InitializePackageAsync(options);
```

内部根据 HostPlayMode 创建 **两个** `IFileSystem`：
- `BuiltinFileSystem` — 负责 StreamingAssets 中的内置资源
- `SandboxFileSystem` — 负责 persistentDataPath 中的缓存资源 + 远端 CDN 请求

### 第 2 步：请求远端版本号

**文件**: [FsmRequestPackageVersion.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmRequestPackageVersion.cs)

```csharp
var operation = package.RequestPackageVersionAsync();
await operation;
string version = operation.PackageVersion;  // 如 "1.0.3"
```

内部流程（SandboxFileSystem）：
1. 向远端 CDN 请求 `{PackageName}.version` 文件
2. URL 通过 `IRemoteService.GetRemoteUrls(fileName)` + `IDownloadUrlPolicy.SelectUrl(urls)` 获取
3. 响应内容是一个纯文本字符串，直接作为版本号返回

相关源码：
- [RequestPackageVersionOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/RequestPackageVersionOperation.cs)
- [RequestRemotePackageVersionOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/RequestRemotePackageVersionOperation.cs)

### 第 3 步：更新资源清单（关键步骤）

**文件**: [FsmUpdatePackageManifest.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmUpdatePackageManifest.cs)

```csharp
var operation = package.LoadPackageManifestAsync(
    new LoadPackageManifestOptions(packageVersion, timeout: 60));
await operation;
```

内部流程（顶层入口 `LoadPackageManifestOperation`，沙盒文件系统实际执行类 `SFSLoadPackageManifestOperation` 状态机 4 阶段）：

1. **下载 hash 文件** —— `DownloadPackageHashOperation` 下载 `{PackageName}_{PackageVersion}.hash`
   - 触发分支：[SFSLoadPackageManifestOperation.cs:45](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageManifestOperation.cs#L45) `ESteps.DownloadPackageHash`
   - 执行：[DownloadPackageHashOperation.cs:58-94](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/DownloadPackageHashOperation.cs#L58-L94) `InternalUpdate()` 的 `DownloadFile` 分支
   - 文件名由 [YooAssetConfiguration.GetPackageHashFileName()](../Assets/YooAsset/Runtime/Settings/YooAssetConfiguration.cs) 生成；下载到临时文件 → 文本校验 → 原子 Move 到缓存路径

2. **下载清单二进制文件** —— `DownloadPackageManifestOperation` 下载 `{PackageName}_{PackageVersion}.bytes`
   - 触发分支：[SFSLoadPackageManifestOperation.cs:69](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageManifestOperation.cs#L69) `ESteps.DownloadPackageManifest`
   - 执行：[DownloadPackageManifestOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/DownloadPackageManifestOperation.cs)（流程与 hash 下载相同）

3. **读取本地 hash 值** —— `LoadCachePackageHashOperation`（从刚下载的 .hash 文件读出字符串）
   - 触发分支：[SFSLoadPackageManifestOperation.cs:93](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageManifestOperation.cs#L93) `ESteps.LoadPackageHash`
   - 执行：[LoadCachePackageHashOperation.cs:41-62](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/LoadCachePackageHashOperation.cs#L41) `InternalUpdate()`，结果暴露为 `PackageHash` 属性，下一步使用

4. **校验 bytes 完整性 + 反序列化** —— `LoadCachePackageManifestOperation` 内部再分 3 步：`LoadFileData → VerifyFileData → LoadManifest`
   - 触发分支：[SFSLoadPackageManifestOperation.cs:118](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageManifestOperation.cs#L118) `ESteps.LoadPackageManifest`
   - 校验：[LoadCachePackageManifestOperation.cs:72](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/LoadCachePackageManifestOperation.cs#L72) 调用 `PackageManifestHelper.VerifyManifestData()`
     实现见 [PackageManifestHelper.cs:25-40](../Assets/YooAsset/Runtime/ResourcePackage/PackageManifestHelper.cs#L25)：哈希字符串长度 == 32 走 `HashUtility.ComputeMD5`，否则走 `HashUtility.ComputeCrc32`
   - 反序列化：[LoadCachePackageManifestOperation.cs:83-101](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/LoadCachePackageManifestOperation.cs#L83) 创建并驱动 `DeserializeManifestOperation`（依次解析 FileHeader / AssetList / BundleList，初始化 `PackageManifest`）

5. **激活新清单** —— 由顶层 `LoadPackageManifestOperation` 在子操作 `Succeeded` 后调用，后续所有 `LoadAssetAsync` / `CreateResourceDownloader` 都基于此 `ActiveManifest`
   - 调用点：[LoadPackageManifestOperation.cs:90](../Assets/YooAsset/Runtime/ResourcePackage/Operations/LoadPackageManifestOperation.cs#L90) → [FileSystemHost.SetActiveManifest()](../Assets/YooAsset/Runtime/ResourcePackage/FileSystemHost.cs#L103)（仅 `ActiveManifest = manifest`）

### 第 4 步：创建下载器

**文件**: [FsmCreateDownloader.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmCreateDownloader.cs)

```csharp
var downloader = package.CreateResourceDownloader(new ResourceDownloaderOptions(
    downloadingMaxNum,   // 最大并发数，范围 1-32
    failedTryAgain       // 失败重试次数
));

// 检查是否需要下载
if (downloader.TotalDownloadCount == 0)
{
    // 无需下载，直接进入游戏
}
else
{
    // 通知 UI：发现 TotalDownloadCount 个文件需要更新，总大小 TotalDownloadBytes
}
```

内部流程（`FileSystemHost.CreateResourceDownloader()`）：
1. 遍历激活清单中的所有 `PackageBundle`
2. 对每个 bundle 调用 `fileSystem.IsDownloadRequired(bundle)` 进行判定
3. 将需要下载的 bundle 打包成 `ResourceDownloaderOperation`
4. 统计 `TotalDownloadCount`（总文件数）和 `TotalDownloadBytes`（总字节数）

相关源码：
- [FileSystemHost.cs](../Assets/YooAsset/Runtime/ResourcePackage/FileSystemHost.cs) — `CreateResourceDownloader()` 方法
- [DownloaderOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/DownloaderOperation.cs) — `ResourceDownloaderOperation` 类
- [BundleInfo.cs](../Assets/YooAsset/Runtime/ResourcePackage/BundleInfo.cs) — `IsDownloadRequired()` 方法

### 第 5 步：执行下载

**文件**: [FsmDownloadPackageFiles.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmDownloadPackageFiles.cs)

```csharp
downloader.DownloadCompleted += OnDownloadCompleted;
downloader.DownloadProgressChanged += OnDownloadProgress;
downloader.DownloadError += OnDownloadError;
downloader.DownloadFileStarted += OnDownloadFileStarted;

downloader.StartDownload();
await downloader;
```

下载器内部是一个**动态并发管理**的状态机，详见第四章"下载文件机制详解"。

### 第 6~8 步：收尾

- **FsmDownloadPackageOver** — 过渡节点，确认下载完成
- **FsmClearCacheBundle** — 调用 `package.ClearCacheAsync(ClearCacheMethods.ClearUnusedBundleFiles)` 清理旧版本不再需要的缓存文件
- **FsmStartGame** — 设置游戏资源包裹，加载首页场景

---

## 三、如何判定是否需要更新（两级判定）

### 第一级：版本号判定（粗粒度）

**代码位置**: [LoadPackageManifestOperation.cs:55](../Assets/YooAsset/Runtime/ResourcePackage/Operations/LoadPackageManifestOperation.cs#L55)

```csharp
if (_host.ActiveManifest != null
    && _host.ActiveManifest.PackageVersion == _options.PackageVersion)
{
    // 版本号相同，跳过清单加载，直接返回成功
    SetResult();
    return;
}
```

**关键点**：
- 版本号是一个 **字符串**（如 `"1.0.3"`），从远端 `{PackageName}.version` 文件获取
- 通过 **字符串相等** 比较，不做语义版本解析
- 版本号可以是任意格式：日期、语义版本、自定义编号均可
- 这是粗粒度的版本判定，仅决定是否需要重新加载清单

### 第二级：Bundle 级判定（细粒度，文件级差异更新）

**代码位置**: [SandboxFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/SandboxFileSystem.cs) → `BundleCache.IsCached(bundle.BundleGuid)`

```csharp
// SandboxFileSystem.IsDownloadRequired()
return BundleCache.IsCached(bundle.BundleGuid) == false;
```

**核心数据结构 `PackageBundle`**:

| 字段 | 说明 |
|------|------|
| `FileHash` (= BundleGuid) | 文件内容的哈希值，是资源包的唯一标识 |
| `FileCrc` | 文件 CRC 校验码 |
| `FileSize` | 文件大小（字节） |
| `UnityCrc` | Unity 引擎生成的 CRC |
| `BundleName` | 资源包名称 |

**判定逻辑**：
- `BundleGuid` 就是文件的 `FileHash`，以它为 key 查找本地缓存
- **缓存已存在**（`IsCached() = true`）→ 不需要下载此 Bundle
- **缓存不存在**（`IsCached() = false`）→ 需要下载此 Bundle

只要文件内容发生变化，FileHash 就不同 → `BundleGuid` 不同 → 缓存中找不到 → 判定需要下载。

### 判定流程总结

```
客户端请求远端版本号 ({PackageName}.version)
    │
    ├── 版本号与当前激活清单相同 → 无需任何更新，直接进入游戏
    │
    └── 版本号不同 → 下载新版本清单 (.hash + .bytes)
            │
            └── 遍历清单中每个 Bundle，调用 IsDownloadRequired()
                    │
                    ├── BundleCache 中存在该 BundleGuid → 跳过（文件已是最新）
                    │
                    └── BundleCache 中不存在 → 加入下载列表
                            │
                            ├── 下载列表为空 → 直接进入游戏
                            │
                            └── 下载列表非空 → 进入下载流程
```

### 双文件系统协同下的判定

HostPlayMode 维护两个文件系统，加载时的路由顺序：

| 顺序 | 文件系统 | CanAcceptBundle() | IsDownloadRequired() |
|------|---------|-------------------|---------------------|
| 1（优先） | BuiltinFileSystem | 检查 bundle 的 fileHash 是否在 StreamingAssets 的 catalog 中 | 始终 `false` |
| 2（保底） | SandboxFileSystem | 始终 `true` | `!BundleCache.IsCached()` |

- 如果 BuiltinFileSystem 能接管（文件已内置在包体内），直接加载，**不下载**
- SandboxFileSystem 做保底，检查缓存；无缓存则标记为需要下载

---

## 四、下载文件机制详解

### 分层架构

```
ResourceDownloaderOperation        ← 批量管理器：并发、重试、进度、暂停/恢复
    └── SFSDownloadBundleOperation  ← 单文件任务：去重、重试调度、URL 选择
          └── DownloadAndCacheFileOperation  ← 网络传输：HTTP 下载、断点续传
                └── SBCWriteCacheOperation   ← 缓存持久化：写入 + 校验
```

### 4.1 批量管理器 `DownloaderOperation`

**文件**: [DownloaderOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/DownloaderOperation.cs)

关键设计：
- **状态机**: `Check → Downloading → Finish → Done`
- **动态并发控制**: 最大并发 1-32，**每帧最多创建 1 个新下载器**（将初始化开销分摊到多帧，避免移动端单帧尖峰）
- **重试机制**: 可配置 `retryCount`
- **暂停/恢复**: `PauseDownload()` / `ResumeDownload()` — 暂停不创建新任务，已运行的不中断
- **取消**: `CancelDownload()` — 中止所有进行中的下载
- **合并**: `Combine(DownloaderOperation)` — 多个下载器合并为一个

**每帧更新逻辑**（`InternalUpdate()` 中）：
1. 遍历 `_downloaders` 列表，检查完成状态
2. 收集进度（已下载字节、已完成数量），触发 `DownloadProgressChanged` 事件
3. 移除已完成/失败的下载器
4. 如有失败文件 → 停止创建新下载器，报告 `DownloadError` 事件
5. 如 `_downloaders.Count < _maxConcurrency` → 从待下载列表取出 1 个，创建新下载器
6. 待下载列表 + 活跃下载列表都为空 → 下载结束

**进度计算**：
```
Progress = _lastDownloadBytes / TotalDownloadBytes   (按字节计算)
或
Progress = _lastDownloadCount / TotalDownloadCount   (无字节信息时按数量计算)
```

### 4.2 单文件下载 `SFSDownloadBundleOperation`

**文件**: [SFSDownloadBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSDownloadBundleOperation.cs)

- **下载去重**: 通过 `DownloadScheduler.TryGetDownloadOperation(bundle)` 检查是否有相同 bundle 已在下载中，存在则复用（引用计数共享）
- **重试策略**: `DownloadRetryController` 控制，失败后可延迟重试
- **URL 选择**: `IDownloadUrlPolicy` 从 `IRemoteService` 提供的候选 URL 列表中选择

### 4.3 实际网络传输 `DownloadAndCacheFileOperation`

**文件**: [DownloadAndCacheFileOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/DownloadAndCacheFileOperation.cs)

这是最终执行 HTTP 下载的操作，状态机为 `CreateRequest → CheckRequest → CacheFile → Done`。

#### 正常下载流程

1. 创建下载请求 → 目标路径为 **临时文件**
2. 等待请求完成（`WatchdogTimeout` 做看门狗检测，不做超时中断）
3. 成功后执行 `SBCWriteCacheOperation`：
   - 将临时文件移动到正式缓存路径
   - 写入 `__info` 元数据文件
4. 删除临时文件

#### 断点续传流程

启用条件：`Bundle.FileSize >= ResumeDownloadMinimumSize`（默认 `long.MaxValue`，即**默认关闭**，需手动配置）

1. 检查临时文件是否已存在
2. 存在 → 读取已有文件大小作为 `resumeOffset`
3. 创建带 `Range: bytes={resumeOffset}-` HTTP 头的请求
4. `appendToFile = true` → 追加写入
5. 如文件已超过目标大小 → 删除后重新下载
6. 如 HTTP 返回 416 (Range Not Satisfiable) → 服务器文件已变化，删除临时文件重新下载

### 4.4 下载调度器 `DownloadSchedulerOperation`

**文件**: [DownloadSchedulerOperation.cs](../Assets/YooAsset/Runtime/DownloadSystem/Operations/DownloadSchedulerOperation.cs)

每个 SandboxFileSystem 有**一个单例调度器**，全局管理所有活跃下载任务：
- 维护 `bundleGuid → DownloadFileBaseOperation` 映射字典（去重）
- 控制全局并发数和每帧请求数
- 自动清理零引用（无人等待）的下载任务
- 支持 `PauseScheduler()` / `ResumeScheduler()`

### 4.5 下载 URL 构建

文件命名规则由 [YooAssetConfiguration.cs](../Assets/YooAsset/Runtime/Settings/YooAssetConfiguration.cs) 定义：

| 文件类型 | 命名格式 |
|---------|---------|
| 版本号文件 | `{PackageName}.version` |
| 清单哈希文件 | `{PackageName}_{PackageVersion}.hash` |
| 清单二进制文件 | `{PackageName}_{PackageVersion}.bytes` |
| Bundle 文件 | 由构建时的 `EFileNameStyle` 决定 |

URL 构建链：
1. `IRemoteService.GetRemoteUrls(fileName)` → 返回候选 URL 列表（用户实现，如 CDN 多域名）
2. `IDownloadUrlPolicy.SelectUrl(urls)` → 从候选中选择一个（可追加时间戳防缓存）

---

## 五、如何加载下载的文件

### 5.1 加载流程总览

```
ResourcePackage.LoadAssetAsync("location")
    → ResourceManager.LoadAssetAsync(assetInfo)
        → 创建 Provider (AssetProvider / SceneProvider / SubAssetsProvider...)
            → Provider 通过 FileSystemHost 获取 BundleInfo
                → FileSystemHost.GetOwnerFileSystem(packageBundle)
                    → 遍历文件系统列表，CanAcceptBundle() 判定归属
            → Provider 创建 LoadBundleOperation
                → BundleInfo.CreateBundleLoader()
                    → fileSystem.LoadPackageBundleAsync(options)
                        → 等待 Bundle 加载完成
            → Provider 拿到 LoadedBundleHandle
                → LoadedBundleHandle.LoadAssetAsync(assetInfo)
                    → AssetBundle.LoadAssetAsync(path, type)
            → 返回 AssetHandle
```

### 5.2 双文件系统加载优先级

`FileSystemHost.GetOwnerFileSystem()` 按注册顺序遍历：

1. **BuiltinFileSystem** 先检查 — `CanAcceptBundle()` 检查 bundle GUID 是否在 StreamingAssets catalog 中
   - 命中 → 从 `Application.streamingAssetsPath/{PackageName}/{fileName}` 加载（`AssetBundle.LoadFromFile`）
   - **无需网络，不占缓存空间**
2. **SandboxFileSystem** 保底接管 — `CanAcceptBundle()` 始终返回 `true`

### 5.3 "边玩边下" 机制

**文件**: [SFSLoadPackageBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageBundleOperation.cs)

```
BundleCache.IsCached(bundle.BundleGuid)?
    │
    ├── true (已缓存)
    │     → 从缓存路径直接加载
    │        路径: {persistentDataPath}/{PackageName}/
    │              cache_bundle_files/{Hash前2字符}/{BundleGuid}/__data
    │        方式: AssetBundle.LoadFromFile(filePath)
    │
    └── false (未缓存)
          │
          ├── DisableOnDemandDownload = false（默认）
          │     → 自动触发下载（SFSDownloadBundleOperation）
          │     → 下载完成后写入缓存
          │     → 从缓存路径加载
          │     （这就是"边玩边下"——按需即时下载 + 缓存）
          │
          └── DisableOnDemandDownload = true
                → 抛出异常，报告资源不可用
                （适用于先批量下载完所有资源，再进游戏的场景）
```

### 5.4 Bundle 类型与加载方式

| Bundle 类型 | Handle 类 | 加载方式 |
|------------|----------|---------|
| AssetBundle | `AssetBundleHandle` | `AssetBundle.LoadFromFile(path)` 或 `LoadFromFileAsync(path)` |
| RawBundle | `RawBundleHandle` | 直接文件 IO，返回 `RawFileObject`（字节数组） |
| ArchiveBundle | `ArchiveBundleHandle` | 解析归档文件，按需提取子资源 |
| VirtualAssetBundle | `VirtualAssetBundleHandle` | 编辑器模拟，使用 Unity `AssetDatabase` |

### 5.5 完整调用链：从 LoadAssetAsync 到拿到资源对象

```
// 1. 用户调用
AssetHandle handle = package.LoadAssetAsync<GameObject>("HeroPrefab");
await handle;

// 2. 内部链路（简化）
ResourcePackage.LoadAssetAsync()
  → ResourceManager.LoadAssetAsync()
    → new AssetProvider()
      → ProviderBase 构造:
          → FileSystemHost.GetMainBundleInfo(assetInfo)
            → ActiveManifest.GetMainPackageBundle(asset)
            → GetOwnerFileSystem(packageBundle) → 确定文件系统
            → new BundleInfo(fileSystem, packageBundle)
          → ResourceManager.GetOrCreateMainBundleLoader(bundleInfo)
            → new LoadBundleOperation(bundleInfo)
              → bundleInfo.CreateBundleLoader()
                → fileSystem.LoadPackageBundleAsync(options)

      → ProviderBase 等待 BundleLoader 完成
        → 拿到 LoadedBundleHandle (IBundleHandle)

      → AssetProvider.InternalProcessBundleHandle()
        → LoadedBundleHandle.LoadAssetAsync(assetInfo)
          → AssetBundleHandle 的情况下:
            → ABHLoadAssetOperation
              → _assetBundle.LoadAssetAsync(assetPath, type)
        → 从 AssetBundleRequest 获取 UnityEngine.Object

// 3. 用户获取
GameObject prefab = handle.GetAssetObject<GameObject>();
```

---

## 六、关键数据结构

### PackageManifest（资源清单）

| 字段 | 说明 |
|------|------|
| `PackageVersion` | 资源版本号字符串 |
| `PackageName` | 包裹名称 |
| `AssetList` | 所有资源描述列表 (`List<PackageAsset>`) |
| `BundleList` | 所有资源包描述列表 (`List<PackageBundle>`) |
| `AssetDictionary` | 资源路径 → PackageAsset 映射 |
| `BundlesByGuid` | BundleGuid → PackageBundle 映射 |

### PackageBundle（资源包描述）

| 字段 | 说明 |
|------|------|
| `FileHash` (= BundleGuid) | 文件内容哈希值，**版本判定的核心依据** |
| `FileCrc` | 文件 CRC 校验码 |
| `FileSize` | 文件大小（字节） |
| `UnityCrc` | Unity 引擎生成的 CRC |
| `BundleName` | 资源包名称 |
| `IsEncrypted` | 是否加密 |

### 缓存路径结构

```
{Application.persistentDataPath}/
  └── {PackageName}/
        ├── cache_bundle_files/          # Bundle 文件缓存
        │     └── {Hash前2字符}/
        │           └── {BundleGuid}/
        │                 ├── __data     # 实际文件数据
        │                 └── __info     # 元数据（文件大小、校验码、时间戳等）
        ├── cache_manifest_files/        # 清单文件缓存
        │     ├── {PackageName}_{Version}.hash
        │     └── {PackageName}_{Version}.bytes
        └── temp_download_files/         # 下载中的临时文件
```

---

## 七、核心接口

用户实现以下接口来接入热更新：

| 接口 | 作用 | 文件 |
|------|------|------|
| `IRemoteService` | 提供远端文件下载 URL 列表 | [IRemoteService.cs](../Assets/YooAsset/Runtime/Interfaces/IRemoteService.cs) |
| `IDownloadUrlPolicy` | URL 选择策略（主 URL / 备用 URL） | [IDownloadUrlPolicy.cs](../Assets/YooAsset/Runtime/Interfaces/IDownloadURLPolicy.cs) |
| `IDownloadRetryPolicy` | 下载重试策略 | 接口定义在 DownloadSystem 中 |
| `IManifestDecryptor` | 清单解密接口 | [IManifestDecryptor.cs](../Assets/YooAsset/Runtime/Interfaces/IManifestDecryptor.cs) |

---

## 八、流程总图

```
┌──────────────────────────────────────────────────────────────┐
│                      热更新入口                               │
│                                                              │
│  1. package.RequestPackageVersionAsync()                     │
│     └─→ 请求 {PackageName}.version → 获取版本号字符串         │
│                                                              │
│  2. package.LoadPackageManifestAsync(version)                │
│     └─→ 下载 .hash + .bytes → 校验完整性 → 反序列化          │
│     └─→ FileSystemHost.SetActiveManifest(manifest)  ◄ 关键   │
│                                                              │
│  3. package.CreateResourceDownloader(options)                │
│     └─→ 遍历清单 BundleList                                  │
│     └─→ 每个 Bundle 调用 IsDownloadRequired()                │
│     └─→ = !BundleCache.IsCached(bundle.BundleGuid)           │
│     └─→ BundleGuid = FileHash（内容变更 → 哈希不同 → 需下载） │
│                                                              │
│  4. if TotalDownloadCount > 0:                               │
│       downloader.StartDownload()                             │
│       └─→ 每帧最多创建 1 个新任务，最多 maxConcurrency 个并发  │
│       └─→ 每文件: DownloadAndCacheFileOperation              │
│            ├─ HTTP GET（可选 Range: bytes=N- 断点续传）       │
│            ├─→ 写入临时文件                                   │
│            ├─→ SBCWriteCacheOperation（移动到缓存路径）       │
│            └─→ 删除临时文件                                   │
│                                                              │
│  5. package.ClearCacheAsync(ClearUnusedBundleFiles)          │
│     └─→ 清理旧版本不再引用的缓存文件                          │
│                                                              │
│  6. 进入游戏，运行时按需加载                                   │
│     └─→ LoadAssetAsync("path")                               │
│          ├─ BuiltinFileSystem 接管 → 从 StreamingAssets 加载  │
│          └─ SandboxFileSystem 接管                            │
│               ├─ 已缓存 → AssetBundle.LoadFromFile(缓存路径)  │
│               └─ 未缓存 → 自动下载（边玩边下）→ 缓存 → 加载   │
└──────────────────────────────────────────────────────────────┘
```

---

## 九、关键文件索引

### 入口和 API

| 文件 | 类/作用 |
|------|---------|
| [ResourcePackage.cs](../Assets/YooAsset/Runtime/ResourcePackage/ResourcePackage.cs) | 用户 API 入口，所有操作的起点 |
| [FileSystemHost.cs](../Assets/YooAsset/Runtime/ResourcePackage/FileSystemHost.cs) | 文件系统中枢，管理多文件系统、创建下载器 |
| [EPlayMode.cs](../Assets/YooAsset/Runtime/ResourcePackage/EPlayMode.cs) | 运行模式枚举 |

### 更新流程核心操作

| 文件 | 作用 |
|------|------|
| [InitializePackageOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/InitializePackageOperation.cs) | 初始化包裹 |
| [RequestPackageVersionOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/RequestPackageVersionOperation.cs) | 请求版本号 |
| [LoadPackageManifestOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/LoadPackageManifestOperation.cs) | 加载并激活清单 |
| [DownloaderOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/DownloaderOperation.cs) | 批量下载管理器 |

### 文件系统实现

| 文件 | 作用 |
|------|------|
| [SandboxFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/SandboxFileSystem.cs) | 沙盒文件系统（HostPlayMode 核心） |
| [BuiltinFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Services/BuiltinFileSystem/BuiltinFileSystem.cs) | 内置文件系统 |
| [IFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Interfaces/IFileSystem.cs) | 文件系统统一接口 |

### 下载和缓存

| 文件 | 作用 |
|------|------|
| [DownloadAndCacheFileOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/Internal/DownloadAndCacheFileOperation.cs) | HTTP 下载 + 缓存写入 |
| [SFSDownloadBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSDownloadBundleOperation.cs) | 沙盒单文件下载任务 |
| [DownloadSchedulerOperation.cs](../Assets/YooAsset/Runtime/DownloadSystem/Operations/DownloadSchedulerOperation.cs) | 全局下载调度器 |
| [SandboxBundleCache.cs](../Assets/YooAsset/Runtime/BundleCache/Services/SandboxBundleCache/SandboxBundleCache.cs) | 沙盒 Bundle 缓存 |

### 资源加载

| 文件 | 作用 |
|------|------|
| [ResourceManager.cs](../Assets/YooAsset/Runtime/ResourceManager/ResourceManager.cs) | 资源加载调度中枢 |
| [ProviderBase.cs](../Assets/YooAsset/Runtime/ResourceManager/Providers/ProviderBase.cs) | Provider 基类，桥接 Bundle 和资源 |
| [LoadBundleOperation.cs](../Assets/YooAsset/Runtime/ResourceManager/Operations/Internal/LoadBundleOperation.cs) | Bundle 加载操作 |
| [SFSLoadPackageBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/SandboxFileSystem/Operations/SFSLoadPackageBundleOperation.cs) | 沙盒 Bundle 加载（含边玩边下） |

### 清单和数据结构

| 文件 | 作用 |
|------|------|
| [PackageManifest.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageManifest.cs) | 资源清单 |
| [PackageBundle.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageBundle.cs) | 资源包描述 |
| [PackageAsset.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageAsset.cs) | 资源描述 |
| [BundleInfo.cs](../Assets/YooAsset/Runtime/ResourcePackage/BundleInfo.cs) | Bundle 运行时包装，含 `IsDownloadRequired()` |
| [AssetInfo.cs](../Assets/YooAsset/Runtime/ResourcePackage/AssetInfo.cs) | 资源信息，对用户的公开类型 |

### 示例代码

| 文件 | 作用 |
|------|------|
| [Boot.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/Boot.cs) | 示例入口 |
| [PatchManager.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/PatchManager.cs) | 状态机编排器 |
| [FsmInitializePackage.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmInitializePackage.cs) | 步骤1：初始化 |
| [FsmRequestPackageVersion.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmRequestPackageVersion.cs) | 步骤2：获取版本号 |
| [FsmUpdatePackageManifest.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmUpdatePackageManifest.cs) | 步骤3：更新清单 |
| [FsmCreateDownloader.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmCreateDownloader.cs) | 步骤4：创建下载器 |
| [FsmDownloadPackageFiles.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/FsmNode/FsmDownloadPackageFiles.cs) | 步骤5：执行下载 |
| [PatchWindow.cs](../Assets/YooAsset/Samples~/Space%20Shooter/GameScript/Runtime/PatchLogic/PatchWindow.cs) | 更新 UI 和事件处理 |
