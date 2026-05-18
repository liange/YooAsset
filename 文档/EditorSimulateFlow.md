# YooAsset 编辑器模拟模式资源加载流程分析

## 一、总体架构

编辑器模拟模式（EditorSimulateMode）的核心设计理念是：**在 Unity 编辑器下运行时，绕过 AssetBundle 构建，直接使用 `AssetDatabase` API 从原始资源文件加载**，同时通过虚拟 Bundle 体系模拟真实运行时的行为。

```
┌─────────────────────────────────────────────────────────────┐
│                      构建时（Editor 程序集）                  │
│                                                             │
│  EditorSimulateBuildInvoker.Build()                         │
│    └→ BundleSimulateBuilder.SimulateBuild()                 │
│         └→ EditorSimulateBuildPipeline.Run()                │
│              输出: .bytes / .hash / .version → 清单输出目录   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                      运行时（Runtime 程序集）                 │
│                                                             │
│  EditorSimulateModeOptions                                  │
│    └→ EditorFileSystem (IFileSystem)                        │
│         ├→ EditorBundleCache (内存缓存，仅记录 GUID)          │
│         ├→ DownloadScheduler (可选，模拟下载用)               │
│         └→ EFSLoadPackageBundleOperation                    │
│              └→ EBCLoadVirtual*Operation                    │
│                   └→ Virtual*BundleHandle                   │
│                        └→ AssetDatabase.LoadAssetAtPath()   │
└─────────────────────────────────────────────────────────────┘
```

### 三种模式对比

| 模式 | 文件系统 | Bundle 类型 | 资源加载 API |
|------|---------|------------|-------------|
| EditorSimulateMode | EditorFileSystem x1 | VirtualAssetBundle / VirtualRawBundle / VirtualArchiveBundle | `AssetDatabase.LoadAssetAtPath` |
| OfflinePlayMode | BuiltinFileSystem x1 | AssetBundle / RawBundle / ArchiveBundle | `AssetBundle.LoadFromFile` |
| HostPlayMode | BuiltinFileSystem + SandboxFileSystem x2 | AssetBundle / RawBundle / ArchiveBundle | `AssetBundle.LoadFromFile` |

---

## 二、构建管线：模拟清单的生成

### 2.1 构建入口

用户代码调用 `EditorSimulateBuildInvoker.Build()` 触发构建：

```csharp
// 文件: Runtime/PackageBuilder/EditorSimulateBuildInvoker.cs
var buildResult = EditorSimulateBuildInvoker.Build(packageName, (int)EBundleType.VirtualAssetBundle);
var packageRoot = buildResult.PackageRootDirectory;
```

`EditorSimulateBuildInvoker` 通过**反射**调用 Editor 程序集中的 `BundleSimulateBuilder.SimulateBuild()`，这使得 Runtime 代码可以不直接依赖 Editor 程序集。

### 2.2 构建管道（4 个任务）

**文件**: [EditorSimulateBuildPipeline.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/EditorSimulateBuildPipeline.cs)

```
TaskPrepare_ESBP → TaskGetBuildMap_ESBP → TaskUpdateBundleInfo_ESBP → TaskCreateManifest_ESBP
```

#### 任务 1：参数验证

**文件**: [TaskPrepare_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskPrepare_ESBP.cs)

检查 `EditorSimulateBuildParameters.BuildBundleType` 必须是 `VirtualAssetBundle`、`VirtualRawBundle` 或 `VirtualArchiveBundle` 之一。

#### 任务 2：生成构建映射

**文件**: [TaskGetBuildMap_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskGetBuildMap_ESBP.cs)

- 调用 `TaskGetBuildMap.CreateBuildMap(simulateBuild: true, ...)`
- 以 `simulateBuild = true` 模式启动收集器（`BundleCollector`），收集器以不同逻辑处理三种虚拟 Bundle：
  - **VirtualRawBundle**：每个包只能包含一个原生文件（`CheckRawBundleMapContent`）
  - **VirtualArchiveBundle**：检查子文件数量不超过上限（`CheckArchiveBundleMapContent`）
  - **VirtualAssetBundle**：无特殊限制
- 收集到的资源路径是 **Unity 工程中的原始路径**（如 `Assets/MyGame/Textures/icon.png`），这些路径最终写入清单

#### 任务 3：更新 Bundle 信息

**文件**: [TaskUpdateBundleInfo_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskUpdateBundleInfo_ESBP.cs)

核心方法：

