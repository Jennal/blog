# C#中使用Pomelo进行MySql的原生Json查询

Mysql已经原生支持json，有5年多了。但我还没有用过，搜了一圈，[MySql本身的语法](https://dev.mysql.com/doc/refman/8.0/en/json-search-functions.html#function_json-extract)还是很好写的

<pre class="sql">
mysql> SELECT c, JSON_EXTRACT(c, "$.id"), g
     > FROM jemp
     > WHERE JSON_EXTRACT(c, "$.id") > 1
     > ORDER BY JSON_EXTRACT(c, "$.name");
+-------------------------------+-----------+------+
| c                             | c->"$.id" | g    |
+-------------------------------+-----------+------+
| {"id": "3", "name": "Barney"} | "3"       |    3 |
| {"id": "4", "name": "Betty"}  | "4"       |    4 |
| {"id": "2", "name": "Wilma"}  | "2"       |    2 |
+-------------------------------+-----------+------+
3 rows in set (0.00 sec)

mysql> SELECT c, c->"$.id", g
     > FROM jemp
     > WHERE c->"$.id" > 1
     > ORDER BY c->"$.name";
+-------------------------------+-----------+------+
| c                             | c->"$.id" | g    |
+-------------------------------+-----------+------+
| {"id": "3", "name": "Barney"} | "3"       |    3 |
| {"id": "4", "name": "Betty"}  | "4"       |    4 |
| {"id": "2", "name": "Wilma"}  | "2"       |    2 |
+-------------------------------+-----------+------+
3 rows in set (0.00 sec)
</pre>

但是有些语言偏偏搞了很复杂的封装，比如C#。我做过的项目，主要用[Pomelo库](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql)来连接MySql。而Pomelo库曾经做了一套`JsonObject`的方案来使用MySql的json，后来又弃用了。但是网上大量留下了`JsonObject`的教程，这是多么尴尬的局面。最尴尬的是Pomelo库本身并没有编写完整的文档，所以能搜到的信息，基本都是误导信息。经过一番探索，我总结一下目前正统的方案。

### 方案开始

首先安装必要的库，我这里使用的是Microsoft的Json序列化方案，因为它支持递归引用的序列化，所以更推荐用它来做序列化。

<pre class="shell">
dotnet add package Microsoft.EntityFrameworkCore.Design --version 3.1.10
dotnet add package Microsoft.EntityFrameworkCore.Relational --version 3.1.10
dotnet add package Pomelo.EntityFrameworkCore.MySql --version 3.2.4
dotnet add package Pomelo.EntityFrameworkCore.MySql.Json.Microsoft --version 3.2.4
</pre>

然后需要在`Startup.cs`中的`ConfigureServices`，添加以下代码

<pre class="csharp">
services.AddEntityFrameworkMySqlJsonMicrosoft();
</pre>

完整示例

<pre class="csharp">
public void ConfigureServices(IServiceCollection services)
{
    var conn = Configuration.GetConnectionString("Mysql");
    Console.WriteLine(conn);
    services.AddDbContext<piContext>(options => 
        options.UseMySql(conn, x => x.ServerVersion("5.7.26-mysql")));
    services.AddEntityFrameworkMySqlJsonMicrosoft();
    services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
    services.AddControllers()
            .AddNewtonsoftJson();
}
</pre>

最后，在需要使用json查询的位置，使用

<pre class="csharp">
var id = 1;
var result = dbContext.JsonTable.Where(o => EF.Functions.JsonExtract<int>(o.Json, "$.id") == id);
</pre>

就能生成这样的Sql代码

<pre class="sql">
select * from JsonTable where JSON_EXTRACT(Json, "$.id") = 1;
</pre>

### 参考

- [Pomelo测试代码](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql/blob/master/test/EFCore.MySql.FunctionalTests/Query/JsonPocoQueryTestBase.cs#L327)
- [MySql官方关于Json的文档](https://dev.mysql.com/doc/refman/8.0/en/json-search-functions.html#function_json-extract)