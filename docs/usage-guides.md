# 使用指南

AssetsTools.NET 常见任务实用指南。

## 目录

1. [读取资产文件](#读取资产文件)
2. [处理 Bundle](#处理-bundle)
3. [修改资产](#修改资产)
4. [处理 MonoBehaviour](#处理-monobehaviour)
5. [处理纹理](#处理纹理)
6. [管理依赖关系](#管理依赖关系)
7. [创建新资产](#创建新资产)

---

## 读取资产文件

### 基本读取

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 加载资产文件
var assetsFile = manager.LoadAssetsFile("level0");

// 获取所有 GameObject
var gameObjects = assetsFile.file.GetAssetsOfType(AssetClassID.GameObject);

foreach (var info in gameObjects)
{
    var baseField = manager.GetBaseField(assetsFile, info);
    string name = baseField["m_Name"].AsString;
    bool isActive = baseField["m_IsActive"].AsBool;

    Console.WriteLine($"{name} - 激活: {isActive}");
}
```

### 读取特定资产类型

```csharp
// 纹理
var textures = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);
foreach (var texInfo in textures)
{
    var baseField = manager.GetBaseField(assetsFile, texInfo);
    string name = baseField["m_Name"].AsString;
    int width = baseField["m_Width"].AsInt;
    int height = baseField["m_Height"].AsInt;
}

// 材质
var materials = assetsFile.file.GetAssetsOfType(AssetClassID.Material);
foreach (var matInfo in materials)
{
    var baseField = manager.GetBaseField(assetsFile, matInfo);
    string shaderName = baseField["m_Shader.m_PathID"].AsLong.ToString();
}
```

---

## 处理 Bundle

### 加载和提取

```csharp
var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 加载 bundle
var bundle = manager.LoadBundleFile("mybundle.unity3d");

// 列出 bundle 中的所有文件
foreach (var dirInfo in bundle.file.BlockAndDirInfo.DirectoryInfos)
{
    Console.WriteLine($"文件: {dirInfo.Name}");
}

// 从 bundle 加载第一个资产文件
var assetsFile = manager.LoadAssetsFileFromBundle(bundle, 0);

// 像往常一样处理资产
var gameObjects = assetsFile.file.GetAssetsOfType(AssetClassID.GameObject);
```

### 解压缩 Bundle

```csharp
var bundle = new AssetBundleFile();
using (var reader = new AssetsFileReader("compressed.bundle"))
{
    bundle.Read(reader);
}

// 检查压缩
var compression = bundle.GetCompressionType();
Console.WriteLine($"压缩: {compression}");

// 解压到文件
using (var writer = new AssetsFileWriter("decompressed.bundle"))
{
    bundle.Unpack(writer);
}
```

### 压缩 Bundle

```csharp
var bundle = new AssetBundleFile();
using (var reader = new AssetsFileReader("uncompressed.bundle"))
{
    bundle.Read(reader);
}

// 使用 LZ4 压缩
using (var writer = new AssetsFileWriter("compressed.bundle"))
{
    bundle.Pack(writer, AssetBundleCompressionType.LZ4);
}

// 带进度报告
using (var writer = new AssetsFileWriter("compressed.bundle"))
{
    var progress = new MyProgressReporter();
    bundle.Pack(writer, AssetBundleCompressionType.LZMA, progress: progress);
}

class MyProgressReporter : IAssetBundleCompressProgress
{
    public void SetProgress(float progress)
    {
        Console.WriteLine($"进度: {progress * 100:F1}%");
    }
}
```

---

## 修改资产

### 基本修改

```csharp
var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

// 获取资产
var assetInfo = assetsFile.file.GetAssetInfo(123);
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 修改字段
baseField["m_Name"].AsString = "修改后的名称";
baseField["m_IsActive"].AsBool = true;

// 序列化并保存
byte[] newData = baseField.WriteToByteArray();
assetInfo.SetNewData(newData);

using (var writer = new AssetsFileWriter("sharedassets0_modified.assets"))
{
    assetsFile.file.Write(writer);
}

manager.UnloadAll();
```

### 批量修改

```csharp
// 修改所有 GameObject
var gameObjects = assetsFile.file.GetAssetsOfType(AssetClassID.GameObject);

foreach (var info in gameObjects)
{
    var baseField = manager.GetBaseField(assetsFile, info);

    // 设置全部为激活
    baseField["m_IsActive"].AsBool = true;

    // 应用更改
    byte[] newData = baseField.WriteToByteArray();
    info.SetNewData(newData);
}

// 保存
using (var writer = new AssetsFileWriter("modified.assets"))
{
    assetsFile.file.Write(writer);
}
```

### 修改数组

```csharp
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 访问数组
var arrayField = baseField["m_SomeArray"]["Array"];

// 迭代数组
for (int i = 0; i < arrayField.Children.Count; i++)
{
    var element = arrayField[i];
    element["value"].AsInt = i * 10;
}

// 添加到数组（更复杂 - 需要重建）
var template = arrayField.TemplateField.Children[0]; // 获取元素模板
var newElement = ValueBuilder.DefaultValueFieldFromTemplate(template);
newElement["value"].AsInt = 999;
arrayField.Children.Add(newElement);
```

---

## 处理 MonoBehaviour

### 使用 Mono.Cecil

```csharp
using AssetsTools.NET.MonoCecil;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 设置 MonoCecil 生成器
var generator = new MonoCecilTempGenerator("path/to/Assembly-CSharp.dll");
manager.MonoTempGenerator = generator;

var assetsFile = manager.LoadAssetsFile("level0");

// 获取 MonoBehaviour
var monoBehaviours = assetsFile.file.GetAssetsOfType(AssetClassID.MonoBehaviour);

foreach (var info in monoBehaviours)
{
    var baseField = manager.GetBaseField(assetsFile, info);

    // 访问 MonoBehaviour 字段
    if (baseField["myCustomField"] != null && !baseField["myCustomField"].IsDummy)
    {
        int value = baseField["myCustomField"].AsInt;
        Console.WriteLine($"自定义字段值: {value}");
    }
}

generator.Dispose();
manager.UnloadAll();
```

### 使用 Cpp2IL（用于 IL2CPP 游戏）

```csharp
using AssetsTools.NET.Cpp2IL;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 设置 Cpp2IL 生成器
var generator = new Cpp2IlTempGenerator("path/to/GameAssembly.dll", "path/to/global-metadata.dat");
manager.MonoTempGenerator = generator;

// 像往常一样使用
var assetsFile = manager.LoadAssetsFile("level0");
var monoBehaviours = assetsFile.file.GetAssetsOfType(AssetClassID.MonoBehaviour);
```

---

## 处理纹理

### 使用 AssetsTools.NET.Texture

```csharp
using AssetsTools.NET.Texture;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

// 获取所有纹理
var textures = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);

foreach (var texInfo in textures)
{
    var baseField = manager.GetBaseField(assetsFile, texInfo);

    // 创建 TextureFile
    var textureFile = TextureFile.ReadTextureFile(baseField);

    // 获取纹理信息
    string name = baseField["m_Name"].AsString;
    Console.WriteLine($"纹理: {name}");
    Console.WriteLine($"  格式: {textureFile.m_TextureFormat}");
    Console.WriteLine($"  大小: {textureFile.m_Width}x{textureFile.m_Height}");

    // 解码为 RGBA32
    byte[] rgbaData = textureFile.GetTextureData(assetsFile.file);

    // 使用外部库保存为 PNG
    // （您需要 PNG 编码器，如 StbImageWriteSharp）
}
```

### 替换纹理

```csharp
using AssetsTools.NET.Texture;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

var texInfo = assetsFile.file.GetAssetInfo(texPathId);
var baseField = manager.GetBaseField(assetsFile, texInfo);

// 加载新纹理数据（RGBA32 格式）
byte[] newRgbaData = LoadYourTexture(); // 您的代码
int newWidth = 512;
int newHeight = 512;

// 创建新纹理
var textureFile = TextureFile.ReadTextureFile(baseField);
byte[] encodedData = textureFile.EncodeTexture(
    newRgbaData,
    newWidth,
    newHeight,
    TextureFormat.RGB24
);

// 更新字段
baseField["m_Width"].AsInt = newWidth;
baseField["m_Height"].AsInt = newHeight;
baseField["image data"].AsByteArray = encodedData;

// 保存
byte[] assetData = baseField.WriteToByteArray();
texInfo.SetNewData(assetData);

using (var writer = new AssetsFileWriter("modified.assets"))
{
    assetsFile.file.Write(writer);
}
```

---

## 管理依赖关系

### 加载带依赖项

```csharp
var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 加载并自动解析依赖项
var assetsFile = manager.LoadAssetsFile("level0", loadDeps: true);

// 所有依赖项现已加载
foreach (var external in assetsFile.file.Metadata.Externals)
{
    Console.WriteLine($"依赖项: {external.PathName}");
}
```

### 跟踪引用

```csharp
var gameObjectInfo = assetsFile.file.GetAssetInfo(1);
var goBaseField = manager.GetBaseField(assetsFile, gameObjectInfo);

// 获取 Transform 引用
var transformPtr = goBaseField["m_Transform"];
var transformAsset = manager.GetExtAsset(assetsFile, transformPtr);

if (transformAsset.baseField != null)
{
    var position = transformAsset.baseField["m_LocalPosition"];
    float x = position["x"].AsFloat;
    float y = position["y"].AsFloat;
    float z = position["z"].AsFloat;
}
```

---

## 创建新资产

### 从模板创建

```csharp
var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

// 创建新 GameObject
var gameObjectValue = manager.CreateValueBaseField(
    assetsFile,
    (int)AssetClassID.GameObject
);

// 设置字段
gameObjectValue["m_Name"].AsString = "新游戏对象";
gameObjectValue["m_IsActive"].AsBool = true;
gameObjectValue["m_Layer"].AsInt = 0;

// 序列化
byte[] newAssetData = gameObjectValue.WriteToByteArray();

// 添加到文件（高级 - 需要手动 PathId 管理）
// 这很复杂 - 考虑使用 BundleCreator 创建新资产
```
