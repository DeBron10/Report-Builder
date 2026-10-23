Let's walk through the complete `GenerateReport()` process using your single Oracle table, `PROJECT_DETAILS`, with sample data.

We'll follow the data from the moment the user selects multiple attributes in your report builder, through AJAX, into the C# WebMethod, into the SQL query, and back to the frontend to display the report.

# 1. The user selects multiple attributes

Suppose your Attributes panel contains these checkboxes:

## Attributes

3 selected

Project ID

Project Name

Start Date

Cost

Category

Place

Status

Selected attribute keys sent to the backend

[   "projectName",   "cost",   "place" ]

This is an interactive example. Try selecting or deselecting fields to see how the selection changes.

The important thing is that the frontend sends the column keys, not the labels displayed to the user.

For example:

- Project Name → `projectName`
- Cost → `cost`
- Place → `place`

If the user selects three attributes, the frontend stores three values in an array called `selectedColumns`.

```
var selectedColumns = [];

selectedColumns.push("projectName");
selectedColumns.push("cost");
selectedColumns.push("place");
```

The array represents the user's current selection. In your actual UI, you build it by reading the checked checkboxes.

# 2. Prepare sample Oracle data

Assume your `PROJECT_DETAILS` table contains the following records.

| PROJECT_ID | PROJECT_NAME        | COST  | PLACE  |
| ---------- | ------------------- | ----- | ------ |
| 101        | Website Development | 50000 | Pune   |
| 102        | Office Renovation   | 85000 | Mumbai |
| 103        | Mobile Application  | 70000 | Pune   |
| 104        | Road Repair         | 40000 | Nashik |
| 105        | Data Migration      | 60000 | Mumbai |

The actual database may also contain `START_DATE`, `CATEGORY`, and `STATUS`.

Now suppose the user selects:

- Project Name
- Cost
- Place

And adds a filter:

\- Cost is between `40000` and `70000`

The report should contain only those three columns and only the rows that match the filter.

# 3. Frontend prepares the request

Your JavaScript reads the selected checkboxes and active filter controls, then creates a request object.

```
var selectedColumns = [];

selectedColumns.push("projectName");
selectedColumns.push("cost");
selectedColumns.push("place");

var filters = [];

var costFilter = {
    column: "cost",
    operation: "between",
    value: "40000",
    value2: "70000",
    values: []
};

filters.push(costFilter);

var requestData = {
    selectedColumns: selectedColumns,
    filters: filters
};
```

Here is what each variable means:

| Variable          | Purpose                                       |
| ----------------- | --------------------------------------------- |
| `selectedColumns` | The columns the user wants in the report      |
| `filters`         | The conditions the user wants to apply        |
| `requestData`     | The complete data package sent to the backend |

Notice that `value` and `value2` are strings in this example. The backend should validate and convert them to the appropriate Oracle types.

## What does `JSON.stringify()` do?

```
JSON.stringify(requestData)
```

It converts the JavaScript object into JSON text so it can be sent in the HTTP request body.

The resulting payload looks like this:

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
      "value2": "70000",
      "values": []
    }
  ]
}
```

This is the request format. It is not the SQL query itself. The backend receives these values and uses them to construct and execute the query.

# 4. Send the request using AJAX

The Generate Report button calls your C# WebMethod.

```
function generateReport() {

    var selectedColumns = readSelectedFields();
    var filters = readFilterValues();

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
            console.log("Record count:", result.totalRecords);

            displayReport(result);
        },

        error: function (xhr, status, error) {
            console.log(xhr.responseText);
            alert("Unable to generate the report.");
        }
    });
}
```

This assumes your existing `readSelectedFields()` returns the checked column keys and `readFilterValues()` returns filters in the request format shown above.

When the user clicks Generate Report, JavaScript sends the request to:

`ReportBuilder.aspx/GenerateReport`

ASP.NET finds the static C# method marked with `[WebMethod]` and passes the JSON properties into its parameters.

# 5. The backend receives the selected columns

Your C# function starts like this:

```
[WebMethod]
public static ReportResult GenerateReport(
    List<string> selectedColumns,
    List<FilterRequest> filters)
{
    // Generate and return the report.
}
```

ASP.NET converts the JSON arrays into C# lists automatically.

For our example, the values are:

```
selectedColumns
    [0] = "projectName"
    [1] = "cost"
    [2] = "place"