| 属性 | 模拟模式的值 | 说明 |
|------|------------|------|
| `UnityHash` | `"00000000000000000000000000000000"` | 固定值，非真实 AB |
| `UnityCRC` | `0` | 固定值 |
| `FileHash` | 基于源文件路径+修改时间+大小的 MD5 | 用于检测文件变化 |
| `FileCRC` | `0` | 不做 CRC 校验 |
| `FileSize` | 累加所有源文件大小 | 反映实际数据量 |
| `BundleDepends` | `Array.Empty<string>()` | 虚拟 Bundle 没有依赖关系 |

#### 任务 4：生成清单

**文件**: [TaskCreateManifest_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskCreateManifest_ESBP.cs)

- 调用 `TaskCreateManifest.CreateManifestFile(processBundleDepends: false, processBundleTags: true, ...)`
- 生成 `PackageManifest` 对象并序列化为 `.bytes` 文件
- 同时生成 `.hash` 和 `.version` 文件
- 清单中 `PackageVersion` 固定为 `"Simulate"`
- `AssetPathsByLocation` 字典：定位地址 → 原始资源路径（如 `Assets/...`）

### 2.3 构建输出目录结构

```
{packageRoot}/{PackageName}/Simulate/
    ├── {PackageName}_Simulate.version
    ├── {PackageName}_Simulate.hash
    └── {PackageName}_Simulate.bytes
```

清单在运行时从该目录加载。

---

## 三、初始化流程

### 3.1 用户入口

```csharp
// 1. 创建参数
var createParameters = new EditorSimulateModeOptions();
createParameters.EditorFileSystemParameters = FileSystemParameters
    .CreateDefaultEditorFileSystemParameters(packageRoot);

// 可选：配置异步模拟和下载模拟
createParameters.EditorFileSystemParameters.AddParameter(
    EFileSystemParameter.AsyncSimulateMinFrame, 5);
createParameters.EditorFileSystemParameters.AddParameter(
    EFileSystemParameter.AsyncSimulateMaxFrame, 10);
createParameters.EditorFileSystemParameters.AddParameter(
    EFileSystemParameter.VirtualDownloadMode, true);
createParameters.EditorFileSystemParameters.AddParameter(
    EFileSystemParameter.VirtualDownloadSpeed, 1024 * 1000);

// 2. 初始化
var operation = package.InitializePackageAsync(createParameters);
```

### 3.2 初始化操作内部流程

**文件**: [InitializePackageOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/InitializePackageOperation.cs)

```
SetPlayMode:   识别 options 为 EditorSimulateModeOptions
                 → _playMode = EPlayMode.EditorSimulateMode
                 → 只使用 EditorFileSystemParameters（单个文件系统）

CreateCore:     new ResourceManager + new FileSystemHost

InitFileSystem: FileSystemHost.InitializeAsync(editorFileSystemParameters)
                  → InitializeFileSystemOperation:
                      1. 反射创建 EditorFileSystem 实例
                      2. 调用 SetParameter() 设置所有参数
                      3. 调用 fileSystem.OnCreate()
```

### 3.3 EditorFileSystem.OnCreate()

**文件**: [EditorFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/EditorFileSystem.cs)

```csharp
public void OnCreate(string packageName, string packageRoot)
{
    // 1. 创建 EditorBundleCache（传入异步模拟帧数、虚拟下载模式等配置）
    var cacheConfig = new EditorBundleCache.Configuration(
        asyncSimulateMinFrame: AsyncSimulateMinFrame,
        asyncSimulateMaxFrame: AsyncSimulateMaxFrame,
        virtualDownloadMode: VirtualDownloadMode,
        ...);
    _bundleCache = new EditorBundleCache(cacheConfig);

    // 2. 创建下载后端（默认 UnityWebRequestBackend，用于虚拟下载模式）
    _downloadBackend = new UnityWebRequestBackend();

    // 3. 创建下载调度器（用于虚拟下载模式）
}
```

### 3.4 EFSInitializeOperation

**文件**: [EFSInitializeOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSInitializeOperation.cs)

```
CheckPlatform:      确保在 UNITY_EDITOR 下运行
InitializeBundleCache: 初始化 EditorBundleCache（立即完成，无需扫描文件）
CreateScheduler:    创建 DownloadSchedulerOperation（用于虚拟下载）
Done
```

---

## 四、资源定位：如何找到原始资源路径

### 4.1 路径解析链

