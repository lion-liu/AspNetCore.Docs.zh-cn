运行以下 .NET Core CLI 命令：

```dotnetcli
dotnet tool install --global dotnet-ef --version 3.1.9
dotnet tool install --global dotnet-aspnet-codegenerator --version 3.1.4
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 3.1.9
dotnet add package Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore --version 3.1.9
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design --version 3.1.4
dotnet add package Microsoft.EntityFrameworkCore.Design --version 3.1.9
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 3.1.9
dotnet add package Microsoft.Extensions.Logging.Debug --version 3.1.9
```

上述命令添加：

* [aspnet-codegenerator 基架工具](xref:fundamentals/tools/dotnet-aspnet-codegenerator)。
* 适用于 .NET Core CLI 的 Entity Framework Core 工具。
* EF Core SQLite 提供程序将 EF Core 包作为依赖项进行安装。
* 基架需要的包：`Microsoft.VisualStudio.Web.CodeGeneration.Design` 和 `Microsoft.EntityFrameworkCore.SqlServer`。

有关允许应用按环境配置其数据库上下文的多个环境配置指南，请参阅 <xref:fundamentals/environments#environment-based-startup-class-and-methods>。

[!INCLUDE[](~/includes/scaffoldTFM.md)]