# AssetsTools.NET v3

[![Nuget](https://img.shields.io/nuget/v/AssetsTools.NET?style=flat-square)](https://www.nuget.org/packages/AssetsTools.NET) [![Prereleases](https://img.shields.io/github/v/release/nesrak1/AssetsTools.NET?include_prereleases&style=flat-square)](https://github.com/nesrak1/AssetsTools.NET/releases) [![discord](https://img.shields.io/discord/862035581491478558?label=discord&logo=discord&logoColor=FFFFFF&style=flat-square)](https://discord.gg/hd9VdswwZs)

一个功能完善的 .NET 库，用于读取、分析和修改 Unity 资产文件和 bundle 文件。基于 [UABE](https://github.com/SeriousCache/UABE/) 的 AssetsTools 库。

---

## 功能特性

- **读写资产文件** - 加载、解析和修改 Unity 资产文件（`.assets`、`.resource` 等）
- **Bundle 支持** - 提取、压缩和重新打包 Unity bundle 文件（`.unity3d`、`.bundle` 等）
- **多种压缩格式** - 支持 LZ4、LZMA 和未压缩的 bundle
- **类型系统** - 支持使用 TypeTree 和 ClassDatabase 进行资产反序列化
- **跨文件引用** - 自动解析依赖关系和 PPtr 引用
- **MonoBehaviour 支持** - 使用 Mono.Cecil 或 Cpp2IL 反序列化自定义脚本
- **纹理处理** - 使用 Texture 扩展解码和编码纹理
- **高性能** - 为大文件提供可选的缓存和快速查找机制
- **线程安全** - 考虑了并发访问的设计

## 安装

通过 NuGet 包管理器安装：

```bash
dotnet add package AssetsTools.NET
```

或通过包管理器控制台：

```powershell
Install-Package AssetsTools.NET
```

## 快速开始

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;

// 创建 AssetsManager 实例
var manager = new AssetsManager();

// 加载类数据库以获取类型信息
manager.LoadClassDatabase("classdata.tpk");

// 加载资产文件
var assetsFileInstance = manager.LoadAssetsFile("level0");

// 通过 path ID 获取资产
var assetInfo = assetsFileInstance.file.GetAssetInfo(1);
var baseField = manager.GetBaseField(assetsFileInstance, assetInfo);

// 读取资产数据
string name = baseField["m_Name"].AsString;
Console.WriteLine($"资产名称: {name}");

// 修改资产数据
baseField["m_Name"].AsString = "新名称";

// 保存更改
byte[] newData = baseField.WriteToByteArray();
assetInfo.SetNewData(newData);

using (var writer = new AssetsFileWriter("level0_modified"))
{
    assetsFileInstance.file.Write(writer);
}
```

## 扩展库

- **AssetsTools.NET.Texture** - 纹理解码/编码（BC7、DXT、ETC、ASTC、PVRTC 等）
- **AssetsTools.NET.MonoCecil** - 使用 Mono.Cecil 反序列化 MonoBehaviour
- **AssetsTools.NET.Cpp2IL** - IL2CPP 程序集分析支持

## 文档

- [入门指南](docs/getting-started.md) - 安装和第一步
- [核心概念](docs/core-concepts.md) - 理解 Unity 文件格式
- [API 参考](docs/api-reference.md) - 完整的 API 文档
- [使用指南](docs/usage-guides.md) - 常见任务和工作流程
- [示例代码](docs/examples.md) - 典型场景的代码示例
- [高级主题](docs/advanced-topics.md) - 性能优化和高级功能
- [故障排除](docs/troubleshooting.md) - 常见问题和解决方案

## 使用场景

- **资产提取** - 从 Unity 游戏中提取纹理、音频、模型
- **模组制作** - 修改游戏资产并创建模组
- **分析** - 分析 Unity 项目结构和依赖关系
- **自动化** - 批量处理资产用于内容管线
- **逆向工程** - 了解 Unity 游戏架构

## 系统要求

- .NET Framework 4.6.2+ 或 .NET Core 3.1+ / .NET 5+
- 推荐使用 .NET 6.0+ 以获得最佳性能

## 相关项目

- [UABEA](https://github.com/nesrak1/UABEA/) - 跨平台 Unity Assets Bundle 提取器（带 UI）
- [AssetsView](AssetsView/) - 传统 WinForms 查看器（包含在此仓库中）

## 许可证

MIT 许可证 - 详见 [LICENSE](LICENSE)

## 贡献

欢迎贡献！请随时提交问题和拉取请求。

---

## 工具导航

[![AssetsTools](logo.png)](#assetstools)
[![AssetsView](https://user-images.githubusercontent.com/12544505/73600640-e57e1b00-4518-11ea-8aab-e8664947f435.png)](#assetsview)
[![UABE Avalonia](uabealogo.png)](https://github.com/nesrak1/UABEA/)
