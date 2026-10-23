Let's understand just the first step: how your frontend asks the backend which table and columns exist, and how the backend returns that information to the frontend.

We will focus only on `LoadReportMetadata()` for now. We won't cover report generation or filters yet.

# 1. Understand the complete flow

Step 1 — Frontend

Your `ReportBuilder.aspx` page calls `LoadReportMetadata()` using AJAX.

It asks: "Which table and columns are available?"

HTTP POST request

Step 2 — ASP.NET backend

The C# `LoadReportMetadata()` function receives the request and queries Oracle's column metadata.

Oracle query and results

Step 3 — Oracle database

Oracle returns the column names and data types for `PROJECT_DETAILS`.

C# builds a response

Step 4 — Frontend receives response

JavaScript reads `response.d` and uses the returned columns to build the Attributes list.

# 2. Frontend: call the backend function

In your `ReportBuilder.aspx`, you have JavaScript that can call the C# WebMethod.

```
$.ajax({
    type: "POST",
    url: "ReportBuilder.aspx/LoadReportMetadata",
    data: "{}",
    contentType: "application/json; charset=utf-8",
    dataType: "json",

    success: function (response) {
        var result = response.d;

        console.log(result.tables);
    },

    error: function (xhr, status, error) {
        console.log(xhr.responseText);
        alert("Unable to load report metadata.");
    }
});
```

Let's understand each important line.

| Code                                           | Meaning                                                                             |
| ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `type: "POST"`                                 | Sends an HTTP POST request.                                                         |
| `url: "ReportBuilder.aspx/LoadReportMetadata"` | Specifies the ASP.NET page and the C# WebMethod to call.                            |
| `data: "{}"`                                   | Sends an empty JSON object because this function doesn't need any input parameters. |
| `contentType`                                  | Tells ASP.NET that the request body contains JSON.                                  |
| `dataType: "json"`                             | Tells jQuery to expect a JSON response.                                             |
| `success`                                      | Runs when the request succeeds.                                                     |
| `error`                                        | Runs when the request fails.                                                        |

Important: You are not sending a table name from the frontend in this request. The backend already knows which table to inspect: `PROJECT_DETAILS`.

The frontend is asking for the table's metadata, not sending a selected table to the backend.

# 3. Backend: `LoadReportMetadata()`

This is the C# function that receives the request.

```
[WebMethod]
public static object LoadReportMetadata()
{
    List<ColumnInfo> columns = new List<ColumnInfo>();

    using (OracleConnection connection =
        new OracleConnection(GetConnectionString()))
    {
        connection.Open();

        string sql =
            "SELECT COLUMN_NAME, DATA_TYPE " +
            "FROM USER_TAB_COLUMNS " +
            "WHERE TABLE_NAME = 'PROJECT_DETAILS' " +
            "ORDER BY COLUMN_ID";

        using (OracleCommand command =
            new OracleCommand(sql, connection))
        using (OracleDataReader reader = command.ExecuteReader())
        {
            while (reader.Read())
            {
                string columnName =
                    reader["COLUMN_NAME"].ToString();

                string oracleType =
                    reader["DATA_TYPE"].ToString();

                ColumnInfo column = new ColumnInfo();

                column.columnName = ToCamelCase(columnName);
                column.displayName = ToDisplayName(columnName);
                column.dataType = oracleType;
                column.filterType =
                    GetFilterType(columnName, oracleType);

                column.filterValues =
                    GetFilterValues(columnName, column.filterType);

                columns.Add(column);
            }
        }
    }

    TableInfo table = new TableInfo();

    table.tableName = "projectDetails";
    table.displayName = "Project Details";
    table.isDefault = true;
    table.columns = columns;

    List<TableInfo> tables = new List<TableInfo>();
    tables.Add(table);

    return new { tables = tables };
}
```

Let's break it down.

## Step A — Create a list for the columns

```
List<ColumnInfo> columns = new List<ColumnInfo>();
```

This creates an empty list in C#.

At this point, it contains nothing:

```
columns = []
```

As Oracle returns column information, we will add each column to this list.

`ColumnInfo` is a C# class we created to hold the details of one column:

```
public class ColumnInfo
{
    public string columnName { get; set; }
    public string displayName { get; set; }
    public string dataType { get; set; }
    public string filterType { get; set; }
    public List<string> filterValues { get; set; }
}
```

For example, one `ColumnInfo` object might eventually look like this:

```
{
  "columnName": "projectName",
  "displayName": "PROJECT NAME",
  "dataType": "VARCHAR2",
  "filterType": "text",
  "filterValues": []
}
```

The exact display name depends on the formatting you choose in `ToDisplayName()`.

## Step B — Connect to Oracle

```
using (OracleConnection connection =
    new OracleConnection(GetConnectionString()))
{
    connection.Open();
}
```

`GetConnectionString()` reads your `OracleDb` connection string from `Web.config`.

`connection.Open()` establishes the database connection.

The `using` statement ensures that the connection is disposed of when the block finishes.

## Step C — Ask Oracle for the table's columns

```
string sql =
    "SELECT COLUMN_NAME, DATA_TYPE " +
    "FROM USER_TAB_COLUMNS " +
    "WHERE TABLE_NAME = 'PROJECT_DETAILS' " +
    "ORDER BY COLUMN_ID";
```

This SQL does not fetch the project records. It fetches information about the table structure.

For example, if your Oracle table has these columns, Oracle might return:

