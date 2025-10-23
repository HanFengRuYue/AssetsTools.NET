# 入门指南

AssetsTools.NET 快速入门教程。

## 目录

1. [安装](#安装)
2. [基本概念](#基本概念)
3. [第一个程序](#第一个程序)
4. [理解工作流程](#理解工作流程)
5. [处理 Bundle](#处理-bundle)
6. [下一步](#下一步)

---

## 安装

### 通过 NuGet 安装（推荐）

安装 AssetsTools.NET 最简单的方法是通过 NuGet 包管理器：

```bash
dotnet add package AssetsTools.NET
```

或在 Visual Studio 的包管理器控制台中使用：

```powershell
Install-Package AssetsTools.NET
```

### 安装扩展包

根据您的需求，您可能还需要安装扩展包：

```bash
# 纹理处理
dotnet add package AssetsTools.NET.Texture

# 通过 Mono.Cecil 支持 MonoBehaviour
dotnet add package AssetsTools.NET.MonoCecil

# IL2CPP 支持
dotnet add package AssetsTools.NET.Cpp2IL
```

### 手动安装

或者，您可以从 [GitHub Releases](https://github.com/nesrak1/AssetsTools.NET/releases) 下载最新版本，并在项目中直接引用 DLL。

---

## 基本概念

在深入代码之前，让我们了解关键概念：

### 1. 资产文件（Assets Files）

Unity 将游戏对象、组件和资源存储在**资产文件**中（通常扩展名为 `.assets`、`.resource`、`.resS`）。这些文件包含：

- **AssetFileInfo** - 每个资产的元数据（ID、类型、大小、偏移量）
- **TypeTree** - 资产类型的结构信息（可选）
- **原始数据** - 资产的实际二进制数据

### 2. Bundle 文件

Unity bundle 是将多个资产文件打包在一起的**容器文件**。它们支持压缩（LZ4、LZMA）并可以包含：

- 一个或多个资产文件
- 资源文件
- 附加数据

### 3. AssetsManager

`AssetsManager` 类是处理资产的主要入口点。它：

- 加载和管理资产文件和 bundle
- 处理文件之间的依赖关系
- 提供高级资产访问 API

### 4. 类型系统

资产可以使用两种方法反序列化：

- **TypeTree** - 嵌入在资产文件中的结构（较新的 Unity 版本）
- **ClassDatabase** - 外部类型定义（旧版 Unity 或 TypeTree 被剥离时需要）

---

## 第一个程序

让我们创建一个简单的程序来读取资产文件并打印资产名称。

### 步骤 1：创建新项目

```bash
dotnet new console -n MyAssetsReader
cd MyAssetsReader
dotnet add package AssetsTools.NET
```

### 步骤 2：编写代码

将 `Program.cs` 的内容替换为：

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;

class Program
{
    static void Main(string[] args)
    {
        // 创建 AssetsManager 实例
        var manager = new AssetsManager();

        // 加载类数据库以获取类型信息
        // 您可以从以下地址下载 classdata.tpk：
        // https://github.com/nesrak1/UABEA/blob/master/classdata.tpk
        manager.LoadClassDatabase("classdata.tpk");

        // 加载资产文件
        // 替换为您实际的资产文件路径
        var assetsFileInstance = manager.LoadAssetsFile("sharedassets0.assets");

        // 获取所有 GameObject 资产
        var gameObjectInfos = assetsFileInstance.file.GetAssetsOfType(AssetClassID.GameObject);

        Console.WriteLine($"找到 {gameObjectInfos.Count} 个游戏对象：");

        foreach (var info in gameObjectInfos)
        {
            // 反序列化资产
            var baseField = manager.GetBaseField(assetsFileInstance, info);

            // 读取 m_Name 字段
            string name = baseField["m_Name"].AsString;

            Console.WriteLine($"  - {name} (PathID: {info.PathId})");
        }

        // 清理
        manager.UnloadAll();

        Console.WriteLine("\n按任意键退出...");
        Console.ReadKey();
    }
}
```

### 步骤 3：运行程序

```bash
dotnet run
```

您应该看到资产文件中所有游戏对象的列表。

---

## 理解工作流程

处理资产的典型工作流程是：

```
1. 创建 AssetsManager
   ↓
2. 加载 ClassDatabase（可选，但推荐）
   ↓
3. 加载资产文件或 Bundle
   ↓
4. 通过 PathID 或类型获取资产信息
   ↓
5. 将资产反序列化为 AssetTypeValueField
   ↓
6. 读取或修改字段
   ↓
7. （可选）将更改写回
   ↓
8. 卸载并清理
```

### 示例：修改资产

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System.IO;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

var assetsFile = manager.LoadAssetsFile("level0");
var assetInfo = assetsFile.file.GetAssetInfo(1); // 获取 PathID = 1 的资产

// 读取资产
var baseField = manager.GetBaseField(assetsFile, assetInfo);

// 修改字段
baseField["m_Name"].AsString = "修改后的名称";
baseField["m_IsActive"].AsBool = true;

// 序列化修改后的数据
byte[] newData = baseField.WriteToByteArray();

// 使用新数据更新资产信息
assetInfo.SetNewData(newData);

// 写入新文件
using (var writer = new AssetsFileWriter("level0_modified"))
{
    assetsFile.file.Write(writer);
}

manager.UnloadAll();
```

---

## 处理 Bundle

如果您的资产在 bundle 文件中：

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;

var manager = new AssetsManager();
manager.LoadClassDatabase("classdata.tpk");

// 加载 bundle
var bundleInstance = manager.LoadBundleFile("mybundle.unity3d");

// 从 bundle 中加载资产文件
var assetsFile = manager.LoadAssetsFileFromBundle(bundleInstance, 0); // 索引 0

// 现在像往常一样处理资产文件
var gameObjects = assetsFile.file.GetAssetsOfType(AssetClassID.GameObject);

foreach (var info in gameObjects)
{
    var baseField = manager.GetBaseField(assetsFile, info);
    Console.WriteLine(baseField["m_Name"].AsString);
}

manager.UnloadAll();
```

---

## 下一步

现在您已经了解了基础知识，请探索以下主题：

- [核心概念](core-concepts.md) - 深入了解 Unity 文件格式
- [API 参考](api-reference.md) - 完整的 API 文档
- [使用指南](usage-guides.md) - 常见任务和工作流程
- [示例代码](examples.md) - 更多代码示例
