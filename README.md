# Dapper Pro
+ Dapper
+ Dapper.Contrib
+ Transaction
+ Stored Procedure
+ Table Value Parameters (TVP)
+ DataTable (C#)

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
