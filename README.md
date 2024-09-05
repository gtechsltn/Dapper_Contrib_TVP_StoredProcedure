# Dapper Pro
+ Dapper
+ Dapper.Contrib
+ Connection and SqlConnection (IDbConnection)
+ Transaction and SqlTransaction (IDbTransaction)
	+ [SqlTransaction Class](https://learn.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqltransaction)
	+ [SqlTransaction Class](https://learn.microsoft.com/en-us/dotnet/api/microsoft.data.sqlclient.sqltransaction)
+ Stored Procedure
+ Table Value Parameters (TVP)
+ [DataTable (C#)](https://learn.microsoft.com/en-us/dotnet/api/system.data.datatable)
+ Dapper with Entity Framework (EF6 or EF Core)
+ DbContext

```
public interface IDbContextFactory<TContext> where TContext : DbContext
```

## Dapper and Transaction
```
using (var transaction = connection.BeginTransaction())
{
    try
    {
        connection.Execute("INSERT INTO Employees (Name, Age) VALUES (@Name, @Age)", new { Name = "John", Age = 30 }, transaction);
        connection.Execute("INSERT INTO Employees (Name, Age) VALUES (@Name, @Age)", new { Name = "Jane", Age = 35 }, transaction);
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

## Dapper and Stored Procedure

### Stored Procedure
```
CREATE PROCEDURE GetCustomerByID
    @CustomerID INT
AS
BEGIN
    SELECT * FROM Customers WHERE CustomerID = @CustomerID
END
```

### Call Stored Procedure with Dapper
```
using Dapper;
using System.Data.SqlClient;

string connectionString = "Data Source=SERVERNAME;Initial Catalog=DATABASENAME;Integrated Security=True;";
int customerID = 1;

using (SqlConnection connection = new SqlConnection(connectionString))
{
    var parameters = new { CustomerID = customerID };
    var result = connection.Query<Customer>("GetCustomerByID", parameters, commandType: System.Data.CommandType.StoredProcedure);
}
```

## Dapper and CRUD with Dapper.Contrib
Dapper.Contrib is an open-source library that provides a set of extension methods to Dapper, which can help you to perform CRUD (Create, Read, Update, Delete) operations more efficiently. With Dapper.Contrib, you can use simple methods like Insert, Update, Delete, and Get to perform these operations instead of writing complex SQL queries.

### Special Attributes
```
[Table ("Customers")]
public class Customer
{
    public int CustomerID { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```
### CRUD with Dapper.Contrib
```
connection.Insert(new Customer { Name = "John", Age = 30 });
connection.Update(new Customer { CustomerID = 1, Name = "Jane", Age = 35 });
connection.Delete(new Customer { CustomerID = 1 });
var customer = connection.Get<Customer>(1);
```

## Using Table-Valued Parameters with Dapper in .NET

### Table-Valued Parameters
```
CREATE TYPE [dbo].[udtt_Project] AS TABLE
(
    [Id] int NULL,
    [Name] nvarchar(4000) NOT NULL,
    [ProjectStartDate] DateTimeOffset NULL,
    [Active] bit NOT NULL, 
    [Draft] bit NOT NULL
)
```

### Stored Procedure
```
CREATE PROCEDURE [dbo].[ProcedureWhichAcceptsProjectsAsTVP]
    @projects [dbo].[udtt_Project] readonly
AS
BEGIN
    SELECT count(*)
    FROM @projects;
END
```

### C# class
```
public class Project
{
    public int? Id { get; set; }
    public string Name { get; set; } = String.Empty;
    public DateTimeOffset? ProjectStartDate { get; set; }
    public bool Active { get; set; }
    public bool Draft { get; set; }
}
```

### C# call Stored Procedure using TVP
```
using var conn = new SqlConnection(connectionString);
conn.Open();
 
// Some example data
List<Project> projects = new()
{
    new Project { Id = 1, Name = "Name1", ProjectStartDate = DateTimeOffset.Parse("2022-11-01"), Active = true, Draft = false },
    new Project { Id = 2, Name = "Name2", ProjectStartDate = DateTimeOffset.Parse("2022-12-01"), Active = false, Draft = true }
};
 
// Create DataTable
DataTable projectsDT = new();
projectsDT.Columns.Add(nameof(Project.Id), typeof(int));
projectsDT.Columns.Add(nameof(Project.Name), typeof(string));
projectsDT.Columns.Add(nameof(Project.ProjectStartDate), typeof(DateTimeOffset));
projectsDT.Columns.Add(nameof(Project.Active), typeof(bool));
projectsDT.Columns.Add(nameof(Project.Draft), typeof(bool));
 
// Add rows to DataTable
foreach (var project in projects)
{
    var row = projectsDT.NewRow();
    row[nameof(Project.Id)] = project.Id ?? (object)DBNull.Value;
    row[nameof(Project.Name)] = project.Name;
    row[nameof(Project.ProjectStartDate)] = project.ProjectStartDate ?? (object)DBNull.Value;
    row[nameof(Project.Active)] = project.Active;
    row[nameof(Project.Draft)] = project.Draft;
    projectsDT.Rows.Add(row);
}
 
// Create parameters
var parameters = new
{
    projects = projectsDT.AsTableValuedParameter("[dbo].[udtt_Project]")
};
 
// Execute Stored Procedure
return await conn.ExecuteScalarAsync<int>(
    "[dbo].[ProcedureWhichAcceptsProjectsAsTVP]",
    param: parameters,
    commandType: CommandType.StoredProcedure);
```

### Dapper automatically generated SQL script 
```
declare @p1 dbo.udtt_Project
insert into @p1 values(1,N'Name1','2022-11-01 00:00:00 +01:00',1,0)
insert into @p1 values(2,N'Name2','2022-12-01 00:00:00 +01:00',0,1)

exec [dbo].[ProcedureWhichAcceptsProjectsAsTVP] @projects=@p1
```

## Dapper and ADO.NET
```
public interface IEmployeeRepository
{
    Employee GetById(int id);
    IEnumerable<Employee> GetAll();
}

public class EmployeeRepository : IEmployeeRepository
{
    private readonly IDbConnection _db;
    public EmployeeRepository(IDbConnection db)
    {
        _db = db;
    }
    public Employee GetById(int id)
    {
        return _db.QuerySingleOrDefault<Employee>("SELECT * FROM Employees WHERE EmployeeId = @EmployeeId", new { EmployeeId = id });
    }
    public IEnumerable<Employee> GetAll()
    {
        return _db.Query<Employee>("SELECT * FROM Employees");
    }
}
```

## Dapper and ADO.NET
```
using Dapper;
using System.Data;

using (IDbConnection connection = new SqlConnection(connectionString))
{
    var result = connection.Query<Product>("SELECT * FROM Products WHERE CategoryID = @CategoryID", new { CategoryID = 1 });
    foreach (var product in result)
    {
        Console.WriteLine(product.ProductName);
    }
}
```

## Dapper and Entity Framework (EF6 or EF Core)
```
public interface IOrderRepository
{
    IEnumerable<Order> GetOrders();
}

public class OrderRepository : IOrderRepository
{
    private readonly DbContext _dbContext;
 
    public OrderRepository(DbContext dbContext)
    {
        _dbContext = dbContext;
    }
 
    public IEnumerable<Order> GetOrders()
    {
        const string query = @"
            SELECT *
            FROM Orders";
 
        using (var connection = _dbContext.Database.GetDbConnection())
        {
            connection.Open();
            return connection.Query<Order>(query);
        }
    }
}

public interface IOrderService
{
    IEnumerable<Order> GetOrders();
}

public class OrderService : IOrderService
{
    private readonly OrderRepository _orderRepository;
 
    public OrderService(OrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }
 
    public IEnumerable<Order> GetOrders()
    {
        return _orderRepository.GetOrders();
    }
}
```

## IDbContext Interface
+ https://subscription.packtpub.com/book/programming/9781785883309/1/ch01lvl1sec15/implementing-the-repository-pattern
+ https://git.hyvatech.com/tabakhpour/ifc/-/blob/1486b81a6189059a2ae73910a82ae74940cef97c/Libraries/Nop.Data/IDbContext.cs

```
public interface IDbContext : IDisposable
```

## IDbContextFactory<TContext> Interface
+ https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.idbcontextfactory-1?view=efcore-8.0
+ https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.dbcontext?view=efcore-8.0

```
public interface IDbContextFactory<TContext> where TContext : DbContext
```

## DbContext Class
```
public class DbContext : IAsyncDisposable, IDisposable, Microsoft.EntityFrameworkCore.Infrastructure.IInfrastructure<IServiceProvider>, Microsoft.EntityFrameworkCore.Internal.IDbContextDependencies, Microsoft.EntityFrameworkCore.Internal.IDbContextPoolable, Microsoft.EntityFrameworkCore.Internal.IDbSetCache
```

## Find the maximum or minimum value from different columns in a table of the same data type
+ https://www.mssqltips.com/sqlservertip/4067/find-max-value-from-multiple-columns-in-a-sql-server-table/

```
IF (OBJECT_ID('tempdb..##TestTable') IS NOT NULL)
	DROP TABLE ##TestTable

CREATE TABLE ##TestTable
(
	ID INT IDENTITY(1,1) PRIMARY KEY,
	Name NVARCHAR(40),
	UpdateByApp1Date DATETIME,
	UpdateByApp2Date DATETIME,
	UpdateByApp3Date DATETIME
)

INSERT INTO ##TestTable(Name, UpdateByApp1Date, UpdateByApp2Date, UpdateByApp3Date )
VALUES('ABC',			'2015-08-05',	'2015-08-04',	'2015-08-06'),
	  ('NewCopmany',	'2014-07-05',	'2012-12-09',	'2015-08-14'),
	  ('MyCompany',		'2015-03-05',	'2015-01-14',	'2015-07-26')
	  
SELECT * FROM ##TestTable

/*
 * Find the maximum or minimum value from different columns in a table of the same data type.
 */
SELECT 
   ID, 
   (SELECT MAX(LastUpdateDate)
      FROM (VALUES (UpdateByApp1Date),(UpdateByApp2Date),(UpdateByApp3Date)) AS UpdateDate(LastUpdateDate)) 
   AS LastUpdateDate
FROM ##TestTable
```

# References
+ https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql/table-valued-parameters
+ https://sii.pl/blog/en/using-table-valued-parameters-with-dapper-in-net/
+ https://www.ssw.com.au/rules/migrate-from-edmx-to-ef-core/
+ https://docs.google.com/document/d/1PL2eMklOxoS3jvxeC9JB6RuEAN9wNE4I0Xp_XdKzVOE/ (manhng83@gmail.com) (gtechsltn@gmail.com)