```
用户传入定位地址 "HeroPrefab"
    ↓
ResourcePackage.ConvertLocationToAssetInfo("HeroPrefab")
    ↓
PackageManifest.TryMappingToAssetPath("HeroPrefab")
    → 查找 AssetPathsByLocation["HeroPrefab"] → "Assets/Game/Prefabs/Hero.prefab"
    ↓
AssetInfo.AssetPath = "Assets/Game/Prefabs/Hero.prefab"
```

清单中的 `AssetPathsByLocation` 字典是在模拟构建时建立的，value 是原始 Unity 资源路径（`Assets/...` 格式）。

### 4.2 EditorFileSystemHelper.GetEditorFilePath()

**文件**: [EditorFileSystemHelper.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/EditorFileSystemHelper.cs)

```csharp
public static string GetEditorFilePath(PackageBundle bundle)
{
    if (bundle.MainAssets.Count == 0)
        return string.Empty;
    var packageAsset = bundle.MainAssets[0];
    return packageAsset.AssetPath;  // 如 "Assets/Game/Prefabs/Hero.prefab"
}
```

这是整个编辑器加载路径的**核心方法**。它直接从清单的 `PackageBundle.MainAssets[0].AssetPath` 获取原始资源路径，后续所有加载操作都基于这个路径。

在真机模式下，`MainAssets` 中的 `AssetPath` 是相对于 Bundle 的资源内部路径（如 `assets/xxx/hero`）；但在编辑器模拟模式下，它直接存储 Unity 项目中的绝对资产路径。

---

## 五、资源加载完整流程

### 5.1 调用链总览

```
package.LoadAssetAsync<GameObject>("HeroPrefab")
  └→ ResourceManager.LoadAssetAsync(assetInfo)
       └→ new AssetProvider(packageName, assetInfo, priority)
            │
            ├ 构造函数（ProviderBase）:
            │   ├→ FileSystemHost.GetMainBundleInfo(assetInfo)
            │   │    └→ ActiveManifest.GetMainPackageBundle(asset)
            │   │         └→ FileSystemHost.GetOwnerFileSystem(packageBundle)
            │   │              └→ EditorFileSystem.CanAcceptBundle() → true
            │   │                   → 返回 BundleInfo(EditorFileSystem, packageBundle)
            │   │
            │   ├→ FileSystemHost.GetDependentBundleInfos(assetInfo)
            │   │    └→ 虚拟 Bundle 无依赖，返回空列表
            │   │
            │   └→ ResourceManager.GetOrCreateMainBundleLoader(bundleInfo)
            │        └→ new LoadBundleOperation(bundleInfo)
            │             └→ bundleInfo.CreateBundleLoader()
            │                  └→ EditorFileSystem.LoadPackageBundleAsync()
            │                       └→ EFSLoadPackageBundleOperation
            │
            └ InternalUpdate():
                 等待 LoadBundleOperation 完成
                   └→ LoadedBundleHandle = VirtualAssetBundleHandle
                        └→ InternalProcessBundleHandle()
                             └→ LoadedBundleHandle.LoadAssetAsync(assetInfo)
                                  └→ VABHLoadAssetOperation
                                       └→ AssetDatabase.LoadAssetAtPath(assetPath)
```

### 5.2 EFSLoadPackageBundleOperation：Bundle 加载过程

**文件**: [EFSLoadPackageBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSLoadPackageBundleOperation.cs)

```
状态机: Prepare → DownloadFile / LoadBundle → CheckResult → Done

Prepare:
  ├→ BundleCache.IsCached(bundle.BundleGuid)?
  │    ├─ true  → 跳过下载，直接进入 LoadBundle
  │    └─ false → 进入 DownloadFile
  │
DownloadFile: (仅在 VirtualDownloadMode = true 时触发)
  └→ EFSDownloadBundleOperation
       └→ SimulateAndCacheFileOperation
            └→ 创建 SimulatedDownloadRequest，按 VirtualDownloadSpeed 模拟下载
            └→ EBCWriteCacheOperation: 仅在内存字典中记录 BundleGuid
            └→ 进入 LoadBundle
  │
LoadBundle:
  └→ EditorBundleCache.LoadBundleAsync(options)
       └→ 根据 BundleType 分发:
            ├ VirtualAssetBundle (12)  → EBCLoadVirtualAssetBundleOperation
            ├ VirtualRawBundle (13)    → EBCLoadVirtualRawBundleOperation
            └ VirtualArchiveBundle (14) → EBCLoadVirtualArchiveBundleOperation
```

### 5.3 异步模拟延迟机制

**文件**: [EBCLoadBundleBaseOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCLoadBundleBaseOperation.cs)

