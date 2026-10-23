Since your report builder currently has only one table, `PROJECT_DETAILS`, we can keep the Oracle backend simple. Your existing HTML/CSS design can stay exactly the same; the backend will supply the table's column metadata, load report data, and apply the selected columns and filters.

We'll use ASP.NET Web Forms, C#, Oracle Database, and jQuery AJAX.

## 1. Backend flow

Oracle database

`PROJECT_DETAILS` table and its columns

C# WebMethods

Load metadata and generate the report

JavaScript / AJAX

Send selected columns and filters, then display results

For now, we'll create two WebMethods:

- `LoadReportMetadata()` — returns the columns of `PROJECT_DETAILS`.
- `GenerateReport()` — receives the selected columns and filters, builds a safe SQL query, and returns the report rows.

## 2. Configure the Oracle connection

In your `Web.config`, add or update the connection string. Replace the example server, service name, username, and password with your Oracle database details.

```
<configuration>
  <connectionStrings>
    <add name="OracleDb"
         connectionString="User Id=YOUR_USERNAME;Password=YOUR_PASSWORD;Data Source=YOUR_HOST:1521/YOUR_SERVICE_NAME;"
         providerName="Oracle.ManagedDataAccess.Client" />
  </connectionStrings>
</configuration>
```

Install or reference the Oracle.ManagedDataAccess NuGet package in your project if it is not already installed.

## 3. Create `ReportBuilder.aspx.cs`

Keep your existing `ReportBuilder.aspx` UI. Add the following backend code to its code-behind file. It assumes your project namespace is `FileUpload` and your Oracle table is `PROJECT_DETAILS`.

This example uses the Oracle data-access library and parameterized filter values. Only columns verified against the table's metadata are included in the generated SQL.

