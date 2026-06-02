---
title: "C# 序列化与IO深度解析"
tags:
  - csharp
  - serialization
  - xml
  - file-io
  - stream
  - byte
type: language
created: 2026-06-02
source_files:
  - drafts/My_Vault/files/C sharp xml序列化_反序列化.md
  - drafts/My_Vault/files/C sharp xml.md
  - drafts/My_Vault/files/C sharp byte.md
  - drafts/My_Vault/files/C Sharp stream 1.md
  - drafts/My_Vault/02_Knowledge/01_Language/常用工具/01_CSharp_文件处理.md
---

# C# 序列化与IO深度解析

## 1. Stream 抽象

`Stream` 是 .NET 所有 I/O 操作的基础抽象类，表示连续的字节序列：

```csharp
public abstract class Stream : MarshalByRefObject, IDisposable
{
    public abstract bool CanRead { get; }
    public abstract bool CanSeek { get; }
    public abstract bool CanWrite { get; }
    public abstract long Length { get; }
    public abstract long Position { get; set; }

    public abstract int Read(byte[] buffer, int offset, int count);
    public abstract void Write(byte[] buffer, int offset, int count);
    public abstract long Seek(long offset, SeekOrigin origin);
    public abstract void Flush();
    public void Close();  // Dispose 的别名
}
```

### 1.1 关键属性与方法

| 成员 | 说明 |
|------|------|
| `CanRead / CanWrite / CanSeek` | 能力查询，操作前应检查 |
| `Position` | 当前读写位置。**频繁使用前务必重置为 0**，否则可能读到错误位置 |
| `Length` | 流的总字节数 |
| `Read(buffer, offset, count)` | 返回实际读取的字节数，可能小于 count |
| `Seek(offset, origin)` | 定位：`Begin`/`Current`/`End`；`offset` 可为负（向头部移动） |
| `Flush()` | 将缓冲区内容强制写入底层设备 |
| `Close() / Dispose()` | 释放资源；推荐用 `using` 包裹 |

### 1.2 常用 Stream 子类

```csharp
// FileStream —— 文件读写
using var fs = new FileStream("data.bin", FileMode.OpenOrCreate);

// MemoryStream —— 内存字节流
using var ms = new MemoryStream();
writer.WriteTo(ms);
byte[] data = ms.ToArray();

// NetworkStream —— 网络数据流（由 Socket 或 TcpClient 获取）

// BufferedStream —— 包装其他 Stream 提供缓冲
using var bs = new BufferedStream(fs, bufferSize: 8192);

// CryptoStream —— 加密/解密流

// GZipStream —— 压缩/解压流
```

### 1.3 Stream 异步操作

```csharp
// APM 模式（旧）：BeginRead / EndRead
IAsyncResult ar = stream.BeginRead(buffer, 0, buffer.Length, callback, state);
int bytesRead = stream.EndRead(ar);

// TAP 模式（新）：ReadAsync / WriteAsync（推荐）
int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
await stream.WriteAsync(buffer, 0, bytesRead);

// CopyToAsync —— 流复制
using var source = File.OpenRead("input.bin");
using var dest = File.Create("output.bin");
await source.CopyToAsync(dest);
```

---

## 2. 文件 I/O

### 2.1 File / FileInfo

```csharp
// File —— 静态方法（每次调用都做安全检查）
if (File.Exists(path))
{
    string content = File.ReadAllText(path);
    string[] lines = File.ReadAllLines(path);
    byte[] data = File.ReadAllBytes(path);
}

File.WriteAllText(path, "Hello");
File.AppendAllText(path, "\nWorld");

// FileInfo —— 实例方法（适用于多次操作同一文件）
var fi = new FileInfo(path);
if (fi.Exists)
{
    using var stream = fi.OpenRead();
    long size = fi.Length;
}

// Directory / DirectoryInfo
var dir = new DirectoryInfo(folderPath);
foreach (var file in dir.GetFiles("*.txt"))
    Console.WriteLine(file.Name);
```

### 2.2 StreamReader / StreamWriter

```csharp
// StreamWriter —— 写入文本
using var writer = new StreamWriter("output.txt");
writer.WriteLine("Hello, World!");

// StreamReader —— 读取文本
using var reader = new StreamReader("input.txt");
string line;
while ((line = reader.ReadLine()) != null)
    Console.WriteLine(line);
```

---

## 3. Byte / BitConverter / Encoding

### 3.1 BitConverter —— 基础类型 ↔ byte[]

