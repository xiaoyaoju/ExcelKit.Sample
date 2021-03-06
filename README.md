# ExcelKit

Excel导入导出套件，支持百万级数据导出和读取（由于Excel原因，此处仅限xlsx）而不占用多少内存，方便易用的方法让导入导出更易使用


使用方式：Nuget安装：`Install-Package ExcelKit`

**重要提示：请使用NPOI2.4.1版本



-----

### ExcelKitAttribute详解：

Code：字段编码，一般用于读取时使用，如Name、Age

Desc：字段描述，对应Excel列头中的文本，如 姓名、地址

AllowNull：字段是否允许为空，一般用于读取

Converter：转换器[导出时]，组件中提供了常用的转换器，如需自定义，则继承自IExportConverter<T>并实现方法

ConverterParam：转换器辅助参数[导出时]，导出时使用，如日期格式化导出，导出保留的小数位等；如果是字符串，则多个参数使用|分隔

Sort：字段顺序[导出时]，导出和读取都可能用到

Width：列宽[导出时]，指定Excel列宽度

Align：对齐方式[导出时]，指定Excel列中的文本对齐方式

FontColor：字体颜色[导出时]，指定Excel列中的字体颜色，枚举项

ForegroundColor：前景色[导出时]，指定Excel列的填充色，枚举项

HeadRowFrozen：是否启用表头行冻结[导出时]

HeadRowFilter：是否启用表头行筛选[导出时]

IsIgnore：是否忽略

-----

-----

### Converter详解：


Converter为内置的接口IExportConverter，主要是为了导出使用；目前提供了单泛型参数，双泛型参数的版本。使用者可以根据接口实现自己的Converter。
程序内部提供了常用的Converter，命名空间为：ExcelKit.Core.Infrastructure.Converter ，内置如下：


* BoolConverter（导出后显示为：是   否）
* DateTimeFmtConverter（日期格式化Converter，如需自定义日期格式，需指定ConverterParam，使用详见下方示例）
* DecimalPointDigitConverter（小数类Converter，如需指定保留几位小数，需指定ConverterParam，使用详见下方示例）
* EnumConverter（枚举Converter，需要在枚举上方打上此特性[System.ComponentModel.Description("用户类型")]，导出时就会根据指定的描述展示对应的文字）
* EnumerableConverter（集合类Converter，如字段定义为public List<string> SkuSellRegion { get; set; }则上方Converter = typeof(EnumerableConverter<string>)，导出后会自动拆分为字符串，以，分隔的长文本）

-----



# 1.导出
* 支持并发多Sheet导出
* 单Sheet最大数据量为1048200
* 可直接保存到本地或者生成Excel信息
* 支持导出时自动拆分Sheet，默认达到1048200时，超过的数据会自动用 _{number}向后自动拆分Sheet，也可在CreateSheet时自定义单Sheet大小
* 导出流程为：创建Excel文件 -> 创建Sheet -> 为某个Sheet追加数据 -> 保存或生成Excel信息
* 多Sheet导出时，一定注意创建的Sheet名称，后面AppendData需要指定Sheet名称，两边要一致。
* 并发导出时，一个任务对应一个Sheet

1.1 泛型类型


```csharp

public class UserDto
{
	[ExcelKit(Desc = "账号", Width = 20, IsIgnore = false, Sort = 20, Align = TextAlign.Right, FontColor = DefineColor.LightBlue)]
	public string Account { get; set; }

	[ExcelKit(Desc = "昵称", Width = 50, Sort = 10, FontColor = DefineColor.Rose, ForegroundColor = DefineColor.LemonChiffon)]
	public string Name { get; set; }
	
	[ExcelKit(Desc = "金额", Width = 20, Sort = 10, Converter = typeof(DecimalPointDigitConverter), ConverterParam = 2)]
	public decimal Money { get; set; } = 20;

	[ExcelKit(Desc = "创建时间", Width = 50, Sort = 10, Converter = typeof(DateTimeFmtConverter), ConverterParam = "yyyy-MM-dd")]
	public DateTime CreateDate { get; set; } = DateTime.Now;
}


using (var context = ContextFactory.GetWriteContext("测试导出文件"))
{
    Parallel.For(1, 4, index =>
    {
       var sheet = context.CrateSheet<UserDto>($"Sheet{index}");

       for (int i = 0; i < 1000000; i++)
       {
           sheet.AppendData<UserDto>($"Sheet{index}", new UserDto { Account = $"{index}-{i}-2010211", Name = $"{index}-{i}-用户", CreateDate = DateTime.Now, Money = Convert.ToDouble(i) });
       }
    });

    filePath = context.Save();
    Console.WriteLine($"文件路径：{filePath}");
}

```
    
    
1.2 动态字段类型