```

using System;
using System.Collections.Generic;
using System.Configuration;
using System.Data;
using System.Web.Services;
using Oracle.ManagedDataAccess.Client;

namespace FileUpload
{
    public partial class ReportBuilder : System.Web.UI.Page
    {
        public class ColumnInfo
        {
            public string columnName { get; set; }
            public string displayName { get; set; }
            public string dataType { get; set; }
            public string filterType { get; set; }
            public List<string> filterValues { get; set; }
        }

        public class TableInfo
        {
            public string tableName { get; set; }
            public string displayName { get; set; }
            public bool isDefault { get; set; }
            public List<ColumnInfo> columns { get; set; }
        }

        public class FilterRequest
        {
            public string column { get; set; }
            public string filterType { get; set; }
            public string operation { get; set; }
            public string value { get; set; }
            public string value2 { get; set; }
            public List<string> values { get; set; }
        }

        public class ReportResult
        {
            public List<string> columns { get; set; }
            public List<Dictionary<string, object>> rows { get; set; }
            public int totalRecords { get; set; }
        }

        private static string GetConnectionString()
        {
            return ConfigurationManager
                .ConnectionStrings["OracleDb"].ConnectionString;
        }

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
                        string columnName = reader["COLUMN_NAME"].ToString();
                        string oracleType = reader["DATA_TYPE"].ToString();

                        ColumnInfo column = new ColumnInfo();

                        column.columnName = ToCamelCase(columnName);
                        column.displayName = ToDisplayName(columnName);
                        column.dataType = oracleType;
                        column.filterType = GetFilterType(columnName, oracleType);
                        column.filterValues = GetFilterValues(
                            columnName, column.filterType);

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

        private static string ToCamelCase(string name)
        {
            string result = name.ToLower();

            if (result.Length > 0)
            {
                result = result.Substring(0, 1).ToLower()
                    + result.Substring(1);
            }

            return result;
        }

        private static string ToDisplayName(string name)
        {
            string result = "";

            for (int i = 0; i < name.Length; i++)
            {
                char current = name[i];

                if (i > 0 && char.IsUpper(current))
                {
                    result += " ";
                }

                result += current;
            }

            return result.Replace("_", " ");
        }

        private static string GetFilterType(
            string columnName, string oracleType)
        {
            string name = columnName.ToUpper();
            string type = oracleType.ToUpper();

            if (type == "DATE" || type.StartsWith("TIMESTAMP"))
            {
                return "date";
            }

            if (type == "NUMBER" || type == "FLOAT"
                || type == "BINARY_FLOAT" || type == "BINARY_DOUBLE")
            {
                if (name.Contains("CATEGORY")
                    || name.Contains("PLACE")
                    || name.Contains("STATUS"))
                {
                    return "multi";
                }

                return "number";
            }

            if (name.Contains("CATEGORY")
                || name.Contains("PLACE")
                || name.Contains("STATUS"))
            {
                return "multi";
            }

            return "text";
        }

        private static List<string> GetFilterValues(
            string columnName, string filterType)
        {
            List<string> values = new List<string>();

            if (filterType != "multi")
            {
                return values;
            }

            string safeColumn = columnName.ToUpper();

            using (OracleConnection connection =
                new OracleConnection(GetConnectionString()))
            {
                connection.Open();

                // Column names are taken from Oracle metadata,
                // not from user input.
                string sql = "SELECT DISTINCT \"" + safeColumn + "\" " +
                             "FROM PROJECT_DETAILS " +
                             "WHERE \"" + safeColumn + "\" IS NOT NULL " +
                             "ORDER BY \"" + safeColumn + "\"";

                using (OracleCommand command =
                    new OracleCommand(sql, connection))
                using (OracleDataReader reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        values.Add(reader[0].ToString());
                    }
                }
            }

            return values;
        }

        [WebMethod]
        public static ReportResult GenerateReport(
            List<string> selectedColumns,
            List<FilterRequest> filters)
        {
            if (selectedColumns == null || selectedColumns.Count == 0)
            {
                throw new Exception("Select at least one attribute.");
            }

            if (filters == null)
            {
                filters = new List<FilterRequest>();
            }

            ReportResult result = new ReportResult();
            result.columns = new List<string>();
            result.rows = new List<Dictionary<string, object>>();

            using (OracleConnection connection =
                new OracleConnection(GetConnectionString()))
            {
                connection.Open();

                // Read the real column names from Oracle.
                Dictionary<string, string> validColumns =
                    new Dictionary<string, string>(
                        StringComparer.OrdinalIgnoreCase);

                string metadataSql =
                    "SELECT COLUMN_NAME FROM USER_TAB_COLUMNS " +
                    "WHERE TABLE_NAME = 'PROJECT_DETAILS'";

                using (OracleCommand command =
                    new OracleCommand(metadataSql, connection))
                using (OracleDataReader reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        string realName = reader["COLUMN_NAME"].ToString();
                        validColumns[ToCamelCase(realName)] = realName;
                        validColumns[realName] = realName;
                    }
                }

                List<string> selectParts = new List<string>();

                for (int i = 0; i < selectedColumns.Count; i++)
                {
                    string selected = selectedColumns[i];

                    if (!validColumns.ContainsKey(selected))
                    {
                        throw new Exception(
                            "Invalid selected column: " + selected);
                    }

                    string realName = validColumns[selected];

                    selectParts.Add("\"" + realName + "\"");
                    result.columns.Add(realName);
                }

                string sql = "SELECT " +
                    string.Join(", ", selectParts.ToArray()) +
                    " FROM PROJECT_DETAILS WHERE 1 = 1";

                using (OracleCommand command =
                    new OracleCommand("", connection))
                {
                    for (int i = 0; i < filters.Count; i++)
                    {
                        FilterRequest filter = filters[i];

                        if (filter == null || string.IsNullOrWhiteSpace(
                            filter.column))
                        {
                            continue;
                        }

                        if (!validColumns.ContainsKey(filter.column))
                        {
                            throw new Exception("Invalid filter column.");
                        }

                        string realColumn = validColumns[filter.column];
                        string quotedColumn = "\"" + realColumn + "\"";
                        string operation = filter.operation;

                        if (operation == ">")
                        {
                            sql += " AND " + quotedColumn + " > :p" + i;
                            command.Parameters.Add(
                                "p" + i, GetOracleType(realColumn,
                                connection), 0).Value = filter.value;
                        }
                        else if (operation == "<")
                        {
                            sql += " AND " + quotedColumn + " < :p" + i;
                            command.Parameters.Add(
                                "p" + i, GetOracleType(realColumn,
                                connection), 0).Value = filter.value;
                        }
                        else if (operation == "=")
                        {
                            sql += " AND " + quotedColumn + " = :p" + i;
                            command.Parameters.Add(
                                "p" + i, GetOracleType(realColumn,
                                connection), 0).Value = filter.value;
                        }
                        else if (operation == "between")
                        {
                            sql += " AND " + quotedColumn +
                                   " BETWEEN :p" + i + "a AND :p" + i + "b";

                            OracleDbType type = GetOracleType(
                                realColumn, connection);

                            command.Parameters.Add(
                                "p" + i + "a", type).Value = filter.value;
                            command.Parameters.Add(
                                "p" + i + "b", type).Value = filter.value2;
                        }
                        else if (operation == "contains")
                        {
                            sql += " AND LOWER(" + quotedColumn +
                                   ") LIKE :p" + i;

                            command.Parameters.Add(
                                "p" + i, OracleDbType.Varchar2).Value =
                                "%" + filter.value.ToLower() + "%";
                        }
                        else if (operation == "in")
                        {
                            if (filter.values == null ||
                                filter.values.Count == 0)
                            {
                                continue;
                            }

                            List<string> inValues = new List<string>();

                            for (int j = 0; j < filter.values.Count; j++)
                            {
                                string parameterName =
                                    "p" + i + "_" + j;

                                inValues.Add(":" + parameterName);

                                command.Parameters.Add(
                                    parameterName, OracleDbType.Varchar2)
                                    .Value = filter.values[j];
                            }

                            sql += " AND " + quotedColumn + " IN (" +
                                string.Join(", ", inValues.ToArray()) + ")";
                        }
                        else
                        {
                            throw new Exception("Unsupported filter operator.");
                        }
                    }

                    command.CommandText = sql;

                    using (OracleDataReader reader = command.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            Dictionary<string, object> row =
                                new Dictionary<string, object>();

                            for (int i = 0; i < result.columns.Count; i++)
                            {
                                object value = reader.IsDBNull(i)
                                    ? null : reader.GetValue(i);

                                row[result.columns[i]] = value;
                            }

                            result.rows.Add(row);
                        }
                    }
                }
            }

            result.totalRecords = result.rows.Count;
            return result;
        }

        private static OracleDbType GetOracleType(
            string columnName, OracleConnection connection)
        {
            string sql =
                "SELECT DATA_TYPE FROM USER_TAB_COLUMNS " +
                "WHERE TABLE_NAME = 'PROJECT_DETAILS' " +
                "AND COLUMN_NAME = :columnName";

            using (OracleCommand command =
                new OracleCommand(sql, connection))
            {
                command.Parameters.Add(
                    "columnName", OracleDbType.Varchar2).Value =
                    columnName.ToUpper();

                string type = Convert.ToString(command.ExecuteScalar());

                if (type == "DATE" || type.StartsWith("TIMESTAMP"))
                {
                    return OracleDbType.Date;
                }

                if (type == "NUMBER" || type == "FLOAT")
                {
                    return OracleDbType.Decimal;
                }

                return OracleDbType.Varchar2;
            }
        }
    }
}

```