```csharp
// 构造函数中
_asyncSimulateFrame = Random.Range(
    _fileCache.Config.AsyncSimulateMinFrame,   // 默认 1
    _fileCache.Config.AsyncSimulateMaxFrame + 1 // 默认 1
);

// InternalUpdate() 中
if (!IsWaitForCompletion)  // 非同步等待模式
{
    _asyncSimulateFrame--;
    if (_asyncSimulateFrame <= 0)
    {
        CreateBundleHandle();  // 延迟结束，创建 Handle
    }
}
else  // 同步等待模式
{
    CreateBundleHandle();  // 不延迟，直接创建
}
```

默认值为 1→1 帧（即 1 帧延迟），可配置为更大值（如 5→10 帧）来模拟网络加载延迟。

---

## 六、三种虚拟 BundleHandle 详解

### 6.1 VirtualAssetBundleHandle（最常用）

**文件**: [VirtualAssetBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/VirtualAssetBundleHandle.cs)

对应 `EBundleType.VirtualAssetBundle (12)`，模拟真实 AssetBundle。

```
创建:
  EBCLoadVirtualAssetBundleOperation.CreateBundleHandle()
    → new VirtualAssetBundleHandle(packageBundle)
    （不加载任何数据，只是一个"空壳"）

LoadAssetAsync:
  → VABHLoadAssetOperation
      #if UNITY_EDITOR
        if (assetType == null)
          Result = AssetDatabase.LoadMainAssetAtPath(assetPath)
        else
          Result = AssetDatabase.LoadAssetAtPath(assetPath, assetType)
      #endif

LoadAllAssetsAsync:
  → VABHLoadAllAssetsOperation
      遍历 bundle.MainAssets，对每个调用 AssetDatabase.LoadMainAssetAtPath

LoadSubAssetsAsync:
  → VABHLoadSubAssetsOperation
      AssetDatabase.LoadAllAssetRepresentationsAtPath(assetPath)

LoadSceneAsync:
  → VABHLoadSceneOperation
      EditorSceneManager.LoadSceneInPlayMode(assetPath)
      或 LoadSceneAsyncInPlayMode(assetPath)

UnloadBundle:
  → 空操作（无需卸载，AssetDatabase 加载的资源由 Unity 编辑器管理生命周期）
```

关键点：
- **不持有任何真实 Bundle 对象**，只有 `PackageBundle` 元数据
- 所有加载操作都包裹在 `#if UNITY_EDITOR` 中
- 非编辑器平台调用会直接返回错误

### 6.2 VirtualRawBundleHandle

**文件**: [VirtualRawBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualRawBundleHandle/VirtualRawBundleHandle.cs)

对应 `EBundleType.VirtualRawBundle (13)`，模拟原生文件（如 Lua 脚本、JSON 配置等）。

```
创建:
  EBCLoadVirtualRawBundleOperation.CreateBundleHandle()
    → EditorFileSystemHelper.GetEditorFilePath(bundle)  // 获取原始路径
    → File.ReadAllBytes(editorFilePath)                   // 读取字节
    → new RawBundle(bytes)                                // 包装为 RawBundle
    → new VirtualRawBundleHandle(bundle, rawBundle)

LoadAssetAsync:
  → RBHLoadAssetOperation
      RawBundle.CreateRawFileObject()
        → RawFileObject.CreateFromBytes(data)

UnloadBundle:
  → _rawBundle.Unload()  // 释放内存中的字节数据
```

与真机模式的关键区别：真机模式下 `RawBundle` 的数据从真实文件系统读取（如 StreamingAssets 或沙盒），编辑器下则直接从 Assets 目录读取。

### 6.3 VirtualArchiveBundleHandle

**文件**: [VirtualArchiveBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualArchiveBundleHandle/VirtualArchiveBundleHandle.cs)

对应 `EBundleType.VirtualArchiveBundle (14)`，模拟归档文件包（将多个文件打包为一个归档）。

```
创建:
  EBCLoadVirtualArchiveBundleOperation.CreateBundleHandle()
    → 收集 bundle.MainAssets 中所有资源路径
    → new VirtualArchiveBundle(archiveAssetPaths)
    → new VirtualArchiveBundleHandle(bundle, virtualArchiveBundle)

LoadAssetAsync:
  → VARBHLoadAssetOperation
      VirtualArchiveBundle.CreateRawFileObject(assetPath)
        → 验证 assetPath 是否属于此归档
        → File.ReadAllBytes(assetPath)  // 从 Assets 目录读取
        → RawFileObject.CreateFromBytes(fileData)
        → 缓存结果

UnloadBundle:
  → _virtualArchiveBundle.Unload()  // 清理所有缓存的 RawFileObject
```