```csharp

using (var context = ContextFactory.GetWriteContext("测试导出文件"))
{
    var sheet = context.CrateSheet("Sheet1", new List<ExcelKitAttribute>()
    {
        new ExcelKitAttribute(){ Code = "Account", Desc = "账号",Width=60 },
        new ExcelKitAttribute(){ Code = "Name", Desc = "昵称" }
    }, 5);

    for (int i = 0; i < 10; i++)
    {
        sheet.AppendData("Sheet1", new Dictionary<string, object>()
        {
            {"Account", $"{i}-2010211" }, {"Name", $"{i}-用户用户" }
        });
    }

    filePath = context.Save();
    Console.WriteLine($"文件路径：{filePath}");
}

```

-----

# 2.读取

* 读取主要是按照Sheet索引（默认从1开始）或者Sheet名称（默认Sheet1）
* 目前仅支持单Sheet读取，多Sheet同时读取暂未加入
* 此方式读取时，读取成功的数据在SucData中，读取一行返回一行，故不像一次性全部读取出来那般占内存
* 对于读取失败的数据，ReadXXXOptions中有 FailData ，会返回读取失败的源数据及失败相关信息，方便记录及导出到新的Excel中
* FailData仅仅是读取Excel失败或者转换为目标数据失败才会进FailData，在SucData中的函数本身如果抛错不会进入FailData
* ReadXXXOptions中的DataStartRow（默认从1开始）和DataEndRow（可空不传则读完）代表读取的数据条数位置，不配置采用默认值
* ReadRowsOptions仅仅是读取行数据，数据返回的是一行，没有对应的Key，默认情况下，空单元格会被直接忽略，返回的行数据都是有值的，当需要返回包含空的单元格时，配置ReadEmptyCell为true，同时指定Excel的列信息ColumnHeaders数组，里面的元素为"A" "B" "C"等，即表头列信息，Excel中可看到


2.1 读取行（默认按照Sheet索引读取，此处为读取第一个Sheet）

```csharp

var context = ContextFactory.GetReadContext();
context.ReadRows("测试导出文件.xlsx", new ReadRowsOptions()
{
	SucData = (rowdata, rowindex) =>
    {
		Console.WriteLine(JsonConvert.SerializeObject(rowdata));
	}
});


```

2.2 读取行（可指定Sheet名称或者Sheet索引，此处指定按照Sheet名称读取）

```csharp

var context = ContextFactory.GetReadContext();
context.ReadRows("测试导出文件.xlsx", new ReadRowsOptions()
{
	ReadWay = ReadWay.SheetName,
	SucData = (rowdata, rowindex) =>
	{
		Console.WriteLine(JsonConvert.SerializeObject(rowdata));
	}
});

```

2.3 泛型读取Sheet

```csharp

var context = ContextFactory.GetReadContext();
context.ReadSheet<UserDto>("测试导出文件.xls", new ReadSheetOptions<UserDto>()
{
	SucData = (rowdata, rowindex) =>
	{
		Console.WriteLine(JsonConvert.SerializeObject(rowdata));
	}
});

```

2.4 动态读取Sheet

```csharp

var context = ContextFactory.GetReadContext();
context.ReadSheet("测试导出文件.xlsx", new ReadSheetDicOptions()
{
	DataEndRow = 10,
	ExcelFields = new (string field, ColumnType type, bool allowNull)[]
	{
		("账号",ColumnType.String,false)),("昵称",ColumnType.String,false))
	},
	SucData = (rowdata, rowindex) =>
	{
		Console.WriteLine(JsonConvert.SerializeObject(rowdata));
	}
});
```