filters
    [0].column = "cost"
    [0].operation = "between"
    [0].value = "40000"
    [0].value2 = "70000"
```

You don't need to manually parse the JSON in C#. ASP.NET does that for a correctly configured WebMethod request.

The filter class defines the shape of each filter:

```
public class FilterRequest
{
    public string column { get; set; }
    public string filterType { get; set; }
    public string operation { get; set; }
    public string value { get; set; }
    public string value2 { get; set; }
    public List<string> values { get; set; }
}
```

For a numeric range, `value` is the lower bound and `value2` is the upper bound. For a multi-select filter, `values` contains the selected options.

# 6. How C# converts selected attributes into SQL

The backend must not blindly put browser-provided column names into SQL. It first checks the requested column keys against the actual Oracle metadata.

For example:

```
Frontend key     Oracle column
--------------------------------
projectName      PROJECT_NAME
cost             COST
place            PLACE
```

The backend then constructs the `SELECT` part by looping through all selected columns.

```
List<string> selectParts = new List<string>();

for (int i = 0; i < selectedColumns.Count; i++)
{
    string selected = selectedColumns[i];

    if (!validColumns.ContainsKey(selected))
    {
        throw new Exception("Invalid selected column.");
    }

    string realName = validColumns[selected];

    selectParts.Add("\"" + realName + "\"");
}
```

Because `selectedColumns` contains three values, the loop runs three times.

The resulting `selectParts` list contains:

```
[0] = "PROJECT_NAME"
[1] = "COST"
[2] = "PLACE"
```

Next, C# builds the SQL:

```
string sql = "SELECT " +
    string.Join(", ", selectParts.ToArray()) +
    " FROM PROJECT_DETAILS WHERE 1 = 1";
```

The query becomes:

```
SELECT "PROJECT_NAME", "COST", "PLACE"
FROM PROJECT_DETAILS
WHERE 1 = 1
```

Why use `WHERE 1 = 1`? It gives us a simple starting condition, so we can append filters with `AND` without checking whether the query already has a `WHERE` clause.

## 7. Add the filter to the SQL

For our Cost range, the backend adds a condition equivalent to:

```
AND "COST" BETWEEN :p0a AND :p0b
```

The complete query is conceptually:

```
SELECT "PROJECT_NAME", "COST", "PLACE"
FROM PROJECT_DETAILS
WHERE 1 = 1
  AND "COST" BETWEEN :p0a AND :p0b
```

The placeholders `:p0a` and `:p0b` are parameters. C# supplies `40000` and `70000` as parameter values instead of inserting them directly into the SQL text.

This is important for security and correct handling of values.

For the numeric range, use decimal values when assigning the parameters. For example, the parameter values should be converted to `decimal` after validating the input, rather than relying on implicit conversion from strings.

```
decimal minimumCost;
decimal maximumCost;

if (!decimal.TryParse(filter.value, out minimumCost) ||
    !decimal.TryParse(filter.value2, out maximumCost))
{
    throw new Exception("Enter valid cost values.");
}

command.Parameters.Add(
    "p" + i + "a", OracleDbType.Decimal).Value = minimumCost;

command.Parameters.Add(
    "p" + i + "b", OracleDbType.Decimal).Value = maximumCost;
```

This is the numeric `between` parameter logic to use in place of assigning strings to decimal parameters. Date filters should similarly be parsed into date values.

# 8. Oracle executes the query

Using the sample records from earlier, the database checks which rows have costs between 40000 and 70000, inclusive.

| Project Name        | Cost  | Place  | Matches? |
| ------------------- | ----- | ------ | -------- |
| Website Development | 50000 | Pune   | Yes      |
| Office Renovation   | 85000 | Mumbai | No       |
| Mobile Application  | 70000 | Pune   | Yes      |
| Road Repair         | 40000 | Nashik | Yes      |
| Data Migration      | 60000 | Mumbai | Yes      |

Oracle returns four rows, with only the three requested columns.

The backend reads them with `OracleDataReader`:

```
using (OracleDataReader reader = command.ExecuteReader())
{
    while (reader.Read())
    {
        // Read each returned row.
    }
}
```

Each time `reader.Read()` returns `true`, another matching database row is available. The backend copies its values into a C# dictionary, adds that dictionary to the result list, and repeats until all rows have been read.

# 9. Backend builds the response

Your backend uses this result class:

```
public class ReportResult
{
    public List<string> columns { get; set; }

