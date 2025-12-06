# Building a Dynamic SQL Dashboard with n8n: A Complete Tutorial

This tutorial will guide you through creating an interactive, data-driven dashboard using n8n that queries a SQL Server database and displays beautiful visualizations using amCharts. The workflow serves a responsive HTML dashboard through a webhook endpoint.

## What You'll Build

An automated dashboard that:
- Queries customer transactions, stock locations, and supplier data from SQL Server
- Aggregates and formats the data
- Generates interactive charts (column chart, donut chart, and force-directed network)
- Serves everything as a single HTML page with a custom background image

## Prerequisites

- n8n instance (self-hosted or cloud)
- SQL Server database with appropriate permissions
- Basic understanding of SQL queries
- (Optional) Background image for your dashboard

---

## Step-by-Step Guide

### Step 1: Create the Webhook Trigger

1. **Add a Webhook node** to your canvas
2. Configure it:
   - **HTTP Method**: GET
   - **Path**: Generate or use a custom path (e.g., `f82f2e67-8c97-4d70-8274-fbdadb1be1ed`)
   - **Response Mode**: "Respond to Webhook" (we'll respond later in the workflow)
   - **Options** → Enable "Raw Body"
   - **Response Headers** → Add header:
     - Name: `content-type`
     - Value: `text/html`

This creates an endpoint that will serve your dashboard when accessed.

### Step 2: Set Up Database Connections

You'll create three parallel SQL queries. For each one:

#### 2a. Customer Transactions Query

1. **Add a Microsoft SQL node** (connect it to Webhook)
2. Name it: `Microsoft SQL - Customer Transactions`
3. Configure your database credentials
4. Set **Operation**: "Execute Query"
5. Enter this query:
```sql
SELECT 
    YEAR(TransactionDate) AS Year,
    MONTH(TransactionDate) AS Month,
    DATENAME(MONTH, TransactionDate) AS MonthName,
    SUM(TransactionAmount) AS TotalTransactionAmount
FROM [WideWorldImporters].[Reports].[CustomerTransactionsView]
WHERE YEAR(TransactionDate) = 2016
    AND TransactionAmount > 0
GROUP BY 
    YEAR(TransactionDate),
    MONTH(TransactionDate),
    DATENAME(MONTH, TransactionDate)
ORDER BY 
    MONTH(TransactionDate);
```

#### 2b. Stock Locations Query

1. **Add another Microsoft SQL node** (also connect to Webhook)
2. Name it: `Microsoft SQL - Stock Locations`
3. Use the same credentials
4. Query:
```sql
SELECT 
    BinLocation as name,
    SUM(LastStocktakeQuantity) AS value
FROM [WideWorldImporters].[Warehouse].[StockItemHoldings]
WHERE LastEditedWhen = '2016-05-31 07:00:00.0000000'
GROUP BY BinLocation
ORDER BY BinLocation;
```

#### 2c. Supplier Transactions Query

1. **Add a third Microsoft SQL node** (connect to Webhook)
2. Name it: `Microsoft SQL - Supplier Transactions`
3. Query:
```sql
SELECT 
    s.SupplierName as name,
    COUNT(*) AS value
FROM [WideWorldImporters].[Warehouse].[StockItemTransactions] t
INNER JOIN [WideWorldImporters].[Purchasing].[Suppliers] s
    ON t.SupplierID = s.SupplierID
GROUP BY s.SupplierID, s.SupplierName
```

### Step 3: Aggregate Each Data Stream

After each SQL query, add an **Aggregate node** to collect all rows into a single array:

1. **Add Aggregate node** after Customer Transactions
   - Name: `Aggregate - Customer Transactions`
   - **Aggregate**: "Aggregate All Item Data"
   - **Destination Field Name**: `customerTransactions`

2. **Add Aggregate node** after Stock Locations
   - Name: `Aggregate - Stock Locations`
   - **Destination Field Name**: `stockLocations`

3. **Add Aggregate node** after Supplier Transactions
   - Name: `Aggregate - Supplier Transactions`
   - **Destination Field Name**: `supplierTransactions`

### Step 4: Add Background Image (Optional)

This parallel branch loads a background image:

1. **Add a "Read Binary File" node** (connect to Webhook)
   - Name: `Read background image from disk`
   - **File Path**: `/usr/share/misc/sales-dash/dashboard.png`
   - (Adjust path to where your image is stored)

2. **Add a Code node** after it
   - Name: `JS - Get Background`
   - **Mode**: Run Once for All Items
   - Code:
```javascript
// Get the binary data
const binaryData = $input.all()[0].binary.data;

// Extract the base64 string
const base64String = binaryData.data;

// Create the data URL
const dataUrl = `data:image/png;base64,${base64String}`;

return [{
  json: {
    bgImage: dataUrl
  }
}];
```

### Step 5: Merge All Data Streams

1. **Add a Merge node**
   - **Mode**: "Merge By Position"
   - **Number of Inputs**: 4
2. Connect the four aggregate nodes to it:
   - Input 1: Aggregate - Customer Transactions
   - Input 2: Aggregate - Stock Locations
   - Input 3: Aggregate - Supplier Transactions
   - Input 4: JS - Get Background

### Step 6: Create Final Data Structure

1. **Add another Aggregate node** after Merge
   - Name: `Aggregate - Merged Data`
   - **Aggregate**: "Aggregate All Item Data"

This creates a single data object containing all four datasets.

### Step 7: Generate the HTML Dashboard

1. **Add an HTML node**
2. In the HTML field, paste this complete dashboard code:

```html
<html>
  <body>
<style>
 body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    font-size: 9pt;
    margin: 0;
    padding: 20px;
    background-image: url('{{ $json.data[3].bgImage }}');
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    background-attachment: fixed;
  }
  .chart-container {
    display: flex;
    flex-wrap: wrap;
    width: 100%;
    gap: 20px;
  }
  
  .chart-wrapper {
    flex: 0 0 calc(50% - 10px);
    max-width: calc(50% - 10px);
    background-color: #ffffff;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    padding: 20px;
    box-sizing: border-box;
  }
  
  .chart-wrapper:nth-child(3) {
    flex: 0 0 100%;
    max-width: 100%;
  }
  
  .chart-title {
    text-align: center;
    font-size: 18pt;
    font-weight: bold;
    margin-bottom: 20px;
    color: #333;
  }
  
  #chartdiv, #piechartdiv, #forcechartdiv {
    width: 100%;
    height: 400px;
  }
</style>
<script src="//cdn.amcharts.com/lib/4/core.js"></script>
<script src="//cdn.amcharts.com/lib/4/charts.js"></script>
<script src="//cdn.amcharts.com/lib/4/themes/animated.js"></script>
<script src="//cdn.amcharts.com/lib/4/plugins/forceDirected.js"></script> 
<script src="//cdn.amcharts.com/lib/4/themes/kelly.js"></script>
  <div class="chart-container">
      <div class="chart-wrapper">
        <div class="chart-title">Transactions</div>
        <div id="chartdiv"></div>
      </div>
      <div class="chart-wrapper">
        <div class="chart-title">Supplier Transactions</div>
        <div id="piechartdiv"></div>
      </div>
    <div class="chart-wrapper">
     <div class="chart-title">Stock Locations</div>
        <div id="forcechartdiv"></div>
    </div>
    </div> 

    <script type="text/javascript">
am4core.useTheme(am4themes_animated);
am4core.useTheme(am4themes_kelly);

// Column Chart - Customer Transactions
var chart = am4core.create("chartdiv", am4charts.XYChart);
chart.marginRight = 400;
chart.data = {{ $json.data[0].customerTransactions.toJsonString() }}

var categoryAxis = chart.xAxes.push(new am4charts.CategoryAxis());
categoryAxis.dataFields.category = "MonthName";
categoryAxis.title.text = "Month";
categoryAxis.renderer.grid.template.location = 0;
categoryAxis.renderer.minGridDistance = 20;

var valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
valueAxis.title.text = "Transaction Amount ($)";

var series = chart.series.push(new am4charts.ColumnSeries());
series.dataFields.valueY = "TotalTransactionAmount";
series.dataFields.categoryX = "MonthName";
series.name = "Total Transactions";
series.tooltipText = "{name}: [bold]${valueY.formatNumber('#,###.00')}[/]";
series.columns.template.fill = am4core.color("#67b7dc");
series.columns.template.stroke = am4core.color("#67b7dc");

chart.cursor = new am4charts.XYCursor();

// Pie Chart - Supplier Transactions
var pieChart = am4core.create("piechartdiv", am4charts.PieChart);
pieChart.data = {{ $json.data[2].supplierTransactions.toJsonString() }}

var pieSeries = pieChart.series.push(new am4charts.PieSeries());
pieSeries.dataFields.value = "value";
pieSeries.dataFields.category = "name";
pieChart.innerRadius = am4core.percent(40);
pieSeries.slices.template.stroke = am4core.color("#4a2abb");
pieSeries.slices.template.strokeWidth = 2;
pieSeries.slices.template.strokeOpacity = 1;
pieChart.legend = new am4charts.Legend();

// Force-Directed Chart - Stock Locations
var forceDirected = am4core.create("forcechartdiv", am4plugins_forceDirected.ForceDirectedTree);
var forceDirectedSeries = forceDirected.series.push(new am4plugins_forceDirected.ForceDirectedSeries())
forceDirectedSeries.data = {{ $json.data[1].stockLocations.toJsonString() }}
forceDirectedSeries.dataFields.name = "name";
forceDirectedSeries.dataFields.id = "name";
forceDirectedSeries.dataFields.value = "value";
forceDirectedSeries.nodes.template.label.text = "{name}"
forceDirectedSeries.nodes.template.tooltipText = "{value}";
    </script>
</body>
</html>
```

### Step 8: Send the Response

1. **Add a "Respond to Webhook" node**
2. Configure:
   - **Respond With**: "Text"
   - **Response Body**: `={{ $json.html }}`
   - This sends the generated HTML back to the browser

### Step 9: Connect Everything and Activate

1. Connect HTML node → Respond to Webhook node
2. Review your workflow - it should look like a tree with 4 parallel branches merging
3. **Activate** the workflow using the toggle in the top-right

---

## How It Works

When someone visits your webhook URL:

1. The webhook triggers and fires 4 parallel processes
2. Three SQL queries fetch different datasets simultaneously
3. Background image is loaded and converted to base64
4. Each dataset is aggregated into arrays
5. All four data streams merge together
6. Final aggregation creates one object with all data
7. HTML node injects the data into the dashboard template
8. Browser receives interactive HTML with live charts

## Customization Tips

**Change the queries**: Modify the SQL to match your database schema and tables

**Adjust chart types**: Explore amCharts documentation to use bar charts, line charts, scatter plots, etc.

**Style the dashboard**: Modify the CSS in the HTML node to match your brand colors

**Add more charts**: Expand the merge node inputs and add additional data streams

**Refresh interval**: Add a Schedule Trigger to regenerate data periodically

**Authentication**: Add HTTP Basic Auth to the webhook node for security

## Troubleshooting

- **No data showing**: Check SQL credentials and table names
- **Charts not rendering**: Ensure CDN scripts are loading (check browser console)
- **Background image missing**: Verify file path and permissions
- **Merge errors**: Ensure all 4 inputs are connected and data flows correctly

---

This workflow showcases n8n's power for creating data-driven applications without traditional backend development. You've now built a production-ready business intelligence dashboard!