### 6.4 三种 Handle 能力矩阵

| 操作 | VirtualAssetBundle | VirtualRawBundle | VirtualArchiveBundle |
|------|:---:|:---:|:---:|
| LoadAssetAsync | AssetDatabase.LoadAssetAtPath | RawFileObject.FromBytes | VirtualArchiveBundle.CreateRawFileObject |
| LoadAllAssetsAsync | 遍历调用 AssetDatabase | 空操作(NotSupport) | 空操作(NotSupport) |
| LoadSubAssetsAsync | AssetDatabase.LoadAllAssetRepresentationsAtPath | 空操作(NotSupport) | 空操作(NotSupport) |
| LoadSceneAsync | EditorSceneManager.LoadSceneInPlayMode | 空操作(NotSupport) | 空操作(NotSupport) |
| UnloadBundle | 空操作 | 释放内存数据 | 清理 RawFileObject 缓存 |
| 数据来源 | AssetDatabase API | File.ReadAllBytes(资产路径) | File.ReadAllBytes(资产路径) |

---

## 七、缓存、下载与清理（编辑器特殊处理）

### 7.1 EditorBundleCache

**文件**: [EditorBundleCache.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/EditorBundleCache.cs)

```
特性:
  - IsReadOnly = true (只读缓存，不产生实际文件)
  - 内部只维护 Dictionary<string, EditorBundleCacheEntry>
  - 每个 Entry 只记录 BundleGuid，不存储实际数据

IsCached(bundleGuid):
  → _cacheEntries.ContainsKey(bundleGuid)

WriteCacheAsync():
  → EBCWriteCacheOperation
      向 _cacheEntries 字典添加一个条目（仅记录 GUID）

  （注意：不写入任何文件到磁盘）

ClearCacheAsync():
  → EBCClearCacheOperation
      从 _cacheEntries 字典中移除条目
  （无需删除文件）

VerifyCacheAsync():
  → BCVerifyCacheCompleteOperation (直接返回成功，不做实际校验)
```

### 7.2 虚拟下载模式

当 `VirtualDownloadMode = true` 时，如果 `BundleCache.IsCached() = false`，会触发模拟下载：

```
EFSDownloadBundleOperation
  └→ SimulateAndCacheFileOperation
       ├ CreateRequest:
       │   创建 SimulatedDownloadRequestArgs(url, fileSize, downloadSpeed)
       │   使用 DownloadBackend.CreateSimulateRequest(args)
       │   （不发起真实网络请求，按 downloadSpeed 速度模拟数据到达）
       │
       ├ CheckRequest:
       │   轮询模拟进度，更新 LatestReport
       │
       └ CacheFile:
            EBCWriteCacheOperation（仅内存记录 GUID）
```

### 7.3 与真机模式的关键差异对比

| 方面 | 编辑器模拟模式 | 真机模式 |
|------|--------------|---------|
| **缓存位置** | 内存字典 | persistentDataPath 文件系统 |
| **缓存内容** | 仅 BundleGuid 字符串 | 真实文件数据 + __info 元数据 |
| **缓存写入** | 字典添加条目 | 文件复制 + 元数据写入 |
| **缓存校验** | 直接成功 | CRC/大小校验 |
| **缓存清理** | 字典移除 | 删除文件 |
| **下载实现** | SimulatedDownloadRequest (模拟速度) | UnityWebRequest (真实 HTTP) |
| **断点续传** | 不支持 | 支持（需配置 ResumeDownloadMinimumSize） |

---