## 4. Important correction: convert Oracle column names to frontend keys

Oracle commonly stores unquoted column names in uppercase. For example, `PROJECT_ID` should become `projectId` to match your frontend.

Replace the `ToCamelCase()` method in the code above with this version:

```

private static string ToCamelCase(string name)
{
    string result = "";
    bool makeNextLetterUpper = false;

    for (int i = 0; i < name.Length; i++)
    {
        char current = name[i];

        if (current == '_')
        {
            makeNextLetterUpper = true;
        }
        else
        {
            if (result.Length == 0)
            {
                result += char.ToLower(current);
            }
            else if (makeNextLetterUpper)
            {
                result += char.ToUpper(current);
                makeNextLetterUpper = false;
            }
            else
            {
                result += char.ToLower(current);
            }
        }
    }

    return result;
}

```

For your current fields, this gives keys such as `projectId`, `projectName`, `startDate`, `cost`, `category`, `place`, and `status`.

## 5. How your frontend calls the backend

Your AJAX request must send `selectedColumns` as an array of column keys and `filters` in the format expected by the C# `FilterRequest` class.

For example, selecting Project Name, Cost, and Place, then filtering Cost between 40000 and 90000 and Place by Pune, sends this JSON:

```
{
  "selectedColumns": [
    "projectName",
    "cost",
    "place"
  ],
  "filters": [
    {
      "column": "cost",
      "operation": "between",
      "value": "40000",
      "value2": "90000",
      "values": []
    },
    {
      "column": "place",
      "operation": "in",
      "value": "",
      "value2": "",
      "values": ["Pune"]
    }
  ]
}
```

Use this AJAX pattern in your existing JavaScript. It assumes `selectedColumns` and `filters` are arrays already prepared in the format above.

```

var requestData = {
    selectedColumns: selectedColumns,
    filters: filters
};

$.ajax({
    type: "POST",
    url: "ReportBuilder.aspx/GenerateReport",
    data: JSON.stringify(requestData),
    contentType: "application/json; charset=utf-8",
    dataType: "json",

    success: function (response) {
        var result = response.d;

        console.log("Columns:", result.columns);
        console.log("Rows:", result.rows);
        console.log("Total records:", result.totalRecords);

        // Pass result to your report display function.
        // Match the returned columns/rows format to your existing UI.
    },

    error: function (xhr, status, error) {
        console.log("Request failed:", error);
        console.log(xhr.responseText);
        alert("Unable to generate the report.");
    }
});

```

For metadata, call:

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

## 6. What to verify before running

- The actual Oracle table is named `PROJECT_DETAILS` and belongs to the schema used by the connection.
- The `OracleDb` connection string works and the project references `Oracle.ManagedDataAccess`.
- Your `ReportBuilder.aspx` page directive points to `FileUpload.ReportBuilder`.
- Your frontend uses the same request property names as the backend.
- Your report-rendering JavaScript handles `result.columns` and `result.rows`. The sample backend returns column names as strings and each row as a dictionary keyed by Oracle column names, so your display function may need a small adjustment.
- Date and numeric filters should be validated and converted to their proper types before executing the query. The backend must never accept arbitrary SQL or column names from the browser.

One important note: this version assumes all selected columns come from the single `PROJECT_DETAILS` table. It does not yet support joins, multiple tables, sorting, pagination, or export on the server. Those can be added later without changing the overall design.
