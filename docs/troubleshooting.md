# 故障排除

使用 AssetsTools.NET 时的常见问题和解决方案。

## 目录

1. [常见错误](#常见错误)
2. [版本兼容性问题](#版本兼容性问题)
3. [性能问题](#性能问题)
4. [数据损坏](#数据损坏)
5. [平台特定问题](#平台特定问题)
6. [调试技巧](#调试技巧)

---

## 常见错误

### "Type tree is null" 或 "ClassDatabase is null"

**问题：** 尝试在没有类型信息的情况下反序列化。

**解决方案：**
```csharp
// 确保加载类数据库
manager.LoadClassDatabase("classdata.tpk");

// 或检查 TypeTree 是否存在
if (assetsFile.file.Metadata.TypeTreeEnabled)
{
    // 可以使用 TypeTree
}
else
{
    // 必须使用 ClassDatabase
    if (manager.ClassDatabase == null)
    {
        throw new Exception("需要 ClassDatabase 但未加载！");
    }
}
```

---

### "Asset not found" 或 "PathId does not exist"

**问题：** 尝试访问不存在的资产。

**解决方案：**
```csharp
// 始终检查资产是否存在
var assetInfo = assetsFile.file.Metadata.AssetInfos
    .FirstOrDefault(a => a.PathId == targetPathId);

if (assetInfo == null)
{
    Console.WriteLine($"未找到 PathId {targetPathId} 的资产");
    return;
}

// 或使用 try-catch
try
{
    var assetInfo = assetsFile.file.GetAssetInfo(targetPathId);
}
catch (Exception)
{
    Console.WriteLine("未找到资产");
}
```

---

### "Stream was disposed" 或 "Cannot access disposed object"

**问题：** 文件流过早关闭。

**解决方案：**
```csharp
// 在卸载前不要释放流
// 错误：
using (var stream = File.OpenRead("file.assets"))
{
    var assetsFile = manager.LoadAssetsFile(stream, "file.assets");
} // 流在此处释放！
var baseField = manager.GetBaseField(assetsFile, info); // 错误！

// 正确：
var assetsFile = manager.LoadAssetsFile("file.assets");
var baseField = manager.GetBaseField(assetsFile, info); // 正确
manager.UnloadAll(); // 现在可以安全释放
```

---

### "Field is DUMMY" 或 DummyFieldAccessException

**问题：** 访问不存在的字段。

**解决方案：**
```csharp
// 始终检查虚拟字段
var field = baseField["m_SomeField"];
if (field.IsDummy)
{
    Console.WriteLine("字段不存在");
}
else
{
    var value = field.AsString;
}

// 或使用 try-catch
try
{
    var value = baseField["m_SomeField"].AsString;
}
catch (DummyFieldAccessException)
{
    Console.WriteLine("未找到字段");
}
```

---

### "Index out of range" 访问数组时

**问题：** 数组索引无效。

**解决方案：**
```csharp
var arrayField = baseField["m_SomeArray"]["Array"];

// 先检查数组大小
if (arrayField.Children.Count > index)
{
    var element = arrayField[index];
}
else
{
    Console.WriteLine($"数组只有 {arrayField.Children.Count} 个元素");
}
```

---

### MonoBehaviour 字段缺失

**问题：** MonoBehaviour 反序列化未设置。

**解决方案：**
```csharp
using AssetsTools.NET.MonoCecil;

// 设置 MonoBehaviour 生成器
var generator = new MonoCecilTempGenerator("Assembly-CSharp.dll");
manager.MonoTempGenerator = generator;

// 现在 MonoBehaviour 字段将可用
var baseField = manager.GetBaseField(assetsFile, monoBehaviourInfo);

// 不要忘记释放
generator.Dispose();
```

---

## 版本兼容性问题

### "Failed to read file" 或 "Invalid file format"

**问题：** 文件来自不支持的 Unity 版本。

**解决方案：**
```csharp
// 先检查 Unity 版本
if (AssetsFile.IsAssetsFile("file.assets"))
{
    var assetsFile = new AssetsFile();
    using (var reader = new AssetsFileReader("file.assets"))
    {
        assetsFile.Read(reader);
        Console.WriteLine($"Unity 版本: {assetsFile.Metadata.UnityVersion}");
    }

    // 为该版本加载适当的 ClassDatabase
    manager.LoadClassPackage("classdata.tpk");
    manager.LoadClassDatabaseFromPackage(assetsFile.Metadata.UnityVersion);
}
```

---

### TypeTree 与 ClassDatabase 不匹配

**问题：** Unity 版本的类型定义错误。

**解决方案：**
```csharp
// 可用时优先使用 TypeTree（最准确）
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 强制使用 TypeTree（如果被剥离将失败）
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.None  // 默认：优先 TypeTree
);

// 强制使用 ClassDatabase
var baseField = manager.GetBaseField(
    assetsFile,
    assetInfo,
    AssetReadFlags.ForceFromCldb
);
```

---

### ClassDatabase 缺少新 Unity 版本的类型

**问题：** ClassDatabase 没有较新 Unity 版本的定义。

**解决方案：**

1. 从 [UABEA releases](https://github.com/nesrak1/UABEA/releases) 下载最新的 classdata.tpk
2. 或者如果可用，使用 TypeTree
3. 或从 Unity 头文件生成 ClassDatabase

---

## 性能问题

### 资产加载缓慢

**解决方案：**
```csharp
// 启用缓存
manager.UseTemplateFieldCache = true;
manager.UseMonoTemplateFieldCache = true;
manager.UseQuickLookup = true;

// 为大文件生成快速查找
assetsFile.file.GenerateQuickLookup();

// 按类型处理资产（更好的缓存命中）
var textures = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);
foreach (var texInfo in textures)
{
    // 一起处理所有纹理
}
```

---

### 内存使用过高

**解决方案：**
```csharp
// 完成后卸载文件
manager.UnloadAssetsFile(assetsFile);

// 不要自动加载依赖项
var assetsFile = manager.LoadAssetsFile("level0", loadDeps: false);

// 对于大型 bundle，不要在内存中解压缩
var bundle = manager.LoadBundleFile("huge.bundle", unpackIfPacked: false);

// 改为解压缩到文件
using (var writer = new AssetsFileWriter("decompressed.bundle"))
{
    bundle.file.Unpack(writer);
}

manager.UnloadBundleFile(bundle);
```

---

### 文件写入缓慢

**解决方案：**
```csharp
// 使用缓冲流进行写入
using (var fileStream = File.Create("output.assets"))
using (var bufferedStream = new BufferedStream(fileStream, 1024 * 1024)) // 1 MB 缓冲区
using (var writer = new AssetsFileWriter(bufferedStream))
{
    assetsFile.file.Write(writer);
}
```

---

## 数据损坏

### "修改后的文件无法在 Unity 中加载"

**问题：** 修改期间文件结构被破坏。

**诊断：**
```csharp
// 写入后验证文件
var verifyFile = new AssetsFile();
using (var reader = new AssetsFileReader("modified.assets"))
{
    try
    {
        verifyFile.Read(reader);
        Console.WriteLine("文件结构有效");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"文件已损坏: {ex.Message}");
    }
}
```

**常见原因：**
1. 字段类型不正确（例如，将 int 写为 string）
2. 缺少对齐
3. 数组大小不正确
4. 引用损坏（无效的 PPtr）

**解决方案：**
```csharp
// 始终保留字段类型
var originalType = baseField["m_SomeField"].TemplateField.ValueType;
// 确保新值匹配类型

// 验证序列化大小是否符合预期
byte[] newData = baseField.WriteToByteArray();
Console.WriteLine($"新大小: {newData.Length}, 原始: {assetInfo.ByteSize}");

// 对于数组，正确更新大小
var arrayField = baseField["m_Array"]["Array"];
arrayField.Children.Add(newElement);
// 写入时自动更新大小
```

---

### 修改后资产引用损坏

**问题：** PPtr 引用变得无效。

**解决方案：**
```csharp
// 修改资产时，保留 PPtr
var transformPtr = baseField["m_Transform"];
long fileId = transformPtr["m_FileID"].AsInt;
long pathId = transformPtr["m_PathID"].AsLong;

// 如果不移动资产，保持这些值不变

// 添加新资产时，确保 PathId 唯一
long maxPathId = assetsFile.file.AssetInfos.Max(a => a.PathId);
long newPathId = maxPathId + 1;
```

---

## 平台特定问题

### 大端序与小端序

**问题：** 为错误的平台字节序写入。

**解决方案：**
```csharp
// 检查文件字节序
bool isBigEndian = assetsFile.file.Header.Endianness == 0;

// 写入时，匹配原始文件
using (var writer = new AssetsFileWriter("output.assets"))
{
    writer.BigEndian = isBigEndian;
    assetsFile.file.Write(writer);
}

// 或序列化资产数据时
byte[] data = baseField.WriteToByteArray(bigEndian: isBigEndian);
```

---

### 路径分隔符（Windows vs Linux/Mac）

**问题：** 平台之间路径分隔符不同。

**解决方案：**
```csharp
// 使用 Path.Combine
string path = Path.Combine("folder", "file.assets");

// 规范化来自 Unity 的路径
string unityPath = "Assets/Scenes/Level0.unity";
string normalizedPath = unityPath.Replace('/', Path.DirectorySeparatorChar);
```

---

## 调试技巧

### 启用详细日志

```csharp
// 添加日志以跟踪发生的事情
Console.WriteLine($"加载文件: {filePath}");
var assetsFile = manager.LoadAssetsFile(filePath);
Console.WriteLine($"Unity 版本: {assetsFile.file.Metadata.UnityVersion}");
Console.WriteLine($"资产数量: {assetsFile.file.AssetInfos.Count}");

// 记录每个处理的资产
foreach (var info in assetsFile.file.AssetInfos)
{
    try
    {
        var baseField = manager.GetBaseField(assetsFile, info);
        Console.WriteLine($"已处理 PathId {info.PathId}: {baseField["m_Name"].AsString}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"失败 PathId {info.PathId}: {ex.Message}");
    }
}
```

---

### 检查原始字节

```csharp
// 读取原始资产数据以进行调试
var assetInfo = assetsFile.file.GetAssetInfo(123);
assetsFile.file.Reader.Position = assetInfo.GetAbsoluteByteOffset(assetsFile.file);
byte[] rawData = assetsFile.file.Reader.ReadBytes((int)assetInfo.ByteSize);

// 转储为十六进制进行分析
string hex = BitConverter.ToString(rawData.Take(100).ToArray());
Console.WriteLine($"前 100 字节: {hex}");
```

---

### 比较修改前后

```csharp
// 保存原始数据以进行比较
byte[] originalData = baseField.WriteToByteArray();

// 进行修改
baseField["m_Name"].AsString = "已修改";

// 比较
byte[] modifiedData = baseField.WriteToByteArray();

if (originalData.Length != modifiedData.Length)
{
    Console.WriteLine($"大小已更改: {originalData.Length} -> {modifiedData.Length}");
}

// 查找第一个差异
for (int i = 0; i < Math.Min(originalData.Length, modifiedData.Length); i++)
{
    if (originalData[i] != modifiedData[i])
    {
        Console.WriteLine($"第一个差异在字节 {i}");
        break;
    }
}
```

---

### 使用异常处理进行调试

```csharp
try
{
    var baseField = manager.GetBaseField(assetsFile, assetInfo);
    // ... 执行操作 ...
}
catch (DummyFieldAccessException ex)
{
    Console.WriteLine($"字段访问错误: {ex.Message}");
    // 检查字段结构
    PrintFieldStructure(baseField);
}
catch (Exception ex)
{
    Console.WriteLine($"错误: {ex.GetType().Name}");
    Console.WriteLine($"消息: {ex.Message}");
    Console.WriteLine($"堆栈跟踪: {ex.StackTrace}");
}

void PrintFieldStructure(AssetTypeValueField field, int depth = 0)
{
    string indent = new string(' ', depth * 2);
    Console.WriteLine($"{indent}{field.FieldName} ({field.TypeName})");

    foreach (var child in field.Children)
    {
        PrintFieldStructure(child, depth + 1);
    }
}
```

---

## 获取帮助

如果您仍然遇到困难：

1. **查看示例** - [examples.md](examples.md) 有许多常见场景
2. **查阅 API 参考** - [api-reference.md](api-reference.md) 详细的 API 文档
3. **搜索 GitHub Issues** - 可能有人遇到过同样的问题：https://github.com/nesrak1/AssetsTools.NET/issues
4. **在 Discord 上询问** - 加入 [AssetsTools Discord](https://discord.gg/hd9VdswwZs)
5. **创建 Issue** - 提供：
   - Unity 版本
   - AssetsTools.NET 版本
   - 最小可复现代码
   - 错误消息和堆栈跟踪
   - 如果可能，提供示例文件

---

## 常见问题解答

### Q: 我可以修改加密的资产文件吗？

**A:** AssetsTools.NET 不处理加密。您需要先使用其他工具解密文件。

### Q: 支持哪些 Unity 版本？

**A:** 支持 Unity 3.5 到最新版本。较新版本可能需要更新的 classdata.tpk 文件。

### Q: 我可以创建全新的资产文件吗？

**A:** 可以，但建议修改现有文件或使用 BundleCreator 辅助类。

### Q: 修改后的文件在 Unity 中工作吗？

**A:** 是的，如果正确修改。始终：
- 保留字段类型
- 保持正确的对齐
- 验证 PPtr 引用
- 测试修改后的文件

### Q: 我可以在生产环境中使用此库吗？

**A:** 是的，但要彻底测试您的修改。建议：
- 始终备份原始文件
- 验证输出文件
- 在 Unity 中测试
- 处理错误情况

### Q: 如何处理非常大的文件（>1GB）？

**A:**
- 使用 `unpackIfPacked: false` 避免在内存中解压缩
- 禁用不必要的缓存
- 使用流而不是一次性加载
- 分批处理资产
