# 3 SuperSocket中的几个基本概念

### 3.1 Package Type

`Package Type`即包类型, 这里描述的是数据包的结构, 例如SuperSocket中就提供了一些基础的包类型`TextPackageInfo`等.

```csharp
public class TextPackageInfo
{
    public string Text{get; set;}
}
```

这里的`TextPackageInfo`标识了这类类型的数据包中, 仅包含了一个字符串, 当然, 我们通常会有更复杂的网络数据包结构.例如, 我将在下列展示一个包含首尾标识的通信包, 它包含了首尾标识, 消息号, 终端Id, 以及消息体:

```csharp
public class SamplePackage
{
    public byte Begin{get; set;}

    public MessageId MessageId{get; set;}

    public string TerminalId{get; set;}

    public SampleBody Body{get; set;}

    public byte End{get; set;}
}
```

当然, 在SuperSocket中也提供了一些接口供我们实现一些类似格式的包, 不过个人不太喜欢这种方式, 官方文档也举了一些例子, 例如,有的包会有一个特殊的字段来代表此包内容的类型. 我们将此字段命名为 "Key". 此字段也告诉我们用何种逻辑处理此类型的包. 这是在网络应用程序中非常常见的一种设计. 例如，你的 Key 字段是整数类型，你的包类型需要实现接口`IKeyedPackageInfo`：

```csharp
public class MyPackage : IKeyedPackageInfo<int>
{
    public int Key { get; set; }

    public short Sequence { get; set; }

    public string Body { get; set; }
}
```

### 3.2 PipelineFilter Type

这种类型在网络协议解析中作用重要. 它定义了如何将 IO 数据流解码成可以被应用程序理解的数据包. 换句话说, 就是把你的二进制流数据, 能够一包一包的识别出来, 同时可以解析成你构建的Package对象. 当然, 你也可以选择不构建, 然后将源数据直接返回.

这些是 PipelineFilter 的基本接口. 你的系统中至少需要一个实现这个接口的 PipelineFilter 类型. 
```csharp
public interface IPipelineFilter
{
    void Reset();

    object Context { get; set; }        
}

public interface IPipelineFilter<TPackageInfo> : IPipelineFilter
    where TPackageInfo : class
{

    IPackageDecoder<TPackageInfo> Decoder { get; set; }

    TPackageInfo Filter(ref SequenceReader<byte> reader);

    IPipelineFilter<TPackageInfo> NextFilter { get; }

}
```
事实上，由于 SuperSocket 已经提供了一些内置的 PipelineFilter 模版，这些几乎可以覆盖 90% 的场景的模版极大的简化了你的开发工作. 所以你不需要完全从头开始实现 PipelineFilter. 即使这些内置的模版无法满足你的需求，完全自己实现PipelineFilter也不是难事. 

### 3.3 使用PackageType和PipelineFilter Type创建SuperSocket

你定义好 Package 类型和 PipelineFilter 类型之后，你就可以使用 SuperSocketHostBuilder 创建 SuperSocket 宿主了。
```csharp
var host = SuperSocketHostBuilder.Create<StringPackageInfo, CommandLinePipelineFilter>();
```

在某些情况下，你可能需要实现接口 IPipelineFilterFactory 来完全控制 PipelineFilter 的创建。
```csharp
public class MyFilterFactory : PipelineFilterFactoryBase<TextPackageInfo>
{
    protected override IPipelineFilter<TPackageInfo> CreateCore(object client)
    {
        return new FixedSizePipelineFilter(10);
    }
}
```
然后在 SuperSocket 宿主被创建出来之后启用这个 PipelineFilterFactory:
```csharp
var host = SuperSocketHostBuilder.Create<StringPackageInfo>();
host.UsePipelineFilterFactory<MyFilterFactory>();
```