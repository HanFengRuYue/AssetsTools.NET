# 核心概念

深入理解 AssetsTools.NET 和 Unity 文件格式。

## 目录

1. [Unity 文件格式概述](#unity-文件格式概述)
2. [资产文件结构](#资产文件结构)
3. [Bundle 文件结构](#bundle-文件结构)
4. [类型系统](#类型系统)
5. [序列化和反序列化](#序列化和反序列化)
6. [依赖关系和引用](#依赖关系和引用)
7. [资产标识符](#资产标识符)

---

## Unity 文件格式概述

Unity 使用多种文件格式来存储游戏数据：

| 格式 | 描述 | 典型扩展名 |
|------|------|-----------|
| **资产文件** | 包含序列化的 Unity 对象 | `.assets`、`.resource`、`.resS` |
| **Bundle 文件** | 支持压缩的多文件容器 | `.unity3d`、`.bundle`、`.assetbundle` |
| **场景文件** | 包含场景数据的特殊资产文件 | 无特定扩展名 |
| **资源文件** | 单独存储大型二进制数据 | `.resource`、`.resS` |

### 文件关系

```
Bundle 文件
├── 资产文件 1
│   ├── GameObject
│   ├── Transform
│   └── ...
├── 资产文件 2
│   ├── Material
│   ├── Texture2D
│   └── ...
└── 资源文件
    └── 原始纹理/音频数据
```

---

## 资产文件结构

资产文件由三个主要部分组成：

### 1. 头部（Header）

头部包含基本文件信息：

```csharp
public class AssetsFileHeader
{
    public uint MetadataSize;        // 元数据部分大小
    public long FileSize;            // 总文件大小
    public uint Version;             // 资产文件版本（例如，Unity 2020 为 22）
    public long DataOffset;          // 资产数据开始位置
    public byte Endianness;          // 0 = 大端序，1 = 小端序
}
```

**版本：**
- 版本 9-21：Unity 3.5 - Unity 2019
- 版本 22+：Unity 2020+

### 2. 元数据（Metadata）

元数据部分包含：

- **Unity 版本** - 引擎版本字符串
- **目标平台** - 平台标识符
- **TypeTree** - 资产类型的结构定义（可选）
- **资产信息** - 文件中所有资产的索引
- **外部引用** - 对其他文件的引用
- **脚本类型** - MonoBehaviour 脚本信息

```csharp
public class AssetsFileMetadata
{
    public string UnityVersion;                          // 例如 "2020.3.15f1"
    public uint TargetPlatform;                          // 平台 ID
    public bool TypeTreeEnabled;                         // 是否有 type tree？
    public List<TypeTreeType> TypeTreeTypes;             // 类型定义
    public List<AssetFileInfo> AssetInfos;               // 资产索引
    public List<AssetsFileExternal> Externals;           // 依赖项
    public List<AssetsFileExternalType> ScriptTypes;     // MonoBehaviour 信息
}
```

### 3. 数据部分（Data Section）

数据部分包含所有资产的原始二进制数据。每个资产的位置由其 `AssetFileInfo` 定义：

```csharp
public class AssetFileInfo
{
    public long PathId;              // 文件内的唯一标识符
    public long ByteOffset;          // 从数据部分开始的偏移量
    public uint ByteSize;            // 资产数据大小
    public int TypeId;               // 资产类型 ID（例如，1 = GameObject）
    public ushort ScriptTypeIndex;   // MonoBehaviour 的 ScriptTypes 索引
}
```

---

## Bundle 文件结构

Bundle 文件是可以容纳多个文件并支持压缩的容器。

### UnityFS 格式

现代 Unity 使用 "UnityFS" 格式（版本 6+）：

```csharp
public class AssetBundleFile
{
    public AssetBundleHeader Header;                    // Bundle 元数据
    public AssetBundleBlockAndDirInfo BlockAndDirInfo;  // 块和文件信息
    public AssetsFileReader DataReader;                 // 解压缩的数据
}
```

### 头部

```csharp
public class AssetBundleHeader
{
    public string Signature;              // "UnityFS" 或 "UnityWeb" 等
    public uint Version;                  // Bundle 格式版本
    public string UnityVersion;           // 引擎版本
    public string EngineVersion;          // 修订版本
    public AssetBundleFSHeader FileStreamHeader;
}
```

### 压缩

Bundle 支持多种压缩算法：

| 类型 | 描述 | 使用场景 |
|------|------|---------|
| **None** | 未压缩 | 快速访问，文件较大 |
| **LZMA** | 高压缩率 | 文件更小，解压较慢 |
| **LZ4** | 平衡 | 良好的折衷，Unity 默认 |

```csharp
// 检查 bundle 压缩
AssetBundleCompressionType compressionType = bundle.GetCompressionType();

// 解压缩 bundle
using (var writer = new AssetsFileWriter("output.bundle"))
{
    bundle.Unpack(writer);
}

// 压缩 bundle
using (var writer = new AssetsFileWriter("output.bundle"))
{
    bundle.Pack(writer, AssetBundleCompressionType.LZ4);
}
```

---

## 类型系统

AssetsTools.NET 支持两种反序列化资产的类型系统：

### 1. TypeTree

TypeTree 是嵌入在资产文件本身中的结构信息（较新的 Unity 版本）：

```csharp
public class TypeTreeType
{
    public int TypeId;                    // 资产类 ID
    public bool IsStripped;               // 此类型是否被剥离？
    public ushort ScriptTypeIndex;        // 用于 MonoBehaviour
    public Hash128 ScriptIdHash;          // 脚本标识符
    public Hash128 TypeHash;              // 类型结构哈希
    public List<TypeTreeNode> Nodes;      // 结构定义
}
```

**优点：**
- 自包含（无需外部文件）
- 对特定 Unity 版本总是准确的

**缺点：**
- 在生产版本中可能被剥离
- 文件大小较大

### 2. ClassDatabase

ClassDatabase 是包含多个 Unity 版本类型定义的外部文件：

```csharp
public class ClassDatabaseFile
{
    public ClassDatabaseFileHeader Header;
    public string UnityVersion;
    public List<ClassDatabaseType> Types;
}
```

**优点：**
- 在 TypeTree 被剥离时仍可工作
- 一个文件涵盖多个 Unity 版本
- 可以包含仅编辑器字段

**缺点：**
- 需要单独的文件
- 可能不匹配自定义引擎修改

### 在 TypeTree 和 ClassDatabase 之间选择

```csharp
// AssetsManager 首先尝试 TypeTree，然后回退到 ClassDatabase
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 强制使用 ClassDatabase
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.ForceFromCldb
);

// 从 ClassDatabase 优先使用编辑器字段
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.PreferEditor
);
```

---

## 序列化和反序列化

### AssetTypeTemplateField

模板字段定义资产的结构：

```csharp
public class AssetTypeTemplateField
{
    public string Name;                   // 字段名称（例如 "m_Name"）
    public string Type;                   // 类型名称（例如 "string"）
    public AssetValueType ValueType;      // 原始类型枚举
    public bool IsArray;                  // 是否是数组？
    public bool IsAligned;                // 之后需要对齐？
    public bool HasValue;                 // 有原始值？
    public List<AssetTypeTemplateField> Children;  // 嵌套字段
}
```

### AssetTypeValueField

值字段包含实际的反序列化数据：

```csharp
public class AssetTypeValueField
{
    public AssetTypeTemplateField TemplateField;  // 结构定义
    public AssetTypeValue Value;                  // 原始值
    public List<AssetTypeValueField> Children;    // 嵌套值
}
```

### 字段访问

```csharp
// 获取 GameObject 资产
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 简单字段访问
string name = baseField["m_Name"].AsString;
bool isActive = baseField["m_IsActive"].AsBool;

// 嵌套字段访问（点号表示法）
int x = baseField["m_Transform.m_Position.x"].AsInt;

// 数组访问
var components = baseField["m_Component"]["Array"];
for (int i = 0; i < components.Children.Count; i++)
{
    var component = components[i];
    var componentPtr = component["component"];
    // 处理组件...
}

// 修改字段
baseField["m_Name"].AsString = "新名称";
baseField["m_IsActive"].AsBool = true;

// 序列化回字节
byte[] newData = baseField.WriteToByteArray();
```

---

## 依赖关系和引用

### 资产引用（PPtr）

Unity 使用 **PPtr**（持久指针）来引用其他资产：

```csharp
public class AssetPPtr
{
    public int FileId;     // 0 = 同一文件，>0 = 外部文件索引
    public long PathId;    // 目标文件中的资产路径 ID
}
```

### 跨文件引用

```csharp
// 读取 PPtr 字段
var scriptRef = baseField["m_Script"];
var pptr = AssetPPtr.FromField(scriptRef);

if (!pptr.IsNull())
{
    if (pptr.FileId == 0)
    {
        // 同一文件中的引用
        var targetAsset = assetsFile.file.GetAssetInfo(pptr.PathId);
    }
    else
    {
        // 外部文件中的引用
        var externalFile = assetsFile.GetDependency(manager, pptr.FileId - 1);
        var targetAsset = externalFile.file.GetAssetInfo(pptr.PathId);
    }
}
```

### 使用 GetExtAsset

AssetsManager 提供辅助方法自动解析引用：

```csharp
// 解析外部资产引用
var scriptField = baseField["m_Script"];
var externalAsset = manager.GetExtAsset(assetsFile, scriptField);

if (externalAsset.baseField != null)
{
    // 访问引用的资产
    string assemblyName = externalAsset.baseField["m_AssemblyName"].AsString;
    string className = externalAsset.baseField["m_ClassName"].AsString;
}
```

### 依赖关系管理

```csharp
// 加载文件及其依赖项
var assetsFile = manager.LoadAssetsFile("level0", loadDeps: true);

// 依赖项会自动加载
// 通过 Externals 列表访问
foreach (var external in assetsFile.file.Metadata.Externals)
{
    Console.WriteLine($"依赖项: {external.PathName}");
}
```

---

## 资产标识符

### PathId

文件中的每个资产都有唯一的 **PathId**（64 位整数）：

```csharp
// 通过 PathId 获取资产
var assetInfo = assetsFile.file.GetAssetInfo(123);

// 迭代所有资产
foreach (var info in assetsFile.file.AssetInfos)
{
    Console.WriteLine($"PathId: {info.PathId}, 类型: {info.TypeId}");
}
```

### TypeId（AssetClassID）

资产按 **TypeId** 分类：

```csharp
public enum AssetClassID
{
    Object = 0,
    GameObject = 1,
    Component = 2,
    Transform = 4,
    Material = 21,
    Texture2D = 28,
    Mesh = 43,
    Shader = 48,
    MonoBehaviour = 114,
    MonoScript = 115,
    // ... 还有更多
}

// 获取特定类型的所有资产
var textures = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);

// 对于 MonoBehaviour，还可以按脚本索引过滤
var specificMonos = assetsFile.file.GetAssetsOfType(
    AssetClassID.MonoBehaviour,
    scriptIndex: 5
);
```

### 脚本类型索引

MonoBehaviour 使用额外的 **ScriptTypeIndex** 来区分不同的脚本类型：

```csharp
// 获取脚本信息
var scriptInfo = AssetHelper.GetAssetsFileScriptInfo(manager, assetsFile, scriptIndex);
Console.WriteLine($"类: {scriptInfo.Namespace}.{scriptInfo.ClassName}");
Console.WriteLine($"程序集: {scriptInfo.AsmName}");
```