## 八、流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                    编辑器模拟构建（一次性）                         │
│                                                                  │
│  EditorSimulateBuildInvoker.Build()                              │
│    └→ BundleSimulateBuilder.SimulateBuild()                      │
│         └→ EditorSimulateBuildPipeline.Run()                     │
│              ├ TaskPrepare_ESBP: 验证 BundleType 为虚拟类型        │
│              ├ TaskGetBuildMap_ESBP: simulateBuild=true 收集资源   │
│              │    AssetPath = "Assets/Game/Prefabs/Hero.prefab"  │
│              ├ TaskUpdateBundleInfo_ESBP:                         │
│              │    FileHash = MD5(路径+时间+大小), UnityCRC = 0    │
│              └ TaskCreateManifest_ESBP:                           │
│                   序列化 PackageManifest → .bytes / .hash / .version│
│                                                                  │
│  输出 → {packageRoot}/{PackageName}/Simulate/                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    运行时初始化                                    │
│                                                                  │
│  new EditorSimulateModeOptions                                   │
│    { EditorFileSystemParameters(packageRoot) }                   │
│                                                                  │
│  package.InitializePackageAsync(options)                         │
│    └→ InitializePackageOperation                                 │
│         ├ SetPlayMode: 识别为 EditorSimulateMode                  │
│         ├ CreateCore: new ResourceManager + FileSystemHost       │
│         └ InitFileSystem:                                        │
│              反射创建 EditorFileSystem                             │
│              ├ OnCreate: EditorBundleCache + DownloadScheduler   │
│              └ EFSInitializeOperation                            │
│                   ├ CheckPlatform (UNITY_EDITOR)                  │
│                   ├ InitializeBundleCache (立即成功)              │
│                   └ CreateScheduler (虚拟下载用)                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    运行时资源加载                                  │
│                                                                  │
│  package.LoadAssetAsync<GameObject>("HeroPrefab")                │
│    │                                                             │
│    ├ 1. 路径解析                                                 │
│    │   PackageManifest.TryMappingToAssetPath("HeroPrefab")       │
│    │     → "Assets/Game/Prefabs/Hero.prefab"                    │
│    │                                                             │
│    ├ 2. 获取 BundleInfo                                          │
│    │   FileSystemHost.GetOwnerFileSystem(packageBundle)          │
│    │     → EditorFileSystem.CanAcceptBundle() → true             │
│    │     → BundleInfo(EditorFileSystem, packageBundle)           │
│    │   (虚拟 Bundle 无依赖，依赖列表为空)                          │
│    │                                                             │
│    ├ 3. 加载 Bundle（获取 BundleHandle）                          │
│    │   EditorFileSystem.LoadPackageBundleAsync()                 │
│    │     └→ EFSLoadPackageBundleOperation                        │
│    │          ├ Prepare: BundleCache.IsCached()?                 │
│    │          ├ DownloadFile: (可选) 模拟下载                     │
│    │          └ LoadBundle:                                      │
│    │               EditorBundleCache.LoadBundleAsync()           │
│    │                 └→ EBCLoadVirtualAssetBundleOperation       │
│    │                      ├ 等待 asyncSimulateFrame 帧            │
│    │                      └ new VirtualAssetBundleHandle()       │
│    │                                                             │
│    └ 4. 从 BundleHandle 加载具体资源                              │
│        AssetProvider.InternalProcessBundleHandle()               │
│          └→ VirtualAssetBundleHandle.LoadAssetAsync(assetInfo)   │
│               └→ VABHLoadAssetOperation                          │
│                    └→ AssetDatabase.LoadAssetAtPath(             │
│                         "Assets/Game/Prefabs/Hero.prefab",       │
│                         typeof(GameObject))                      │
│                         → 返回 GameObject 实例                    │
│                                                                  │
│  返回 AssetHandle → GetAssetObject<GameObject>()                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 九、Provider 层面的无差异设计

**重要设计特点**：Provider（AssetProvider、SceneProvider 等）在编辑器和真机模式下**代码完全一致**，没有任何 `#if UNITY_EDITOR` 分支。

模式差异通过**多态**实现：

1. `FileSystemHost.GetOwnerFileSystem()` 根据 `CanAcceptBundle()` 选择不同的 `IFileSystem`
2. 不同的 `IFileSystem.LoadPackageBundleAsync()` 返回不同的 `BundleHandle`
3. Provider 拿到 `IBundleHandle` 后调用 `LoadAssetAsync()`，实际行为由具体的 Handle 实现决定

```
                    AssetProvider
                         │
              LoadedBundleHandle.LoadAssetAsync(assetInfo)
                         │
          ┌──────────────┼──────────────┐
          │              │              │
   VirtualAssetBundle  AssetBundle   RawBundle
       Handle            Handle        Handle
          │              │              │
   AssetDatabase    AssetBundle.     RawBundle.
   LoadAssetAtPath  LoadAssetAsync   CreateRawFileObject
```

这种设计使得用户代码在所有模式下 API 完全一致，切换 PlayMode 只需改变初始化参数。

---

## 十、与真机模式的核心差异总结