    public List<Dictionary<string, object>> rows { get; set; }

    public int totalRecords { get; set; }
}
```

This is the structure of the response:

- `columns` — the columns selected by the user.
- `rows` — the matching records returned from Oracle.
- `totalRecords` — the number of records returned.

One small change to the earlier backend code: make sure the response uses the frontend column keys (`projectName`, `cost`, `place`) instead of Oracle's uppercase column names.

In the selected-columns loop, use:

```
result.columns.Add(selected);
```

instead of:

```
result.columns.Add(realName);
```

Then, when reading the result rows, use:

```
using (OracleDataReader reader = command.ExecuteReader())
{
    while (reader.Read())
    {
        Dictionary<string, object> row =
            new Dictionary<string, object>();

        for (int i = 0; i < result.columns.Count; i++)
        {
            object value = null;

            if (!reader.IsDBNull(i))
            {
                value = reader.GetValue(i);
            }

            row[result.columns[i]] = value;
        }

        result.rows.Add(row);
    }
}

result.totalRecords = result.rows.Count;

return result;
```

The SQL still uses the real Oracle column names. Only the response keys are changed to match your frontend.

## 10. What the backend sends back

For the sample data, the response will have this structure. ASP.NET wraps it in `d` when returning it from the Web Forms WebMethod.

```
{
  "d": {
    "columns": [
      "projectName",
      "cost",
      "place"
    ],
    "rows": [
      {
        "projectName": "Website Development",
        "cost": 50000,
        "place": "Pune"
      },
      {
        "projectName": "Mobile Application",
        "cost": 70000,
        "place": "Pune"
      },
      {
        "projectName": "Road Repair",
        "cost": 40000,
        "place": "Nashik"
      },
      {
        "projectName": "Data Migration",
        "cost": 60000,
        "place": "Mumbai"
      }
    ],
    "totalRecords": 4
  }
}
```

The actual row order depends on Oracle unless you add an `ORDER BY` clause. This JSON is an illustrative response based on our sample records.

Notice that the backend does not return every column in the database. It returns only the three requested columns and only the matching records.

# 11. Frontend receives the response and displays the report

Your AJAX success function gets the response:

```
success: function (response) {
    var result = response.d;

    console.log(result.columns);
    console.log(result.rows);
    console.log(result.totalRecords);

    displayReportFromBackend(result);
}
```

The frontend can then build the report table dynamically from the returned columns and rows.

Here is a simple example:

```
function displayReportFromBackend(result) {

    var html = "";

    html += "<thead>";
    html += "<tr>";

    for (var i = 0; i < result.columns.length; i++) {
        var column = result.columns[i];

        html += "<th>";
        html += escapeHTML(column);
        html += "</th>";
    }

    html += "</tr>";
    html += "</thead>";

    html += "<tbody>";

    for (var j = 0; j < result.rows.length; j++) {

        var row = result.rows[j];

        html += "<tr>";

        for (var k = 0; k < result.columns.length; k++) {

            var columnName = result.columns[k];
            var value = row[columnName];

            if (value == null) {
                value = "";
            }

            html += "<td>";
            html += escapeHTML(String(value));
            html += "</td>";
        }

        html += "</tr>";
    }

    html += "</tbody>";

    document.getElementById("reportTable").innerHTML = html;

    document.getElementById("recordCount").textContent =
        result.totalRecords;
}
```

This assumes `reportTable` is the HTML table element and `recordCount` is your existing record-count element. The example uses your existing `escapeHTML()` helper to safely display returned values.

Your actual UI can keep its existing table styling and report layout; the important part is that the table headers and cells now come from the backend response.

## The complete journey, in one example

1\. User selects attributes

`projectName`, `cost`, `place`

2\. JavaScript sends JSON

`selectedColumns` and `filters` are sent through AJAX.

3\. C# builds and executes SQL

Oracle returns the requested columns for rows matching the Cost range.

4\. C# returns JSON data

The response contains `columns`, `rows`, and `totalRecords`.

5\. JavaScript renders the report

The frontend creates the table headers and rows using the response, without hardcoding the selected attributes.

The key concept is this: the frontend tells the backend what the user wants; the backend queries Oracle and returns the data; the frontend renders whatever columns and rows come back.

That is what makes your report builder dynamic.