```csharp
// 基础类型 → byte[]
byte[] intBytes = BitConverter.GetBytes(42);         // 4 bytes
byte[] floatBytes = BitConverter.GetBytes(3.14f);     // 4 bytes
byte[] doubleBytes = BitConverter.GetBytes(3.14);     // 8 bytes
byte[] boolBytes = BitConverter.GetBytes(true);       // 1 byte

// byte[] → 基础类型
int value = BitConverter.ToInt32(intBytes, 0);
float f = BitConverter.ToSingle(floatBytes, 0);

// 注意字节序：BitConverter.IsLittleEndian 可查询
```

### 3.2 Encoding —— string ↔ byte[]

```csharp
string text = "你好，世界";

// string → byte[]
byte[] utf8 = Encoding.UTF8.GetBytes(text);        // 最常见
byte[] ascii = Encoding.ASCII.GetBytes("Hello");    // 仅 ASCII
byte[] defaultEnc = Encoding.Default.GetBytes(text); // 系统默认编码（不推荐跨平台使用）

// byte[] → string
string result = Encoding.UTF8.GetString(utf8);
```

### 3.3 数组合并

```csharp
// CopyTo —— 从指定索引开始复制到目标数组
int[] source = { 1, 2, 3 };
int[] dest = new int[5];
source.CopyTo(dest, 2);  // dest = [0, 0, 1, 2, 3]

// Concat + ToArray
int[] merged = source.Concat(new[] { 4, 5 }).ToArray();

// Array.Copy
Array.Copy(source, 0, dest, 0, source.Length);
```

---

## 4. XML 操作

### 4.1 XML 基础结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Root>
    <name>唐老狮</name>
    <age>18</age>
    <Item id="1" num="10"/>
    <Friend>
        <name>小明</name>
        <age>8</age>
    </Friend>
</Root>
```

### 4.2 XmlDocument —— DOM 方式读取

```csharp
XmlDocument xml = new XmlDocument();

// 从文件加载
xml.Load(path);

// 从字符串加载
xml.LoadXml(xmlString);

// 查询单个节点
XmlNode root = xml.SelectSingleNode("Root");
XmlNode nameNode = root.SelectSingleNode("name");
Console.WriteLine(nameNode.InnerText);  // "唐老狮"

// 查询多个同名节点
XmlNodeList friends = root.SelectNodes("Friend");
foreach (XmlNode friend in friends)
{
    Console.WriteLine(friend.SelectSingleNode("name").InnerText);
    Console.WriteLine(friend.SelectSingleNode("age").InnerText);
}

// 读取属性
XmlNode item = root.SelectSingleNode("Item");
string id = item.Attributes["id"].Value;
string num = item.Attributes.GetNamedItem("num").Value;

// 简便路径写法
XmlNode atk = xml.SelectSingleNode("Root/atk");
```

### 4.3 XmlDocument —— 创建与修改

```csharp
// 创建新 XML
XmlDocument xml = new XmlDocument();

// 添加声明
XmlDeclaration decl = xml.CreateXmlDeclaration("1.0", "UTF-8", "");
xml.AppendChild(decl);

// 添加根节点
XmlElement root = xml.CreateElement("Root");
xml.AppendChild(root);

// 添加子节点
XmlElement name = xml.CreateElement("name");
name.InnerText = "Alice";
root.AppendChild(name);

// 添加带属性的节点
XmlElement item = xml.CreateElement("Item");
item.SetAttribute("id", "1");
item.SetAttribute("num", "10");
root.AppendChild(item);

// 保存
xml.Save(path);

// 修改 —— 移除节点
XmlNode target = xml.SelectSingleNode("Root/atk");
target.ParentNode.RemoveChild(target);

// 修改 —— 添加节点
XmlElement speed = xml.CreateElement("speed");
speed.InnerText = "20";
root.AppendChild(speed);

xml.Save(path);
```

### 4.4 存储路径选择

| 路径 | 可读 | 可写 | 打包后可用 |
|------|------|------|-----------|
| `Resources` | ✅ | ❌ | ✅ |
| `StreamingAssets` | ✅ PC | PC 可写 | ✅ |
| `DataPath` | ✅ | ❌ | ❌ (路径变化) |
| `PersistentDataPath` | ✅ | ✅ | ✅ |

---

## 5. XML 序列化 / 反序列化

### 5.1 XmlSerializer 基本用法

```csharp
// 数据类 —— 仅公开成员被序列化
public class PlayerData
{
    [XmlElement("PlayerName")]         // 自定义 XML 元素名
    public string Name { get; set; }

    [XmlAttribute("Level")]            // 序列化为 XML 属性而非元素
    public int Level { get; set; }

    [XmlArray("Inventory")]            // 自定义数组外包裹元素名
    [XmlArrayItem("Slot")]             // 自定义每个元素的标签名
    public List<int> ItemIds { get; set; }

    public int[] Scores;               // 数组自动序列化为子元素

    // 不支持 Dictionary —— 需自定义
}

