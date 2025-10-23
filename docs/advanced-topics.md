# 高级主题

AssetsTools.NET 的高级功能和优化技巧。

## 目录

1. [性能优化](#性能优化)
2. [缓存策略](#缓存策略)
3. [内存管理](#内存管理)
4. [线程安全](#线程安全)
5. [自定义模板生成器](#自定义模板生成器)
6. [低级文件操作](#低级文件操作)

---

## 性能优化

### 启用缓存

缓存显著提高处理许多资产时的性能：

```csharp
var manager = new AssetsManager();

// 启用所有缓存选项
manager.UseTemplateFieldCache = true;        // 缓存类型结构
manager.UseMonoTemplateFieldCache = true;    // 缓存 MonoBehaviour 模板
manager.UseRefTypeManagerCache = true;       // 缓存引用类型管理器
manager.UseQuickLookup = true;               // 使用字典进行 PathId 查找

manager.LoadClassDatabase("classdata.tpk");

// 同时为每个文件启用快速查找
var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");
assetsFile.file.GenerateQuickLookup();
```

**性能影响：**
- `UseTemplateFieldCache`: 重复类型访问快 10-50 倍
- `UseQuickLookup`: O(1) vs O(n) 资产查找
- `GenerateQuickLookup()`: 推荐用于 1000+ 资产的文件

### 批量处理

批量处理资产以最大化缓存命中：

```csharp
manager.UseTemplateFieldCache = true;

// 按类型分组
var textureInfos = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);

foreach (var texInfo in textureInfos)
{
    // 第一个纹理后模板被缓存
    var baseField = manager.GetBaseField(assetsFile, texInfo);
    // 处理...
}
```

### 延迟加载

仅在需要时加载依赖项：

```csharp
// 不要自动加载依赖项
var assetsFile = manager.LoadAssetsFile("level0", loadDeps: false);

// 需要时加载特定依赖项
if (needsDependency)
{
    manager.LoadDependencies(assetsFile);
}
```

### 跳过不必要的反序列化

使用标志跳过不需要的部分：

```csharp
// 获取模板而不反序列化数据
var template = manager.GetTemplateBaseField(assetsFile, assetInfo);

// 跳过 MonoBehaviour 字段
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.SkipMonoBehaviourFields
);
```

---

## 缓存策略

### 何时使用缓存

**在以下情况下使用模板缓存：**
- 处理许多相同类型的资产
- 多次读取同一文件
- 处理大量资产（100+ 资产）

**在以下情况下禁用缓存：**
- 一次性处理
- 低内存环境
- 资产具有不同的结构

### 缓存失效

```csharp
var manager = new AssetsManager();
manager.UseTemplateFieldCache = true;

// 处理第一个文件
var file1 = manager.LoadAssetsFile("file1.assets");
// ... 处理 file1 ...

// 切换到不同的 Unity 版本前清除缓存
manager.UnloadAllAssetsFiles(clearCache: true);

// 加载不同版本
manager.LoadClassDatabaseFromPackage("2019.4.30f1");
var file2 = manager.LoadAssetsFile("file2.assets");
```

---

## 内存管理

### 正确清理

始终在完成后卸载文件：

```csharp
try
{
    var manager = new AssetsManager();
    var assetsFile = manager.LoadAssetsFile("large.assets");

    // 处理文件...
}
finally
{
    // 清理
    manager.UnloadAll();
}
```

### 大文件处理

对于非常大的 bundle，避免自动解压缩：

```csharp
// 不要在内存中解压缩
var bundle = manager.LoadBundleFile("huge.bundle", unpackIfPacked: false);

// 手动解压缩到文件
using (var writer = new AssetsFileWriter("decompressed.bundle"))
{
    bundle.file.Unpack(writer);
}

// 关闭 bundle
manager.UnloadBundleFile(bundle);

// 现在加载解压缩版本
var decompressedBundle = manager.LoadBundleFile("decompressed.bundle");
```

### 流管理

小心处理流：

```csharp
// 错误：流将被释放
using (var stream = File.OpenRead("file.assets"))
{
    var assetsFile = manager.LoadAssetsFile(stream, "file.assets");
    // assetsFile.file.Reader.BaseStream 现在已被释放！
}

// 正确：让 AssetsManager 管理流
var assetsFile = manager.LoadAssetsFile("file.assets");
// 流保持打开直到 UnloadAssetsFile
```

---

## 线程安全

### 从多个线程读取

AssetsManager 在设计时考虑了线程安全，但需要小心：

```csharp
var manager = new AssetsManager();
manager.UseTemplateFieldCache = true;  // 线程安全的并发字典
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("level0");

// 安全：并行读取不同的资产
Parallel.ForEach(assetsFile.file.AssetInfos, info =>
{
    var baseField = manager.GetBaseField(assetsFile, info);
    // 处理 baseField...
});
```

### Reader 锁定

每个 AssetsFileInstance 都有用于其 reader 的锁：

```csharp
// 内部锁定自动处理
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 如果需要，手动锁定
lock (assetsFile.LockReader)
{
    assetsFile.file.Reader.Position = customOffset;
    // 读取自定义数据...
}
```

### 写入不是线程安全的

永远不要从多个线程写入：

```csharp
// 错误：不要这样做
Parallel.ForEach(assets, asset =>
{
    asset.SetNewData(data);  // 不是线程安全的
});

// 正确：顺序写入
foreach (var asset in assets)
{
    asset.SetNewData(data);
}
```

---

## 自定义模板生成器

### 实现 IMonoBehaviourTemplateGenerator

```csharp
using AssetsTools.NET.Extra;

public class MyTemplateGenerator : IMonoBehaviourTemplateGenerator
{
    public AssetTypeTemplateField GetTemplateField(
        AssetTypeTemplateField baseField,
        string assemblyName,
        string nameSpace,
        string className,
        UnityVersion unityVersion)
    {
        // 您的自定义逻辑来生成 MonoBehaviour 模板
        // 这可以从缓存加载、从自定义格式生成等

        var newTemplate = baseField.Clone();

        // 添加自定义字段
        var customField = new AssetTypeTemplateField
        {
            Name = "myCustomField",
            Type = "int",
            ValueType = AssetValueType.Int32,
            IsArray = false,
            IsAligned = false,
            HasValue = true
        };

        newTemplate.Children.Add(customField);
        return newTemplate;
    }

    public void Dispose()
    {
        // 清理资源
    }
}

// 使用
var manager = new AssetsManager();
manager.MonoTempGenerator = new MyTemplateGenerator();
```

---

## 低级文件操作

### 直接使用 AssetsFile

为了获得最大控制，直接使用 AssetsFile：

```csharp
var assetsFile = new AssetsFile();

using (var reader = new AssetsFileReader("file.assets"))
{
    assetsFile.Read(reader);
}

// 直接访问结构
Console.WriteLine($"Unity 版本: {assetsFile.Metadata.UnityVersion}");
Console.WriteLine($"平台: {assetsFile.Metadata.TargetPlatform}");

// 手动反序列化
var info = assetsFile.GetAssetInfo(123);
assetsFile.Reader.Position = info.GetAbsoluteByteOffset(assetsFile);

// 读取原始字节
byte[] rawData = assetsFile.Reader.ReadBytes((int)info.ByteSize);
```

### 自定义序列化

手动序列化资产：

```csharp
var template = new AssetTypeTemplateField
{
    Name = "Base",
    Type = "GameObject",
    Children = new List<AssetTypeTemplateField>
    {
        new AssetTypeTemplateField
        {
            Name = "m_Name",
            Type = "string",
            ValueType = AssetValueType.String
        }
    }
};

var valueField = ValueBuilder.DefaultValueFieldFromTemplate(template);
valueField["m_Name"].AsString = "MyObject";

byte[] data = valueField.WriteToByteArray();
```

### ContentReplacer 模式

实现自定义替换器：

```csharp
public class MyCustomReplacer : IContentReplacer
{
    private byte[] data;

    public MyCustomReplacer(byte[] data)
    {
        this.data = data;
    }

    public ContentReplacerType GetReplacerType() => ContentReplacerType.AddOrModify;

    public long GetSize() => data.Length;

    public long Write(AssetsFileWriter writer, bool compress = false)
    {
        writer.Write(data);
        return data.Length;
    }

    public long WriteReplacer(AssetsFileWriter writer, bool compress = false) => Write(writer, compress);

    public bool HasPreview() => true;

    public Stream GetPreviewStream() => new MemoryStream(data);
}

// 使用
assetInfo.Replacer = new MyCustomReplacer(myData);
```

---

## 性能基准测试

### 测量性能改进

```csharp
using System.Diagnostics;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");
var assetsFile = manager.LoadAssetsFile("large.assets");

// 不使用缓存
var sw = Stopwatch.StartNew();
for (int i = 0; i < 100; i++)
{
    var info = assetsFile.file.AssetInfos[i];
    var baseField = manager.GetBaseField(assetsFile, info);
}
sw.Stop();
Console.WriteLine($"无缓存: {sw.ElapsedMilliseconds}ms");

// 使用缓存
manager.UseTemplateFieldCache = true;
sw.Restart();
for (int i = 0; i < 100; i++)
{
    var info = assetsFile.file.AssetInfos[i];
    var baseField = manager.GetBaseField(assetsFile, info);
}
sw.Stop();
Console.WriteLine($"有缓存: {sw.ElapsedMilliseconds}ms");
```

### 内存分析

```csharp
// 监控内存使用
long memoryBefore = GC.GetTotalMemory(true);

var manager = new AssetsManager();
var assetsFile = manager.LoadAssetsFile("large.assets");
// ... 执行操作 ...

long memoryAfter = GC.GetTotalMemory(false);
Console.WriteLine($"内存使用: {(memoryAfter - memoryBefore) / 1024 / 1024}MB");

manager.UnloadAll();
GC.Collect();
long memoryAfterCleanup = GC.GetTotalMemory(true);
Console.WriteLine($"清理后: {(memoryAfterCleanup - memoryBefore) / 1024 / 1024}MB");
```
