I'll keep your HTML structure and CSS styling exactly as you provided, including the sidebar, panels, filter layout, report preview, and CSV export. I'll simplify the JavaScript using beginner-friendly functions, ordinary `for` loops, and string concatenation—without arrow functions, template literals, or methods like `.map()` and `.filter()`.

Since your next step is connecting Oracle, I'll keep the sample data for now but organize the script so you can replace the sample data with your ASP.NET WebMethods later.

One important change: your current code hides the builder after generating a report, which I'll preserve. I'll also keep the filter types you requested: numeric comparisons, date ranges, and multi-select Category, Place, and Status filters.

## 1. Keep your HTML and CSS unchanged

For a standalone `ReportBuilder.aspx` page, put this ASP.NET directive at the top of the file:

```
<%@ Page Language="C#" AutoEventWireup="true"
    CodeBehind="ReportBuilder.aspx.cs"
    Inherits="FileUpload.ReportBuilder" %>
```

Keep your existing HTML and CSS exactly as pasted. Replace only the existing `<script> ... </script>` block with the code below.

## 2. Simplified JavaScript — complete replacement

```

/* =========================================
   STEP 1: SAMPLE TABLE AND SAMPLE DATA
========================================= */

var tables = {
    projectDetails: {
        name: "Project Details",

        columns: [
            { name: "Project ID", key: "projectId", type: "number" },
            { name: "Project Name", key: "projectName", type: "text" },
            { name: "Start Date", key: "startDate", type: "date" },
            { name: "Cost", key: "cost", type: "number" },
            { name: "Category", key: "category", type: "multi" },
            { name: "Place", key: "place", type: "multi" },
            { name: "Status", key: "status", type: "multi" }
        ],

        rows: [
            {
                projectId: 101,
                projectName: "Website Development",
                startDate: "2026-01-10",
                cost: 50000,
                category: "IT",
                place: "Pune",
                status: "Completed"
            },
            {
                projectId: 102,
                projectName: "Office Renovation",
                startDate: "2026-02-15",
                cost: 85000,
                category: "Construction",
                place: "Mumbai",
                status: "In Progress"
            },
            {
                projectId: 103,
                projectName: "Mobile Application",
                startDate: "2026-03-05",
                cost: 70000,
                category: "IT",
                place: "Pune",
                status: "Pending"
            },
            {
                projectId: 104,
                projectName: "Road Repair",
                startDate: "2026-04-12",
                cost: 40000,
                category: "Construction",
                place: "Nashik",
                status: "Completed"
            },
            {
                projectId: 105,
                projectName: "Data Migration",
                startDate: "2026-05-20",
                cost: 60000,
                category: "IT",
                place: "Mumbai",
                status: "In Progress"
            },
            {
                projectId: 106,
                projectName: "School Renovation",
                startDate: "2026-06-10",
                cost: 95000,
                category: "Education",
                place: "Pune",
                status: "Pending"
            },
            {
                projectId: 107,
                projectName: "Water Supply Project",
                startDate: "2026-07-01",
                cost: 120000,
                category: "Infrastructure",
                place: "Nagpur",
                status: "In Progress"
            },
            {
                projectId: 108,
                projectName: "Hospital Upgrade",
                startDate: "2026-08-18",
                cost: 150000,
                category: "Healthcare",
                place: "Mumbai",
                status: "Completed"
            }
        ]
    }
};


/* =========================================
   STEP 2: STORE USER SELECTIONS
========================================= */

var selectedTables = ["projectDetails"];
var selectedColumns = [];
var filters = [];
var nextFilterId = 1;

var reportRows = [];
var reportColumns = [];


/* =========================================
   STEP 3: DISPLAY AVAILABLE FIELDS
========================================= */

function displayFields() {
    var fieldList = document.getElementById("fieldList");

    fieldList.innerHTML = "";

    for (var tableIndex = 0;
         tableIndex < selectedTables.length;
         tableIndex++) {

        var tableKey = selectedTables[tableIndex];
        var currentTable = tables[tableKey];

        for (var columnIndex = 0;
             columnIndex < currentTable.columns.length;
             columnIndex++) {

            var column = currentTable.columns[columnIndex];

            var fieldKey = tableKey + "." + column.key;

            var checkbox = document.createElement("input");

            checkbox.type = "checkbox";
            checkbox.id = "field_" + fieldKey;
            checkbox.value = fieldKey;
            checkbox.checked = isFieldSelected(fieldKey);

            checkbox.onchange = function () {
                readSelectedFields();
            };

            var label = document.createElement("label");

            label.htmlFor = checkbox.id;
            label.textContent = column.name;

            var fieldOption = document.createElement("div");

            fieldOption.className = "field-option";
            fieldOption.appendChild(checkbox);
            fieldOption.appendChild(label);

            fieldList.appendChild(fieldOption);
        }
    }

    updateFieldCount();
}


/* =========================================
   STEP 4: CHECK AND READ SELECTED FIELDS
========================================= */

function isFieldSelected(fieldKey) {
    for (var i = 0; i < selectedColumns.length; i++) {
        if (selectedColumns[i] === fieldKey) {
            return true;
        }
    }

    return false;
}


function readSelectedFields() {
    selectedColumns = [];

    var fieldList = document.getElementById("fieldList");
    var checkboxes = fieldList.getElementsByTagName("input");

    for (var i = 0; i < checkboxes.length; i++) {
        if (checkboxes[i].checked) {
            selectedColumns.push(checkboxes[i].value);
        }
    }

    updateFieldCount();
}


function updateFieldCount() {
    document.getElementById("fieldCount").textContent =
        selectedColumns.length + " fields selected";

    document.getElementById("selectedFieldCount").textContent =
        selectedColumns.length;
}


function clearFieldSelection() {
    selectedColumns = [];
    displayFields();
}


/* =========================================
   STEP 5: FIND A COLUMN'S DETAILS
========================================= */

function getColumnDetails(fieldKey) {
    if (fieldKey === "") {
        return null;
    }

    var parts = fieldKey.split(".");

    var tableKey = parts[0];
    var columnKey = parts[1];

    var currentTable = tables[tableKey];

    if (currentTable === undefined) {
        return null;
    }

    for (var i = 0; i < currentTable.columns.length; i++) {
        if (currentTable.columns[i].key === columnKey) {
            return currentTable.columns[i];
        }
    }

    return null;
}


/* =========================================
   STEP 6: GET DISTINCT FILTER VALUES
========================================= */

function getAvailableValues(fieldKey) {
    var values = [];

    var parts = fieldKey.split(".");
    var tableKey = parts[0];
    var columnKey = parts[1];

    var currentTable = tables[tableKey];

    for (var rowIndex = 0;
         rowIndex < currentTable.rows.length;
         rowIndex++) {

        var value = currentTable.rows[rowIndex][columnKey];
        var found = false;

        for (var valueIndex = 0;
             valueIndex < values.length;
             valueIndex++) {

            if (values[valueIndex] === value) {
                found = true;
                break;
            }
        }

        if (!found) {
            values.push(value);
        }
    }

    values.sort();

    return values;
}


/* =========================================
   STEP 7: ADD A FILTER
========================================= */

function addFilter() {
    filters.push({
        id: nextFilterId,
        columnKey: "",
        condition: "",
        value: "",
        value2: "",
        selectedValues: []
    });

    nextFilterId = nextFilterId + 1;

    displayFilters();
}


/* =========================================
   STEP 8: DISPLAY FILTERS
========================================= */

function displayFilters() {
    var filterList = document.getElementById("filterList");

    filterList.innerHTML = "";

    for (var i = 0; i < filters.length; i++) {
        var filter = filters[i];

        var filterRow = document.createElement("div");
        filterRow.className = "filter-row";

        /* Field dropdown */

        var fieldSelect = document.createElement("select");
        fieldSelect.id = "filterField_" + filter.id;

        var emptyOption = document.createElement("option");
        emptyOption.value = "";
        emptyOption.textContent = "Choose a field";

        fieldSelect.appendChild(emptyOption);

        for (var tableIndex = 0;
             tableIndex < selectedTables.length;
             tableIndex++) {

            var tableKey = selectedTables[tableIndex];
            var currentTable = tables[tableKey];

            for (var columnIndex = 0;
                 columnIndex < currentTable.columns.length;
                 columnIndex++) {

                var column = currentTable.columns[columnIndex];

                var option = document.createElement("option");

                option.value = tableKey + "." + column.key;
                option.textContent = column.name;

                fieldSelect.appendChild(option);
            }
        }

        fieldSelect.value = filter.columnKey;

        fieldSelect.onchange = function () {
            readFilterValues();

            var currentFilterId = parseInt(
                this.id.replace("filterField_", ""), 10
            );

            var currentFilter = getFilterById(currentFilterId);
            var columnDetails = getColumnDetails(this.value);

            currentFilter.columnKey = this.value;
            currentFilter.value = "";
            currentFilter.value2 = "";
            currentFilter.selectedValues = [];

            if (columnDetails === null) {
                currentFilter.condition = "";
            } else if (columnDetails.type === "number") {
                currentFilter.condition = ">";
            } else if (columnDetails.type === "date") {
                currentFilter.condition = "between";
            } else if (columnDetails.type === "multi") {
                currentFilter.condition = "in";
            } else {
                currentFilter.condition = "contains";
            }

            displayFilters();
        };

        filterRow.appendChild(fieldSelect);

        /* Condition dropdown */

        var columnDetails = getColumnDetails(filter.columnKey);
        var fieldType = "";

        if (columnDetails !== null) {
            fieldType = columnDetails.type;
        }

        var conditionSelect = document.createElement("select");
        conditionSelect.id = "filterCondition_" + filter.id;

        if (fieldType === "number") {
            addCondition(conditionSelect, ">", "Greater than");
            addCondition(conditionSelect, "<", "Less than");
            addCondition(conditionSelect, "=", "Equal to");
            addCondition(conditionSelect, "between", "Between");
        } else if (fieldType === "date") {
            addCondition(conditionSelect, "between", "Between");
        } else if (fieldType === "multi") {
            addCondition(conditionSelect, "in", "Is any of");
        } else if (fieldType === "text") {
            addCondition(conditionSelect, "contains", "Contains");
            addCondition(conditionSelect, "=", "Equal to");
        } else {
            addCondition(conditionSelect, "", "Choose a field first");
        }

        conditionSelect.value = filter.condition;

        conditionSelect.onchange = function () {
            readFilterValues();

            var currentFilterId = parseInt(
                this.id.replace("filterCondition_", ""), 10
            );

            var currentFilter = getFilterById(currentFilterId);

            currentFilter.condition = this.value;
            currentFilter.value = "";
            currentFilter.value2 = "";
            currentFilter.selectedValues = [];

            displayFilters();
        };

        filterRow.appendChild(conditionSelect);

        /* Value controls */

        var valueContainer = document.createElement("div");
        valueContainer.className = "filter-value";

        if (fieldType === "number" && filter.condition === "between") {
            addRangeInputs(
                valueContainer,
                filter,
                "number",
                "Minimum value",
                "Maximum value"
            );
        } else if (fieldType === "date") {
            addRangeInputs(
                valueContainer,
                filter,
                "date",
                "Start date",
                "End date"
            );
        } else if (fieldType === "multi") {
            addMultiSelect(valueContainer, filter);
        } else if (fieldType === "number") {
            addSingleInput(
                valueContainer,
                filter,
                "number",
                "Enter an amount"
            );
        } else if (fieldType === "text") {
            addSingleInput(
                valueContainer,
                filter,
                "text",
                "Enter text"
            );
        } else {
            var help = document.createElement("div");
            help.className = "value-help";
            help.textContent = "Choose a field to configure its filter.";
            valueContainer.appendChild(help);
        }

        filterRow.appendChild(valueContainer);

        /* Remove button */

        var removeButton = document.createElement("button");

        removeButton.className = "remove-button";
        removeButton.textContent = "X";
        removeButton.setAttribute("data-filter-id", filter.id);

        removeButton.onclick = function () {
            var filterId = parseInt(
                this.getAttribute("data-filter-id"), 10
            );

            removeFilter(filterId);
        };

        filterRow.appendChild(removeButton);
        filterList.appendChild(filterRow);
    }

    updateSummary();
}


/* =========================================
   STEP 9: FILTER CONTROL HELPERS
========================================= */

function getFilterById(filterId) {
    for (var i = 0; i < filters.length; i++) {
        if (filters[i].id === filterId) {
            return filters[i];
        }
    }

    return null;
}


function addCondition(selectElement, value, label) {
    var option = document.createElement("option");

    option.value = value;
    option.textContent = label;

    selectElement.appendChild(option);
}


function addSingleInput(container, filter, inputType, placeholderText) {
    var input = document.createElement("input");

    input.type = inputType;
    input.id = "filterValue_" + filter.id;
    input.placeholder = placeholderText;
    input.value = filter.value;

    input.oninput = function () {
        readFilterValues();
    };

    container.appendChild(input);
}


function addRangeInputs(
    container,
    filter,
    inputType,
    firstPlaceholder,
    secondPlaceholder
) {
    var rangeContainer = document.createElement("div");
    rangeContainer.className = "range-inputs";

    var firstInput = document.createElement("input");

    firstInput.type = inputType;
    firstInput.id = "filterValue_" + filter.id;
    firstInput.placeholder = firstPlaceholder;
    firstInput.value = filter.value;

    firstInput.onchange = function () {
        readFilterValues();
    };

    firstInput.oninput = function () {
        readFilterValues();
    };

    var secondInput = document.createElement("input");

    secondInput.type = inputType;
    secondInput.id = "filterValue2_" + filter.id;
    secondInput.placeholder = secondPlaceholder;
    secondInput.value = filter.value2;

    secondInput.onchange = function () {
        readFilterValues();
    };

    secondInput.oninput = function () {
        readFilterValues();
    };

    rangeContainer.appendChild(firstInput);
    rangeContainer.appendChild(secondInput);

    container.appendChild(rangeContainer);
}


function addMultiSelect(container, filter) {
    var select = document.createElement("select");

    select.id = "filterValues_" + filter.id;
    select.multiple = true;

    var values = getAvailableValues(filter.columnKey);

    for (var i = 0; i < values.length; i++) {
        var option = document.createElement("option");

        option.value = values[i];
        option.textContent = values[i];
        option.selected = isValueSelected(
            filter.selectedValues,
            values[i]
        );

        select.appendChild(option);
    }

    select.onchange = function () {
        readFilterValues();
    };

    container.appendChild(select);

    var help = document.createElement("div");

    help.className = "value-help";
    help.textContent = "Hold Ctrl on Windows to select multiple values.";

    container.appendChild(help);
}


function isValueSelected(selectedValues, value) {
    for (var i = 0; i < selectedValues.length; i++) {
        if (selectedValues[i] === value) {
            return true;
        }
    }

    return false;
}


/* =========================================
   STEP 10: READ FILTER INPUTS
========================================= */

function readFilterValues() {
    for (var i = 0; i < filters.length; i++) {
        var filter = filters[i];

        var fieldSelect = document.getElementById(
            "filterField_" + filter.id
        );

        var conditionSelect = document.getElementById(
            "filterCondition_" + filter.id
        );

        if (fieldSelect !== null) {
            filter.columnKey = fieldSelect.value;
        }

        if (conditionSelect !== null) {
            filter.condition = conditionSelect.value;
        }

        var firstInput = document.getElementById(
            "filterValue_" + filter.id
        );

        if (firstInput !== null) {
            filter.value = firstInput.value;
        } else {
            filter.value = "";
        }

        var secondInput = document.getElementById(
            "filterValue2_" + filter.id
        );

        if (secondInput !== null) {
            filter.value2 = secondInput.value;
        } else {
            filter.value2 = "";
        }

        var multiSelect = document.getElementById(
            "filterValues_" + filter.id
        );

        filter.selectedValues = [];

        if (multiSelect !== null) {
            for (var optionIndex = 0;
                 optionIndex < multiSelect.options.length;
                 optionIndex++) {

                if (multiSelect.options[optionIndex].selected) {
                    filter.selectedValues.push(
                        multiSelect.options[optionIndex].value
                    );
                }
            }
        }
    }

    updateSummary();
}


/* =========================================
   STEP 11: REMOVE FILTER AND UPDATE SUMMARY
========================================= */

function removeFilter(filterId) {
    var remainingFilters = [];

    for (var i = 0; i < filters.length; i++) {
        if (filters[i].id !== filterId) {
            remainingFilters.push(filters[i]);
        }
    }

    filters = remainingFilters;

    displayFilters();
}


function updateSummary() {
    document.getElementById("selectedFieldCount").textContent =
        selectedColumns.length;

    var activeCount = 0;

    for (var i = 0; i < filters.length; i++) {
        var filter = filters[i];

        if (filter.columnKey === "") {
            continue;
        }

        if (filter.condition === "in") {
            if (filter.selectedValues.length > 0) {
                activeCount++;
            }
        } else if (filter.condition === "between") {
            if (filter.value !== "" && filter.value2 !== "") {
                activeCount++;
            }
        } else {
            if (filter.value !== "") {
                activeCount++;
            }
        }
    }

    document.getElementById("activeFilterCount").textContent =
        activeCount;
}


/* =========================================
   STEP 12: CHECK WHETHER A RECORD MATCHES
========================================= */

function recordMatchesFilter(record, filter) {
    var parts = filter.columnKey.split(".");
    var columnKey = parts[1];

    var columnDetails = getColumnDetails(filter.columnKey);
    var recordValue = record[columnKey];

    if (columnDetails === null) {
        return true;
    }

    /* Multi-select filter */

    if (filter.condition === "in") {
        if (filter.selectedValues.length === 0) {
            return true;
        }

        for (var i = 0; i < filter.selectedValues.length; i++) {
            if (String(recordValue) === String(filter.selectedValues[i])) {
                return true;
            }
        }

        return false;
    }

    /* Date range */

    if (columnDetails.type === "date") {
        if (filter.value === "" || filter.value2 === "") {
            return true;
        }

        return recordValue >= filter.value &&
            recordValue <= filter.value2;
    }

    /* Number conditions */

    if (columnDetails.type === "number") {
        if (filter.value === "") {
            return true;
        }

        var numberValue = Number(recordValue);
        var firstNumber = Number(filter.value);

        if (filter.condition === ">") {
            return numberValue > firstNumber;
        }

        if (filter.condition === "<") {
            return numberValue < firstNumber;
        }

        if (filter.condition === "=") {
            return numberValue === firstNumber;
        }

        if (filter.condition === "between") {
            if (filter.value2 === "") {
                return true;
            }

            var secondNumber = Number(filter.value2);

            return numberValue >= firstNumber &&
                numberValue <= secondNumber;
        }
    }

    /* Text conditions */

    if (columnDetails.type === "text") {
        if (filter.value === "") {
            return true;
        }

        var textValue = String(recordValue).toLowerCase();
        var searchValue = String(filter.value).toLowerCase();

        if (filter.condition === "contains") {
            return textValue.indexOf(searchValue) !== -1;
        }

        if (filter.condition === "=") {
            return textValue === searchValue;
        }
    }

    return true;
}


function recordMatchesAllFilters(tableKey, record) {
    for (var i = 0; i < filters.length; i++) {
        var filter = filters[i];

        if (filter.columnKey === "") {
            continue;
        }

        var parts = filter.columnKey.split(".");

        if (parts[0] !== tableKey) {
            continue;
        }

        /* Ignore incomplete filters */

        if (filter.condition === "in") {
            if (filter.selectedValues.length === 0) {
                continue;
            }
        } else if (filter.condition === "between") {
            if (filter.value === "" || filter.value2 === "") {
                continue;
            }
        } else if (filter.value === "") {
            continue;
        }

        if (!recordMatchesFilter(record, filter)) {
            return false;
        }
    }

    return true;
}


/* =========================================
   STEP 13: GENERATE REPORT
========================================= */

function generateReport() {
    readSelectedFields();
    readFilterValues();

    if (selectedColumns.length === 0) {
        alert("Please select at least one data field.");
        return;
    }

    /* Validate ranges before generating the report */

    for (var i = 0; i < filters.length; i++) {
        var filter = filters[i];

        var columnDetails = getColumnDetails(filter.columnKey);

        if (columnDetails === null ||
            filter.condition !== "between" ||
            filter.value === "" ||
            filter.value2 === "") {

            continue;
        }

        if (columnDetails.type === "number" &&
            Number(filter.value) > Number(filter.value2)) {

            alert("The minimum value cannot exceed the maximum value.");
            return;
        }

        if (columnDetails.type === "date" &&
            filter.value > filter.value2) {

            alert("The start date cannot be later than the end date.");
            return;
        }
    }

    reportRows = [];
    reportColumns = [];

    for (var columnIndex = 0;
         columnIndex < selectedColumns.length;
         columnIndex++) {

        reportColumns.push(selectedColumns[columnIndex]);
    }

    for (var tableIndex = 0;
         tableIndex < selectedTables.length;
         tableIndex++) {

        var tableKey = selectedTables[tableIndex];
        var currentTable = tables[tableKey];

        for (var rowIndex = 0;
             rowIndex < currentTable.rows.length;
             rowIndex++) {

            var record = currentTable.rows[rowIndex];

            if (recordMatchesAllFilters(tableKey, record)) {
                reportRows.push({
                    tableKey: tableKey,
                    data: record
                });
            }
        }
    }

    displayReport();

    document.getElementById("reportDescription").textContent =
        "Report generated using your selected fields and filters.";

    document.getElementById("builder").classList.add("hidden");
}


/* =========================================
   STEP 14: DISPLAY REPORT TABLE
========================================= */

function displayReport() {
    var reportTable = document.getElementById("reportTable");

    var headingHTML = "<thead><tr>";
    var bodyHTML = "<tbody>";

    for (var columnIndex = 0;
         columnIndex < reportColumns.length;
         columnIndex++) {

        var columnDetails = getColumnDetails(reportColumns[columnIndex]);

        headingHTML = headingHTML + "<th>" +
            escapeHTML(columnDetails.name) + "</th>";
    }

    headingHTML = headingHTML + "</tr></thead>";

    for (var rowIndex = 0;
         rowIndex < reportRows.length;
         rowIndex++) {

        var reportRecord = reportRows[rowIndex];

        bodyHTML = bodyHTML + "<tr>";

        for (columnIndex = 0;
             columnIndex < reportColumns.length;
             columnIndex++) {

            var fieldKey = reportColumns[columnIndex];
            var parts = fieldKey.split(".");

            var columnTableKey = parts[0];
            var columnKey = parts[1];

            var cellValue = "";

            if (columnTableKey === reportRecord.tableKey) {
                cellValue = reportRecord.data[columnKey];
            }

            if (cellValue === null || cellValue === undefined) {
                cellValue = "";
            }

            if (columnKey === "status" && cellValue !== "") {
                var statusClass = "status-pending";

                if (cellValue === "Completed") {
                    statusClass = "status-completed";
                } else if (cellValue === "In Progress") {
                    statusClass = "status-progress";
                }

                bodyHTML = bodyHTML +
                    "<td><span class='status-label " +
                    statusClass + "'>" +
                    escapeHTML(String(cellValue)) +
                    "</span></td>";
            } else {
                bodyHTML = bodyHTML + "<td>" +
                    escapeHTML(String(cellValue)) + "</td>";
            }
        }

        bodyHTML = bodyHTML + "</tr>";
    }

    if (reportRows.length === 0) {
        bodyHTML = bodyHTML +
            "<tr><td colspan='" + reportColumns.length + "'>" +
            "<div class='empty-state'>" +
            "<div class='empty-title'>No records found</div>" +
            "Try changing your filter conditions or selected values." +
            "</div></td></tr>";
    }

    bodyHTML = bodyHTML + "</tbody>";

    reportTable.innerHTML = headingHTML + bodyHTML;

    document.getElementById("recordCount").textContent =
        reportRows.length;
}


/* =========================================
   STEP 15: ESCAPE DISPLAY VALUES
========================================= */

function escapeHTML(value) {
    var result = String(value);

    result = result.replace(/&/g, "&amp;");
    result = result.replace(/</g, "&lt;");
    result = result.replace(/>/g, "&gt;");
    result = result.replace(/"/g, "&quot;");
    result = result.replace(/'/g, "&#39;");

    return result;
}


/* =========================================
   STEP 16: SHOW OR HIDE BUILDER
========================================= */

function toggleBuilder() {
    var builder = document.getElementById("builder");

    builder.classList.toggle("hidden");
}


/* =========================================
   STEP 17: RESET THE BUILDER
========================================= */

function resetBuilder() {
    selectedTables = ["projectDetails"];
    selectedColumns = [];
    filters = [];
    nextFilterId = 1;
    reportRows = [];
    reportColumns = [];

    document.getElementById("builder").classList.remove("hidden");

    displayFields();
    displayFilters();

    document.getElementById("recordCount").textContent = "0";

    document.getElementById("reportDescription").textContent =
        "Your generated report will appear here.";

    document.getElementById("reportTable").innerHTML =
        "<thead><tr><th>Report Preview</th></tr></thead>" +
        "<tbody><tr><td><div class='empty-state'>" +
        "<div class='empty-title'>Ready to build your report?</div>" +
        "Select data fields and add filters, then click Generate Report." +
        "</div></td></tr></tbody>";

    updateSummary();
}


/* =========================================
   STEP 18: EXPORT REPORT TO CSV
========================================= */

function exportCSV() {
    if (reportColumns.length === 0) {
        alert("Generate a report before exporting.");
        return;
    }

    var csvContent = "";

    /* Column headings */

    for (var columnIndex = 0;
         columnIndex < reportColumns.length;
         columnIndex++) {

        var columnDetails = getColumnDetails(reportColumns[columnIndex]);

        var heading = columnDetails.name;
        heading = heading.replace(/"/g, '""');

        csvContent = csvContent + '"' + heading + '"';

        if (columnIndex < reportColumns.length - 1) {
            csvContent = csvContent + ",";
        }
    }

    csvContent = csvContent + "\r\n";

    /* Report data */

    for (var rowIndex = 0;
         rowIndex < reportRows.length;
         rowIndex++) {

        var reportRecord = reportRows[rowIndex];

        for (columnIndex = 0;
             columnIndex < reportColumns.length;
             columnIndex++) {

            var fieldKey = reportColumns[columnIndex];
            var parts = fieldKey.split(".");

            var columnTableKey = parts[0];
            var columnKey = parts[1];

            var cellValue = "";

            if (columnTableKey === reportRecord.tableKey) {
                cellValue = reportRecord.data[columnKey];
            }

            if (cellValue === null || cellValue === undefined) {
                cellValue = "";
            }

            cellValue = String(cellValue);
            cellValue = cellValue.replace(/"/g, '""');

            csvContent = csvContent + '"' + cellValue + '"';

            if (columnIndex < reportColumns.length - 1) {
                csvContent = csvContent + ",";
            }
        }

        csvContent = csvContent + "\r\n";
    }

    var csvFile = new Blob(
        [csvContent],
        { type: "text/csv;charset=utf-8;" }
    );

    var downloadUrl = URL.createObjectURL(csvFile);
    var downloadLink = document.createElement("a");

    downloadLink.href = downloadUrl;
    downloadLink.download = "custom-report.csv";

    document.body.appendChild(downloadLink);

    downloadLink.click();

    document.body.removeChild(downloadLink);

    URL.revokeObjectURL(downloadUrl);
}


/* =========================================
   STEP 19: INITIALIZE THE PAGE
========================================= */

displayFields();
displayFilters();
updateSummary();

```

## 3. Important things to know

- Design: the CSS and HTML are unchanged; the script uses the same IDs and class names.
- Syntax: the logic uses regular functions, `for` loops, `if/else`, and string concatenation.
- Filters: changing the selected field now resets the condition to a valid default for that field.
- Data: the script still uses sample data in the browser. It does not query Oracle yet.
- Backend integration: when you're ready, we'll replace the sample `tables` object and local filtering in `generateReport()` with AJAX calls to `LoadReportMetadata()` and `GenerateReport()` in `ReportBuilder.aspx.cs`. The UI can remain the same.

One detail: this sample currently has only `Project Details`. If you later add `Project Team` and other tables, the backend will need defined relationships and SQL joins before it can generate a combined report.