| 维度 | EditorSimulateMode | OfflinePlayMode | HostPlayMode |
|------|:---:|:---:|:---:|
| **文件系统** | EditorFileSystem x1 | BuiltinFileSystem x1 | BuiltinFileSystem + SandboxFileSystem x2 |
| **Bundle 类型** | VirtualAsset/Raw/Archive (12/13/14) | AssetBundle/Raw/Archive (2/3/4) | AssetBundle/Raw/Archive (2/3/4) |
| **清单版本** | 固定 "Simulate" | 构建时指定 | 从 CDN 请求 |
| **资源定位** | 清单中 AssetPath = Assets 原始路径 | AssetPath = 资源内部路径 | AssetPath = 资源内部路径 |
| **Bundle 加载** | EBCLoadVirtual*Operation (无真实文件) | BFSLoadPackageBundleOperation (StreamingAssets) | SFSLoadPackageBundleOperation (缓存+按需下载) |
| **资源加载 API** | `AssetDatabase.LoadAssetAtPath` | `AssetBundle.LoadAssetAsync` | `AssetBundle.LoadAssetAsync` |
| **场景加载** | `EditorSceneManager.LoadSceneInPlayMode` | `SceneManager.LoadSceneAsync` | `SceneManager.LoadSceneAsync` |
| **卸载** | 空操作 或 释放内存 | `AssetBundle.Unload(true)` | `AssetBundle.Unload(true)` |
| **缓存** | 内存字典（仅 GUID） | 文件系统（只读 catalog） | 文件系统（读写 __data + __info） |
| **异步延迟** | 可配帧数模拟 (1-10) | 真实 IO 延迟 | 真实 IO + 网络延迟 |
| **下载** | 可选的虚拟下载模拟 | 不支持 | 真实 HTTP 下载 + 断点续传 |
| **依赖关系** | 无（虚拟 Bundle 独立） | 有 | 有 |
| **加密** | 不支持 | 支持 | 支持 |

---

## 十一、关键文件索引

### 构建管线（Editor 程序集）

| 文件 | 作用 |
|------|------|
| [EditorSimulateBuildPipeline.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/EditorSimulateBuildPipeline.cs) | 模拟构建管线 |
| [EditorSimulateBuildParameters.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/EditorSimulateBuildParameters.cs) | 构建参数（限制为虚拟类型） |
| [TaskPrepare_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskPrepare_ESBP.cs) | 参数验证 |
| [TaskGetBuildMap_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskGetBuildMap_ESBP.cs) | 生成资源映射 |
| [TaskUpdateBundleInfo_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskUpdateBundleInfo_ESBP.cs) | 更新 Bundle 信息 |
| [TaskCreateManifest_ESBP.cs](../Assets/YooAsset/Editor/BundleBuilder/BuildPipeline/EditorSimulateBuildPipeline/BuildTasks/TaskCreateManifest_ESBP.cs) | 生成清单 |
| [BundleSimulateBuilder.cs](../Assets/YooAsset/Editor/BundleBuilder/BundleSimulateBuilder.cs) | 模拟构建入口 |

### 运行时调用入口

| 文件 | 作用 |
|------|------|
| [EditorSimulateBuildInvoker.cs](../Assets/YooAsset/Runtime/PackageBuilder/EditorSimulateBuildInvoker.cs) | 反射调用 Editor 构建 |
| [InitializePackageOptions.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/InitializePackageOptions.cs) | `EditorSimulateModeOptions` 定义 |
| [InitializePackageOperation.cs](../Assets/YooAsset/Runtime/ResourcePackage/Operations/InitializePackageOperation.cs) | 初始化操作 |
| [FileSystemParameters.cs](../Assets/YooAsset/Runtime/FileSystem/FileSystemParameters.cs) | `CreateDefaultEditorFileSystemParameters()` |

### EditorFileSystem

| 文件 | 作用 |
|------|------|
| [EditorFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/EditorFileSystem.cs) | 编辑器文件系统 |
| [EditorFileSystemHelper.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/EditorFileSystemHelper.cs) | `GetEditorFilePath()` 核心方法 |
| [EFSInitializeOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSInitializeOperation.cs) | 初始化操作 |
| [EFSLoadPackageBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSLoadPackageBundleOperation.cs) | Bundle 加载操作 |
| [EFSDownloadBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSDownloadBundleOperation.cs) | 模拟下载操作 |
| [EFSLoadPackageManifestOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSLoadPackageManifestOperation.cs) | 清单加载操作 |
| [EFSRequestPackageVersionOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSRequestPackageVersionOperation.cs) | 版本请求操作 |
| [EFSEnsurePackageBundleOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSEnsurePackageBundleOperation.cs) | Bundle 确保操作 |
| [EFSClearCacheOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/EFSClearCacheOperation.cs) | 缓存清理操作 |
| [SimulateAndCacheFileOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/internal/SimulateAndCacheFileOperation.cs) | 模拟下载并缓存 |
| [LoadEditorPackageManifestOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/internal/LoadEditorPackageManifestOperation.cs) | 加载清单二进制 |
| [LoadEditorPackageHashOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/internal/LoadEditorPackageHashOperation.cs) | 加载清单哈希 |
| [LoadEditorPackageVersionOperation.cs](../Assets/YooAsset/Runtime/FileSystem/Services/EditorFileSystem/Operations/internal/LoadEditorPackageVersionOperation.cs) | 加载清单版本 |

