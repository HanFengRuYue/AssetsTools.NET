# API 参考

AssetsTools.NET v3 完整 API 文档。

## 目录

1. [AssetsManager](#assetsmanager)
2. [AssetsFile 和 AssetBundleFile](#assetsfile-和-assetbundlefile)
3. [AssetTypeValueField](#assettypevaluefield)
4. [AssetFileInfo](#assetfileinfo)
5. [辅助类](#辅助类)
6. [枚举和常量](#枚举和常量)

---

## AssetsManager

管理资产文件和 bundle 的主要类。

### 命名空间

```csharp
using AssetsTools.NET.Extra;
```

### 构造函数

```csharp
public AssetsManager()
```

创建新的 AssetsManager 实例。

---

### 属性

#### UseTemplateFieldCache

```csharp
public bool UseTemplateFieldCache { get; set; }
```

缓存非 MonoBehaviour 类型的模板字段。在读取同一类型的许多资产时提高性能。

**默认值:** `false`

#### UseMonoTemplateFieldCache

```csharp
public bool UseMonoTemplateFieldCache { get; set; }
```

缓存 MonoBehaviour 模板字段。对于有许多相同 MonoBehaviour 类型实例的游戏很有用。

**默认值:** `false`

#### UseQuickLookup

```csharp
public bool UseQuickLookup { get; set; }
```

使用字典进行更快的 PathId 资产查找，而不是顺序搜索。

**默认值:** `false`

#### ClassDatabase

```csharp
public ClassDatabaseFile ClassDatabase { get; private set; }
```

当前加载的类数据库。

#### MonoTempGenerator

```csharp
public IMonoBehaviourTemplateGenerator MonoTempGenerator { get; set; }
```

MonoBehaviour 反序列化的模板生成器。使用 `MonoCecilTempGenerator` 或 `Cpp2IlTempGenerator`。

#### Files / Bundles

```csharp
public List<AssetsFileInstance> Files { get; private set; }
public List<BundleFileInstance> Bundles { get; private set; }
```

所有已加载的资产文件和 bundle 列表。

---

### 方法

#### LoadClassDatabase

```csharp
public void LoadClassDatabase(string path)
public void LoadClassDatabase(Stream stream)
```

加载类数据库文件（.tpk 格式）。

**示例:**
```csharp
manager.LoadClassDatabase("classdata.tpk");
```

#### LoadAssetsFile

```csharp
public AssetsFileInstance LoadAssetsFile(string path, bool loadDeps = false)
public AssetsFileInstance LoadAssetsFile(Stream stream, string path, bool loadDeps = false)
```

加载资产文件。

**参数:**
- `path` - 文件路径
- `loadDeps` - 自动加载依赖项？

**示例:**
```csharp
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");
var assetsFileWithDeps = manager.LoadAssetsFile("level0", loadDeps: true);
```

#### LoadBundleFile

```csharp
public BundleFileInstance LoadBundleFile(string path, bool unpackIfPacked = true)
```

加载 bundle 文件。

**参数:**
- `path` - 文件路径
- `unpackIfPacked` - 如果已压缩则自动解压缩？

**示例:**
```csharp
var bundle = manager.LoadBundleFile("mybundle.unity3d");
```

#### LoadAssetsFileFromBundle

```csharp
public AssetsFileInstance LoadAssetsFileFromBundle(BundleFileInstance bunInst, int index, bool loadDeps = false)
public AssetsFileInstance LoadAssetsFileFromBundle(BundleFileInstance bunInst, string name, bool loadDeps = false)
```

从 bundle 中加载资产文件。

**示例:**
```csharp
var bundle = manager.LoadBundleFile("mybundle.unity3d");
var assetsFile = manager.LoadAssetsFileFromBundle(bundle, 0);
// 或
var assetsFile = manager.LoadAssetsFileFromBundle(bundle, "CAB-123456789");
```

#### GetBaseField

```csharp
public AssetTypeValueField GetBaseField(AssetsFileInstance inst, AssetFileInfo info, AssetReadFlags readFlags = AssetReadFlags.None)
public AssetTypeValueField GetBaseField(AssetsFileInstance inst, long pathId, AssetReadFlags readFlags = AssetReadFlags.None)
```

将资产反序列化为值字段树。

**示例:**
```csharp
var assetInfo = assetsFile.file.GetAssetInfo(123);
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 或通过 PathId
var baseField = manager.GetBaseField(assetsFile, 123);

// 带标志
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.ForceFromCldb | AssetReadFlags.PreferEditor
);
```

#### GetExtAsset

```csharp
public AssetExternal GetExtAsset(AssetsFileInstance relativeTo, AssetTypeValueField ptrField)
```

解析 PPtr 引用到外部资产。

**示例:**
```csharp
var scriptPtrField = baseField["m_Script"];
var scriptAsset = manager.GetExtAsset(assetsFile, scriptPtrField);

if (scriptAsset.baseField != null)
{
    string className = scriptAsset.baseField["m_ClassName"].AsString;
}
```

#### CreateValueBaseField

```csharp
public AssetTypeValueField CreateValueBaseField(AssetsFileInstance inst, int typeId, ushort scriptIndex = 0xffff)
```

创建新的值字段。用于创建新资产。

**示例:**
```csharp
// 创建新 GameObject 并使用默认值
var gameObjectValue = manager.CreateValueBaseField(
    assetsFile,
    (int)AssetClassID.GameObject
);
```

#### UnloadAll

```csharp
public void UnloadAll(bool unloadClassData = false)
```

卸载所有资产、bundle 以及可选的类数据库。

**示例:**
```csharp
manager.UnloadAll(unloadClassData: true);
```

---

## AssetsFile 和 AssetBundleFile

低级类直接处理文件。

### AssetsFile

#### 属性

```csharp
public AssetsFileHeader Header { get; set; }
public AssetsFileMetadata Metadata { get; set; }
public IList<AssetFileInfo> AssetInfos { get; }
```

#### 主要方法

##### Write

```csharp
public void Write(AssetsFileWriter writer, long filePos = 0)
```

写入资产文件。应用所有通过 `AssetFileInfo.SetNewData()` 设置的资产修改。

**示例:**
```csharp
using (var writer = new AssetsFileWriter("modified.assets"))
{
    assetsFile.Write(writer);
}
```

##### GetAssetsOfType

```csharp
public List<AssetFileInfo> GetAssetsOfType(AssetClassID typeId)
public List<AssetFileInfo> GetAssetsOfType(int typeId, ushort scriptIndex)
```

获取特定类型的所有资产。

**示例:**
```csharp
var textures = assetsFile.GetAssetsOfType(AssetClassID.Texture2D);
var monoBehaviours = assetsFile.GetAssetsOfType(AssetClassID.MonoBehaviour, scriptIndex: 5);
```

### AssetBundleFile

#### 主要方法

##### Unpack

```csharp
public void Unpack(AssetsFileWriter writer)
```

解压缩并写入未压缩的 bundle。

##### Pack

```csharp
public void Pack(AssetsFileWriter writer, AssetBundleCompressionType compType, bool blockDirAtEnd = true)
```

压缩并写入 bundle。

**参数:**
- `compType` - 压缩类型（None、LZMA、LZ4、LZ4Fast）

##### GetCompressionType

```csharp
public AssetBundleCompressionType GetCompressionType()
```

获取使用的压缩算法。

---

## AssetTypeValueField

表示树结构中的反序列化资产数据。

### 索引器

#### 字符串索引器

```csharp
public AssetTypeValueField this[string name] { get; }
```

按名称访问子字段。支持点号表示法进行嵌套访问。

**示例:**
```csharp
string name = baseField["m_Name"].AsString;
float x = baseField["m_LocalPosition.x"].AsFloat;
```

#### 整数索引器

```csharp
public AssetTypeValueField this[int index] { get; }
```

按索引访问子级（用于数组）。

### 值访问器

获取/设置原始值的便捷属性：

```csharp
public bool AsBool { get; set; }
public sbyte AsSByte { get; set; }
public byte AsByte { get; set; }
public short AsShort { get; set; }
public ushort AsUShort { get; set; }
public int AsInt { get; set; }
public uint AsUInt { get; set; }
public long AsLong { get; set; }
public ulong AsULong { get; set; }
public float AsFloat { get; set; }
public double AsDouble { get; set; }
public string AsString { get; set; }
public byte[] AsByteArray { get; set; }
```

**示例:**
```csharp
// 读取
string name = baseField["m_Name"].AsString;
int layer = baseField["m_Layer"].AsInt;

// 写入
baseField["m_Name"].AsString = "新名称";
baseField["m_Layer"].AsInt = 5;
```

### 方法

#### WriteToByteArray

```csharp
public byte[] WriteToByteArray(bool bigEndian = false)
```

序列化为字节数组。

**示例:**
```csharp
byte[] assetData = baseField.WriteToByteArray();
assetInfo.SetNewData(assetData);
```

#### Clone

```csharp
public AssetTypeValueField Clone()
```

深度克隆此字段和所有子级。

---

## AssetFileInfo

关于单个资产的元数据。

### 属性

```csharp
public long PathId { get; set; }                  // 唯一 ID
public long ByteOffset { get; set; }              // 数据部分偏移量
public uint ByteSize { get; set; }                // 字节大小
public int TypeId { get; set; }                   // 资产类型
```

### 方法

#### SetNewData

```csharp
public void SetNewData(byte[] data)
```

为此资产设置新数据。写入文件时更改生效。

**示例:**
```csharp
var baseField = manager.GetBaseField(assetsFile, assetInfo);
baseField["m_Name"].AsString = "已修改";

byte[] newData = baseField.WriteToByteArray();
assetInfo.SetNewData(newData);

using (var writer = new AssetsFileWriter("output.assets"))
{
    assetsFile.file.Write(writer);
}
```

---

## 辅助类

### AssetHelper

常见资产操作的静态工具方法。

```csharp
// 快速获取资产名称，无需完整反序列化
public static string GetAssetNameFast(AssetsFile file, ClassDatabaseFile classFile, AssetFileInfo info)

// 获取 MonoBehaviour 脚本信息
public static AssetsFileScriptInfo GetAssetsFileScriptInfo(AssetsManager am, AssetsFileInstance inst, ushort scriptIndex)
```

### BundleHelper

Bundle 操作的静态工具。

```csharp
// 从 bundle 加载资产数据
public static byte[] LoadAssetDataFromBundle(AssetBundleFile bundle, int index)

// 从 bundle 加载并反序列化资产
public static AssetTypeValueField LoadAssetFromBundle(AssetBundleFile bundle, AssetsManager am, int index)
```

---

## 枚举和常量

### AssetClassID

常见 Unity 资产类型：

```csharp
public enum AssetClassID
{
    GameObject = 1,
    Transform = 4,
    Camera = 20,
    Material = 21,
    Texture2D = 28,
    Mesh = 43,
    Shader = 48,
    AudioClip = 83,
    MonoBehaviour = 114,
    MonoScript = 115,
    Sprite = 213,
    // ... 还有更多
}
```

### AssetReadFlags

控制资产反序列化的标志：

```csharp
[Flags]
public enum AssetReadFlags
{
    None = 0,
    ForceFromCldb = 1,           // 强制使用 ClassDatabase
    PreferEditor = 2,            // 优先使用 ClassDatabase 的编辑器字段
    SkipMonoBehaviourFields = 4  // 不反序列化 MonoBehaviour 字段
}
```

### AssetBundleCompressionType

Bundle 压缩算法：

```csharp
public enum AssetBundleCompressionType
{
    None,      // 无压缩
    LZMA,      // LZMA 压缩（最高压缩率）
    LZ4,       // LZ4 压缩（平衡）
    LZ4Fast    // LZ4 快速变体
}
```
