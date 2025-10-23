# 代码示例

AssetsTools.NET 常见场景的实用代码示例。

## 目录

1. [列出场景中的所有对象](#列出场景中的所有对象)
2. [提取所有纹理](#提取所有纹理)
3. [查找和替换文本](#查找和替换文本)
4. [批量重命名资产](#批量重命名资产)
5. [在文件间复制资产](#在文件间复制资产)
6. [创建简单 Bundle](#创建简单-bundle)
7. [导出场景层次结构](#导出场景层次结构)

---

## 列出场景中的所有对象

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;

class Program
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        // 加载场景文件
        var sceneFile = manager.LoadAssetsFile("level0");

        // 获取所有 GameObject
        var gameObjects = sceneFile.file.GetAssetsOfType(AssetClassID.GameObject);

        Console.WriteLine($"找到 {gameObjects.Count} 个游戏对象：\n");

        foreach (var goInfo in gameObjects)
        {
            var baseField = manager.GetBaseField(sceneFile, goInfo);

            string name = baseField["m_Name"].AsString;
            bool isActive = baseField["m_IsActive"].AsBool;
            int layer = baseField["m_Layer"].AsInt;

            // 获取 Transform 来查找位置
            var transformPtr = baseField["m_Transform"];
            var transformAsset = manager.GetExtAsset(sceneFile, transformPtr);

            if (transformAsset.baseField != null)
            {
                var position = transformAsset.baseField["m_LocalPosition"];
                float x = position["x"].AsFloat;
                float y = position["y"].AsFloat;
                float z = position["z"].AsFloat;

                Console.WriteLine($"{name}");
                Console.WriteLine($"  激活: {isActive}");
                Console.WriteLine($"  层级: {layer}");
                Console.WriteLine($"  位置: ({x:F2}, {y:F2}, {z:F2})\n");
            }
        }

        manager.UnloadAll();
    }
}
```

---

## 提取所有纹理

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using AssetsTools.NET.Texture;
using System;
using System.IO;

class TextureExtractor
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");
        var textures = assetsFile.file.GetAssetsOfType(AssetClassID.Texture2D);

        Directory.CreateDirectory("extracted_textures");

        Console.WriteLine($"提取 {textures.Count} 个纹理...\n");

        foreach (var texInfo in textures)
        {
            var baseField = manager.GetBaseField(assetsFile, texInfo);
            string name = baseField["m_Name"].AsString;

            try
            {
                var textureFile = TextureFile.ReadTextureFile(baseField);

                // 获取 RGBA 数据
                byte[] rgba = textureFile.GetTextureData(assetsFile.file);

                if (rgba != null)
                {
                    // 保存为 TGA（简单格式，无依赖）
                    string outputPath = $"extracted_textures/{name}.tga";
                    SaveAsTGA(outputPath, rgba, textureFile.m_Width, textureFile.m_Height);

                    Console.WriteLine($"已提取: {name} ({textureFile.m_Width}x{textureFile.m_Height})");
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"提取 {name} 失败: {ex.Message}");
            }
        }

        manager.UnloadAll();
        Console.WriteLine("\n完成！");
    }

    static void SaveAsTGA(string path, byte[] rgba, int width, int height)
    {
        using (var writer = new BinaryWriter(File.Create(path)))
        {
            // TGA 头部
            writer.Write((byte)0);  // ID 长度
            writer.Write((byte)0);  // 颜色映射类型
            writer.Write((byte)2);  // 图像类型（未压缩 RGB）
            writer.Write((short)0); // 颜色映射起始
            writer.Write((short)0); // 颜色映射长度
            writer.Write((byte)0);  // 颜色映射深度
            writer.Write((short)0); // X 起始
            writer.Write((short)0); // Y 起始
            writer.Write((short)width);
            writer.Write((short)height);
            writer.Write((byte)32); // 每像素位数（RGBA）
            writer.Write((byte)8);  // 图像描述符

            // 写入像素数据（将 RGBA 转换为 TGA 的 BGRA）
            for (int i = 0; i < rgba.Length; i += 4)
            {
                writer.Write(rgba[i + 2]); // B
                writer.Write(rgba[i + 1]); // G
                writer.Write(rgba[i + 0]); // R
                writer.Write(rgba[i + 3]); // A
            }
        }
    }
}
```

---

## 查找和替换文本

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;

class TextReplacer
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var assetsFile = manager.LoadAssetsFile("resources.assets");

        // 查找所有 TextAsset 对象
        var textAssets = assetsFile.file.GetAssetsOfType(AssetClassID.TextAsset);

        int replacementCount = 0;

        foreach (var info in textAssets)
        {
            var baseField = manager.GetBaseField(assetsFile, info);

            string name = baseField["m_Name"].AsString;
            string text = baseField["m_Script"].AsString;

            // 查找和替换
            string newText = text.Replace("旧文本", "新文本");

            if (newText != text)
            {
                baseField["m_Script"].AsString = newText;

                byte[] newData = baseField.WriteToByteArray();
                info.SetNewData(newData);

                replacementCount++;
                Console.WriteLine($"已修改: {name}");
            }
        }

        if (replacementCount > 0)
        {
            using (var writer = new AssetsFileWriter("resources_modified.assets"))
            {
                assetsFile.file.Write(writer);
            }

            Console.WriteLine($"\n在 {replacementCount} 个文件中替换了文本");
            Console.WriteLine("已保存到: resources_modified.assets");
        }
        else
        {
            Console.WriteLine("未进行替换");
        }

        manager.UnloadAll();
    }
}
```

---

## 批量重命名资产

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;

class AssetRenamer
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

        // 为所有 GameObject 添加前缀
        var gameObjects = assetsFile.file.GetAssetsOfType(AssetClassID.GameObject);

        foreach (var info in gameObjects)
        {
            var baseField = manager.GetBaseField(assetsFile, info);

            string oldName = baseField["m_Name"].AsString;
            string newName = "已修改_" + oldName;

            baseField["m_Name"].AsString = newName;

            byte[] newData = baseField.WriteToByteArray();
            info.SetNewData(newData);

            Console.WriteLine($"{oldName} -> {newName}");
        }

        using (var writer = new AssetsFileWriter("sharedassets0_renamed.assets"))
        {
            assetsFile.file.Write(writer);
        }

        manager.UnloadAll();
        Console.WriteLine("\n所有游戏对象已重命名！");
    }
}
```

---

## 在文件间复制资产

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;
using System.IO;
using System.Linq;

class AssetCopier
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var sourceFile = manager.LoadAssetsFile("source.assets");
        var targetFile = manager.LoadAssetsFile("target.assets");

        // 从源文件获取特定资产
        var sourceAssetInfo = sourceFile.file.GetAssetInfo(123); // PathID 123
        var baseField = manager.GetBaseField(sourceFile, sourceAssetInfo);

        // 如果需要可以修改
        baseField["m_Name"].AsString = "复制的资产";

        // 序列化
        byte[] assetData = baseField.WriteToByteArray();

        // 在目标文件中查找下一个可用的 PathId
        long nextPathId = 1;
        foreach (var info in targetFile.file.AssetInfos)
        {
            if (info.PathId >= nextPathId)
                nextPathId = info.PathId + 1;
        }

        // 创建新的 AssetFileInfo
        var newInfo = new AssetFileInfo
        {
            PathId = nextPathId,
            TypeId = sourceAssetInfo.TypeId,
            ScriptTypeIndex = sourceAssetInfo.ScriptTypeIndex
        };
        newInfo.SetNewData(assetData);

        // 添加到目标文件
        targetFile.file.Metadata.AssetInfos.Add(newInfo);

        // 保存目标文件
        using (var writer = new AssetsFileWriter("target_modified.assets"))
        {
            targetFile.file.Write(writer);
        }

        manager.UnloadAll();
        Console.WriteLine("资产复制成功！");
    }
}
```

---

## 创建简单 Bundle

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;
using System.Collections.Generic;
using System.IO;

class Program
{
    static void Main()
    {
        // 从资产文件创建简单 bundle
        var bundleCreator = new BundleCreator();

        // 添加文件到 bundle
        bundleCreator.AddAssetFile("sharedassets0.assets", "sharedassets0.assets");
        bundleCreator.AddAssetFile("level0", "level0");

        // 使用 LZ4 压缩创建 bundle
        bundleCreator.CreateBundle(
            "mybundle.unity3d",
            "2020.3.15f1",  // Unity 版本
            AssetBundleCompressionType.LZ4
        );

        Console.WriteLine("Bundle 创建成功！");
    }
}

class BundleCreator
{
    private List<(string sourcePath, string bundlePath)> files = new List<(string, string)>();

    public void AddAssetFile(string sourcePath, string bundlePath)
    {
        files.Add((sourcePath, bundlePath));
    }

    public void CreateBundle(string outputPath, string unityVersion, AssetBundleCompressionType compression)
    {
        var bundle = new AssetBundleFile();

        // 创建头部
        bundle.Header = new AssetBundleHeader
        {
            Signature = "UnityFS",
            Version = 7,
            UnityVersion = unityVersion,
            EngineVersion = "2020.3.15f1"
        };

        // 为每个文件创建目录信息
        var dirInfos = new List<AssetBundleDirectoryInfo>();

        foreach (var (sourcePath, bundlePath) in files)
        {
            byte[] fileData = File.ReadAllBytes(sourcePath);

            var dirInfo = new AssetBundleDirectoryInfo
            {
                Name = bundlePath,
                Offset = 0, // 将被计算
                DecompressedSize = fileData.Length,
                Flags = 0
            };
            dirInfo.SetNewData(new ContentReplacerFromBuffer(fileData));

            dirInfos.Add(dirInfo);
        }

        bundle.BlockAndDirInfo = new AssetBundleBlockAndDirInfo
        {
            DirectoryInfos = dirInfos
        };

        // 写入 bundle
        using (var writer = new AssetsFileWriter(outputPath))
        {
            bundle.Write(writer);
        }

        // 如果需要压缩
        if (compression != AssetBundleCompressionType.None)
        {
            var tempBundle = new AssetBundleFile();
            using (var reader = new AssetsFileReader(outputPath))
            {
                tempBundle.Read(reader);
            }

            using (var writer = new AssetsFileWriter(outputPath + ".tmp"))
            {
                tempBundle.Pack(writer, compression);
            }

            File.Delete(outputPath);
            File.Move(outputPath + ".tmp", outputPath);
        }
    }
}
```

---

## 导出场景层次结构

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;

class SceneHierarchyExporter
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var sceneFile = manager.LoadAssetsFile("level0");

        // 构建层次结构
        var hierarchy = BuildHierarchy(manager, sceneFile);

        // 导出到文本文件
        var sb = new StringBuilder();
        sb.AppendLine("场景层次结构");
        sb.AppendLine("===============\n");

        foreach (var root in hierarchy)
        {
            ExportNode(sb, root, 0);
        }

        File.WriteAllText("scene_hierarchy.txt", sb.ToString());

        manager.UnloadAll();
        Console.WriteLine("层次结构已导出到 scene_hierarchy.txt");
    }

    static List<GameObjectNode> BuildHierarchy(AssetsManager manager, AssetsFileInstance file)
    {
        var allNodes = new Dictionary<long, GameObjectNode>();
        var gameObjects = file.file.GetAssetsOfType(AssetClassID.GameObject);

        // 第一遍：创建所有节点
        foreach (var goInfo in gameObjects)
        {
            var baseField = manager.GetBaseField(file, goInfo);
            var node = new GameObjectNode
            {
                PathId = goInfo.PathId,
                Name = baseField["m_Name"].AsString,
                IsActive = baseField["m_IsActive"].AsBool
            };

            // 获取 Transform
            var transformPtr = baseField["m_Transform"];
            var transformAsset = manager.GetExtAsset(file, transformPtr);

            if (transformAsset.baseField != null)
            {
                var parentPtr = AssetPPtr.FromField(transformAsset.baseField["m_Father"]);
                node.ParentTransformPathId = parentPtr.IsNull() ? -1 : parentPtr.PathId;
            }

            allNodes[goInfo.PathId] = node;
        }

        // 第二遍：构建层次结构
        foreach (var node in allNodes.Values)
        {
            if (node.ParentTransformPathId != -1)
            {
                // 从 Transform PathId 查找父 GameObject
                foreach (var potential in allNodes.Values)
                {
                    var potentialGO = file.file.GetAssetInfo(potential.PathId);
                    var potentialBaseField = manager.GetBaseField(file, potentialGO);
                    var potentialTransformPtr = AssetPPtr.FromField(potentialBaseField["m_Transform"]);

                    if (potentialTransformPtr.PathId == node.ParentTransformPathId)
                    {
                        potential.Children.Add(node);
                        break;
                    }
                }
            }
        }

        // 返回根节点（没有父级）
        var roots = new List<GameObjectNode>();
        foreach (var node in allNodes.Values)
        {
            if (node.ParentTransformPathId == -1)
                roots.Add(node);
        }

        return roots;
    }

    static void ExportNode(StringBuilder sb, GameObjectNode node, int depth)
    {
        string indent = new string(' ', depth * 2);
        string activeStr = node.IsActive ? "" : " [未激活]";

        sb.AppendLine($"{indent}{node.Name}{activeStr}");

        foreach (var child in node.Children)
        {
            ExportNode(sb, child, depth + 1);
        }
    }

    class GameObjectNode
    {
        public long PathId;
        public string Name;
        public bool IsActive;
        public long ParentTransformPathId;
        public List<GameObjectNode> Children = new List<GameObjectNode>();
    }
}
```

---

## 更多示例

### 批量修改材质属性

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;

class MaterialModifier
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");

        // 获取所有材质
        var materials = assetsFile.file.GetAssetsOfType(AssetClassID.Material);

        foreach (var matInfo in materials)
        {
            var baseField = manager.GetBaseField(assetsFile, matInfo);

            string name = baseField["m_Name"].AsString;
            Console.WriteLine($"修改材质: {name}");

            // 修改着色器属性
            var savedProperties = baseField["m_SavedProperties"];

            // 修改颜色
            var colors = savedProperties["m_Colors"]["Array"];
            foreach (var colorPair in colors.Children)
            {
                var colorName = colorPair["first"].AsString;
                if (colorName == "_Color")
                {
                    var color = colorPair["second"];
                    color["r"].AsFloat = 1.0f; // 红色
                    color["g"].AsFloat = 0.0f;
                    color["b"].AsFloat = 0.0f;
                    color["a"].AsFloat = 1.0f;
                }
            }

            // 保存修改
            byte[] newData = baseField.WriteToByteArray();
            matInfo.SetNewData(newData);
        }

        using (var writer = new AssetsFileWriter("sharedassets0_modified.assets"))
        {
            assetsFile.file.Write(writer);
        }

        manager.UnloadAll();
        Console.WriteLine("\n所有材质已修改！");
    }
}
```

### 提取音频文件

```csharp
using AssetsTools.NET;
using AssetsTools.NET.Extra;
using System;
using System.IO;

class AudioExtractor
{
    static void Main()
    {
        var manager = new AssetsManager();
        manager.LoadClassDatabase("classdata.tpk");

        var assetsFile = manager.LoadAssetsFile("sharedassets0.assets");
        var audioClips = assetsFile.file.GetAssetsOfType(AssetClassID.AudioClip);

        Directory.CreateDirectory("extracted_audio");

        foreach (var clipInfo in audioClips)
        {
            var baseField = manager.GetBaseField(assetsFile, clipInfo);

            string name = baseField["m_Name"].AsString;
            var audioData = baseField["m_AudioData"].AsByteArray;

            if (audioData != null && audioData.Length > 0)
            {
                // 保存为原始格式（通常是 FSB5 或 OGG/MP3）
                string outputPath = $"extracted_audio/{name}.dat";
                File.WriteAllBytes(outputPath, audioData);

                Console.WriteLine($"已提取: {name} ({audioData.Length} 字节)");
            }
        }

        manager.UnloadAll();
        Console.WriteLine("\n音频提取完成！");
    }
}
```