### EditorBundleCache

| 文件 | 作用 |
|------|------|
| [EditorBundleCache.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/EditorBundleCache.cs) | 编辑器缓存系统 |
| [EBCLoadBundleBaseOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCLoadBundleBaseOperation.cs) | 异步模拟延迟基类 |
| [EBCLoadVirtualAssetBundleOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCLoadVirtualAssetBundleOperation.cs) | 创建 VirtualAssetBundleHandle |
| [EBCLoadVirtualRawBundleOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCLoadVirtualRawBundleOperation.cs) | 创建 VirtualRawBundleHandle |
| [EBCLoadVirtualArchiveBundleOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCLoadVirtualArchiveBundleOperation.cs) | 创建 VirtualArchiveBundleHandle |
| [EBCWriteCacheOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCWriteCacheOperation.cs) | 内存缓存写入 |
| [EBCClearCacheOperation.cs](../Assets/YooAsset/Runtime/BundleCache/Services/EditorBundleCache/Operations/EBCClearCacheOperation.cs) | 缓存清理 |

### Virtual BundleHandle

| 文件 | 作用 |
|------|------|
| [VirtualAssetBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/VirtualAssetBundleHandle.cs) | VirtualAssetBundle Handle |
| [VABHLoadAssetOperation.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/Operations/VABHLoadAssetOperation.cs) | AssetDatabase.LoadAssetAtPath |
| [VABHLoadAllAssetsOperation.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/Operations/VABHLoadAllAssetsOperation.cs) | AssetDatabase.LoadMainAssetAtPath |
| [VABHLoadSubAssetsOperation.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/Operations/VABHLoadSubAssetsOperation.cs) | AssetDatabase.LoadAllAssetRepresentationsAtPath |
| [VABHLoadSceneOperation.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualAssetBundleHandle/Operations/VABHLoadSceneOperation.cs) | EditorSceneManager.LoadSceneInPlayMode |
| [VirtualRawBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualRawBundleHandle/VirtualRawBundleHandle.cs) | VirtualRawBundle Handle |
| [VirtualArchiveBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualArchiveBundleHandle/VirtualArchiveBundleHandle.cs) | VirtualArchiveBundle Handle |
| [VirtualArchiveBundle.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualArchiveBundleHandle/VirtualArchiveBundle.cs) | 虚拟归档数据容器 |
| [VARBHLoadAssetOperation.cs](../Assets/YooAsset/Runtime/BundleHandle/Services/VirtualArchiveBundleHandle/Operations/VARBHLoadAssetOperation.cs) | 从归档创建 RawFileObject |

### 公共接口和数据结构

| 文件 | 作用 |
|------|------|
| [EPlayMode.cs](../Assets/YooAsset/Runtime/ResourcePackage/EPlayMode.cs) | PlayMode 枚举 |
| [EBundleType.cs](../Assets/YooAsset/Runtime/BundleHandle/EBundleType.cs) | BundleType 枚举（含 12/13/14 虚拟类型） |
| [IBundleHandle.cs](../Assets/YooAsset/Runtime/BundleHandle/Interfaces/IBundleHandle.cs) | BundleHandle 统一接口 |
| [IFileSystem.cs](../Assets/YooAsset/Runtime/FileSystem/Interfaces/IFileSystem.cs) | 文件系统统一接口 |
| [IBundleCache.cs](../Assets/YooAsset/Runtime/BundleCache/Interfaces/IBundleCache.cs) | 缓存系统统一接口 |
| [PackageManifest.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageManifest.cs) | 资源清单 |
| [PackageBundle.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageBundle.cs) | Bundle 描述 |
| [PackageAsset.cs](../Assets/YooAsset/Runtime/ResourcePackage/PackageAsset.cs) | 资源描述 |
| [BundleInfo.cs](../Assets/YooAsset/Runtime/ResourcePackage/BundleInfo.cs) | Bundle 运行时包装 |
| [AssetInfo.cs](../Assets/YooAsset/Runtime/ResourcePackage/AssetInfo.cs) | 资源信息（含 LoadMethod） |