// 序列化
var player = new PlayerData { Name = "Hero", Level = 42, ItemIds = [1, 2, 3] };

var serializer = new XmlSerializer(typeof(PlayerData));
using var writer = new StreamWriter(path);
serializer.Serialize(writer, player);

// 反序列化
using var reader = new StreamReader(path);
PlayerData loaded = (PlayerData)serializer.Deserialize(reader);
```

### 5.2 序列化注意事项

1. **仅序列化 public 成员** —— private/protected/internal 被忽略
2. **不支持 Dictionary** —— 需通过 `IXmlSerializable` 自定义
3. **必须有无参构造函数** —— 反序列化时通过反射构造实例
4. **List 默认值不会被清空** —— 反序列化时追加，可能产生重复数据
5. **引用为 null 的成员不出现在 XML 中**

### 5.3 IXmlSerializable —— 自定义序列化

```csharp
public class CustomData : IXmlSerializable
{
    public int X { get; set; }
    public string Name { get; set; }

    public XmlSchema GetSchema() => null;

    // 反序列化
    public void ReadXml(XmlReader reader)
    {
        // 方式1：读属性
        X = int.Parse(reader["X"]);

        // 方式2：读子元素
        reader.Read();  // 进入根节点内部
        while (reader.NodeType != XmlNodeType.EndElement)
        {
            if (reader.Name == "Name")
            {
                Name = reader.ReadElementContentAsString();
            }
            else
            {
                reader.Skip();
            }
        }
    }

    // 序列化
    public void WriteXml(XmlWriter writer)
    {
        writer.WriteAttributeString("X", X.ToString());
        writer.WriteElementString("Name", Name);
    }
}
```

### 5.4 自定义序列化 Dictionary

```csharp
public class SerializableDictionary<TKey, TValue>
    : Dictionary<TKey, TValue>, IXmlSerializable
{
    public XmlSchema GetSchema() => null;

    public void ReadXml(XmlReader reader)
    {
        var keySerializer = new XmlSerializer(typeof(TKey));
        var valueSerializer = new XmlSerializer(typeof(TValue));

        reader.Read();  // 跳过根节点
        while (reader.NodeType != XmlNodeType.EndElement)
        {
            TKey key = (TKey)keySerializer.Deserialize(reader);
            TValue value = (TValue)valueSerializer.Deserialize(reader);
            Add(key, value);
        }
    }

    public void WriteXml(XmlWriter writer)
    {
        var keySerializer = new XmlSerializer(typeof(TKey));
        var valueSerializer = new XmlSerializer(typeof(TValue));

        foreach (var kvp in this)
        {
            keySerializer.Serialize(writer, kvp.Key);
            valueSerializer.Serialize(writer, kvp.Value);
        }
    }
}
```

---

## 6. TimeSpan / Stopwatch / Uri

### 6.1 TimeSpan —— 时间间隔

```csharp
var ts = new TimeSpan(1, 30, 0);  // 1:30:00
Console.WriteLine(ts.TotalMinutes);  // 90

TimeSpan duration = ts.Duration();    // 绝对值（如果 ts 为负）
TimeSpan later = ts.Add(TimeSpan.FromHours(2));  // 相加
TimeSpan parsed = TimeSpan.Parse("01:30:00");    // 解析

// 常用创建方式
var t1 = TimeSpan.FromSeconds(30);
var t2 = TimeSpan.FromMilliseconds(500);
```

### 6.2 Stopwatch —— 高精度计时

```csharp
var sw = Stopwatch.StartNew();  // 创建+启动

// ... 执行待测代码 ...

sw.Stop();
Console.WriteLine($"{sw.ElapsedMilliseconds} ms");
Console.WriteLine($"{sw.Elapsed.TotalSeconds:F3} s");

sw.Reset();   // 归零
sw.Restart(); // 归零+启动
```

### 6.3 Uri

```csharp
var uri = new Uri("https://example.com/path?query=value#fragment");
Console.WriteLine(uri.Scheme);    // "https"
Console.WriteLine(uri.Host);      // "example.com"
Console.WriteLine(uri.AbsolutePath);  // "/path"
Console.WriteLine(uri.Query);     // "?query=value"
```

---

## 7. 面试要点

1. **Stream 的 Position 陷阱**：多次使用前务必重置为 0，否则读/写位置错误
2. **Flush 的作用**：强制将缓冲区数据写入底层设备
3. **XmlSerializer 限制**：仅 public 成员、不支持 Dictionary、无参构造函数
4. **Encoding 选择**：UTF8 最通用；ASCII 仅处理英文；Default 平台相关不安全
5. **文件路径选择**：PersistentDataPath 是唯一可读可写且打包后能找到的路径
6. **using 与 IDisposable**：Stream、StreamReader/Writer、XmlReader/Writer 都应用 using 包裹