| COLUMN_NAME  | DATA_TYPE |
| ------------ | --------- |
| PROJECT_ID   | NUMBER    |
| PROJECT_NAME | VARCHAR2  |
| START_DATE   | DATE      |
| COST         | NUMBER    |
| CATEGORY     | VARCHAR2  |
| PLACE        | VARCHAR2  |
| STATUS       | VARCHAR2  |

`USER_TAB_COLUMNS` is an Oracle data dictionary view that describes columns in tables accessible under the current user's schema.

`ORDER BY COLUMN_ID` returns the columns in their defined order.

## Step D — Read each column one by one

```
using (OracleCommand command =
    new OracleCommand(sql, connection))
using (OracleDataReader reader = command.ExecuteReader())
{
    while (reader.Read())
    {
        string columnName =
            reader["COLUMN_NAME"].ToString();

        string oracleType =
            reader["DATA_TYPE"].ToString();

        // Create and populate the ColumnInfo object.
    }
}
```

Here's what happens:

1. `ExecuteReader()` executes the SQL query.
2. `reader.Read()` moves to the next returned row.
3. `reader["COLUMN_NAME"]` gets the current row's column name.
4. `reader["DATA_TYPE"]` gets its Oracle data type.
5. The loop repeats until every returned column has been processed.

For the first row, the values might be:

```
columnName = "PROJECT_ID"
oracleType = "NUMBER"
```

For the next row:

```
columnName = "PROJECT_NAME"
oracleType = "VARCHAR2"
```

And so on.

## Step E — Convert Oracle metadata into your frontend format

```
ColumnInfo column = new ColumnInfo();

column.columnName = ToCamelCase(columnName);
column.displayName = ToDisplayName(columnName);
column.dataType = oracleType;
column.filterType = GetFilterType(columnName, oracleType);

column.filterValues =
    GetFilterValues(columnName, column.filterType);

columns.Add(column);
```

These lines turn Oracle's raw column information into an object that your frontend can use.

For example:

- Oracle name: `PROJECT_ID`
- Frontend key: `projectId`
- Oracle type: `NUMBER`
- Filter type: `number`

The `GetFilterType()` function determines which filter control the frontend should show. `GetFilterValues()` retrieves available values for multi-select filters such as Category, Place, and Status.

`columns.Add(column)` then adds the finished object to the list.

After the loop, your list contains all the processed column objects.

## Step F — Build the table object

```
TableInfo table = new TableInfo();

table.tableName = "projectDetails";
table.displayName = "Project Details";
table.isDefault = true;
table.columns = columns;
```

Notice the difference:

- `PROJECT_DETAILS` is the actual Oracle table name.
- `projectDetails` is the frontend's table key.
- `Project Details` is the friendly label shown to the user.

The backend puts the columns inside the table object using `table.columns = columns`.

It then creates a list containing this single table:

```
List<TableInfo> tables = new List<TableInfo>();
tables.Add(table);
```

We use a list even though you currently have only one table so that your frontend response can support additional tables later.

# 4. Backend sends the response to the frontend

The final line is:

```
return new { tables = tables };
```

This creates an object containing the `tables` list. ASP.NET serializes the returned object into JSON for the AJAX response.

The response body will have this general structure:

```
{
  "d": {
    "tables": [
      {
        "tableName": "projectDetails",
        "displayName": "Project Details",
        "isDefault": true,
        "columns": [
          {
            "columnName": "projectId",
            "displayName": "PROJECT ID",
            "dataType": "NUMBER",
            "filterType": "number",
            "filterValues": []
          },
          {
            "columnName": "projectName",
            "displayName": "PROJECT NAME",
            "dataType": "VARCHAR2",
            "filterType": "text",
            "filterValues": []
          }
        ]
      }
    ]
  }
}
```

This is an illustrative excerpt; the real response contains all the table columns and their actual metadata. The `d` wrapper is commonly used by ASP.NET Web Forms page methods.

# 5. Frontend receives the response

Now return to the JavaScript success function:

```
success: function (response) {
    var result = response.d;

    console.log(result.tables);
}
```

Why do we use `response.d`?

Because the Web Forms WebMethod response is usually wrapped inside a property named `d`.

The returned object is therefore accessed like this:

```
response
  └── d
      └── tables
          └── first table
              ├── tableName
              ├── displayName
              └── columns
```

You can access the first table and its columns like this:

```
var result = response.d;

var firstTable = result.tables[0];

console.log(firstTable.tableName);
console.log(firstTable.displayName);
console.log(firstTable.columns);
```

Expected console output:

```
projectDetails
Project Details
Array of column objects
```

## 6. How those columns appear in your Attributes panel

The response doesn't automatically update your page's HTML. Your JavaScript must read `result.tables[0].columns` and create the attribute checkboxes.

For example, this simple loop demonstrates the idea:

```
var columns = result.tables[0].columns;
var html = "";

for (var i = 0; i < columns.length; i++) {
    html += "<label>";
    html += "<input type='checkbox' value='";
    html += columns[i].columnName;
    html += "'>";
    html += columns[i].displayName;
    html += "</label>";
}

document.getElementById("fieldList").innerHTML = html;
```

This is only a basic example of rendering the fields; your existing design may require its original CSS classes and checkbox event handlers to be retained.

The important point is that the JavaScript is no longer required to hardcode the table's columns. It receives the metadata from C# and uses that data to construct the interface.

The key distinction: at this first step, the frontend asks the backend what the table contains. Later, when the user selects Project Name, Cost, or Place and clicks Generate Report, the frontend sends those selected column keys back to a different WebMethod, `GenerateReport()`. That second request is where the selected columns and filters determine the report SQL.
