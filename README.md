# Unified Power BI Semantic Model: Advanced Business Analytics with DAX & M

## Table of Contents

- [The Need for a Comprehensive Data Model](#the-need-for-comprehensive-data-model)

- [Tools](#tools)

- [Tables](#tables)
  - [Date Table](#date-table)
  - [Sales Table](#sales-table)
...

## The Need for a Comprehensive Data Model

Company faced significant challenges due to fragmented and inconsistent data scattered across multiple sources, such as sales records, purchase orders, inventory details, customer interactions, and financial accounts. This fragmentation hindered the company’s ability to generate timely, accurate, and comprehensive insights necessary for strategic decision-making. Establishing a unified, structured, and comprehensive data analytics model became crucial to streamline data integration, ensure data integrity, and empower Suspensia with actionable insights for improved business performance and informed decision-making.

## Tools

- PowerBI
- SQL Server

## Tables

### Date Table

The Date Table serves as the foundational time dimension for the entire data model, enabling consistent and accurate time-based analytics and comparisons. It provides a standardized way to analyze data across different time periods such as yearly, quarterly, monthly, weekly, and daily views.

#### **Source:**
Created dynamically using **Power Query (M language)** within Power BI. This method ensures the date table automatically updates to include current dates.

#### **Columns Included:**
| Column Name   | Description                                      | Example        |
|---------------|--------------------------------------------------|----------------|
| `Date`        | Individual date entries                          | 2024-01-01     |
| `Year`        | Year number                                      | 2024           |
| `Quarter`     | Quarter name                                     | Q1, Q2, Q3, Q4 |
| `QuarterNo`   | Quarter number                                   | 1, 2, 3, 4     |
| `Month`       | Month name                                       | January        |
| `MonthNo`     | Month number                                     | 1              |
| `Week`        | Week number within the year                      | 1-52           |
| `WeekNo`      | Week number aligned to fiscal or calendar weeks  | 1-52           |
| `Day`         | Day of the month                                 | 1-31           |
| `DayOfWeekNo` | Numeric representation of the weekday            | 1 (Monday)     |

Below is the M code to dynamically generate a Date Table within Power BI, which automatically updates from January 1, 2018, to the current date:


```m language
let
    //Variables
    Start_Date = #date(2018, 1, 1),
    End_Date = DateTime.Date(DateTime.LocalNow()),
    Duration_VAR = Duration.Days(Duration.From(End_Date-Start_Date))+1,
    Dates = List.Dates(Start_Date,Duration_VAR,#duration(1,0,0,0)),
    #"Converted to Table" = Table.FromList(Dates, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Renamed Columns" = Table.RenameColumns(#"Converted to Table",{{"Column1", "Date"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"Date", type date}}),
    #"Inserted Year" = Table.AddColumn(#"Changed Type", "Year", each Date.Year([Date]), Int64.Type),
    #"Inserted QuarterNo" = Table.AddColumn(#"Inserted Year", "QuarterNo", each Date.QuarterOfYear([Date]), Int64.Type),
    #"Custom Column for Quarter" = Table.AddColumn(#"Inserted QuarterNo", "Quarter", each "Q"&Text.From([QuarterNo])),
    #"Inserted Month" = Table.AddColumn(#"Custom Column for Quarter", "Month", each Text.Start(Date.MonthName([Date]),3), type text),
    #"Inserted MonthNo" = Table.AddColumn(#"Inserted Month", "MonthNo", each Date.Month([Date]), Int64.Type),
    #"Inserted WeekNo" = Table.AddColumn(#"Inserted MonthNo", "WeekNo", each Date.WeekOfYear([Date]), Int64.Type),
    #"Custom Column for Week" = Table.AddColumn(#"Inserted WeekNo", "Week", each "Week "&Text.From([WeekNo])),
    #"Inserted Day" = Table.AddColumn(#"Custom Column for Week", "Day", each Text.Start(Date.DayOfWeekName([Date]),3), type text),
    #"Inserted DayOfWeekNo" = Table.AddColumn(#"Inserted Day", "DayOfWeekNo", each Date.DayOfWeek([Date]), Int64.Type)
in
    #"Inserted DayOfWeekNo"
```

### Sales Table

#### Purpose
The **Sales Table** is the central repository capturing detailed transactional information related to product sales. It enables comprehensive analysis of sales performance, revenue generation, profitability, and customer purchasing behaviors over time. The CSV files containing sales data are automatically exported to the company server through scheduled exports from the accounting software, ensuring timely and accurate data integration.

#### Source and Creation Method
- **Source:** Multiple CSV files (`SALES_SUS_2018_2023.csv`, `SALES_SUS_2024.csv`, `SALES_SUS_2025.csv`)
- **Creation Method:** Imported, merged, cleaned, and structured using Power Query (M language).

#### (M) Code to Form Sales Table

The following M code is used to load, clean, and prepare sales data from three yearly CSV exports. Each step is explained below.

```powerquery-m
let
    // Combine multiple yearly CSV tables into one unified sales table
    Source = Table.Combine({SALES_SUS_2018_2023, SALES_SUS_2024, SALES_SUS_2025}),

    // Remove unnecessary or unused columns to simplify the model
    #"Removed Columns" = Table.RemoveColumns(Source,{
        "Apply to Invoice Number", "Progress Billing Invoice", "Ship By", "Quote", "Quote #", 
        "Quote Good Thru Date", "Ship Via", "Ship Date", "Date Due", "Sales Tax ID", 
        "Invoice Note", "Note Prints After Line Items", "Statement Note", 
        "Stmt Note Prints Before Ref", "Internal Note", "Beginning Balance Transaction", 
        "AR Date Cleared in Bank Rec", "Number of Distributions", "Invoice/CM Distribution", 
        "Apply to Invoice Distribution", "Apply To Sales Order", "Apply to Proposal", 
        "Serial Number", "SO/Proposal Distribution", "Weight", "Stocking Quantity", 
        "Stocking Unit Price", "Return Authorization", "Receipt Number", 
        "Voided by Transaction", "Retainage Percent", "Recur Number", "Recur Frequency", 
        "Ship to Name", "Ship to Address-Line One", "Ship to Address-Line Two", 
        "Discount Amount", "Discount Date", "Displayed Terms", "Accounts Receivable Account", 
        "Accounts Receivable Amount", "SO/Proposal Number", "GL Date Cleared in Bank Rec", 
        "Tax Type", "UPC / SKU", "Inv Acnt Date Cleared In Bank Rec", 
        "COS Acnt Date Cleared In Bank Rec", "U/M ID", "U/M No. of Stocking Units", 
        "Job ID", "Sales Tax Agency ID", "Transaction Period", "Transaction Number", 
        "Description", "Inventory Account"
    }),

    // Rename 'Item ID' to 'Sold AS (Item ID)' for clarity and consistency
    #"Item ID Renamed Sold AS" = Table.RenameColumns(#"Removed Columns",{{"Item ID", "Sold AS (Item ID)"}}),

    // Invert the sign of 'Amount' to convert negative values (used in accounting) to positives
    #"Amount Multiplied by (-1)" = Table.TransformColumns(#"Item ID Renamed Sold AS", {
        {"Amount", each _ * -1, Currency.Type}
    }),

    // Rename columns to match semantic model naming conventions
    #"Renamed Columns" = Table.RenameColumns(#"Amount Multiplied by (-1)", {
        {"Amount", "Sales_Amount"},
        {"Quantity", "Sales_Quantity"},
        {"Unit Price", "Sales_Unit Price"}
    }),

    // Ensure 'Cost of Sales Amount' is explicitly typed as a number
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns", {
        {"Cost of Sales Amount", type number}
    }),

    // Replace empty strings in item IDs with the literal string \"(blank)\"
    #"Blanks replaced with \"(blank)\"" = Table.ReplaceValue(#"Changed Type", "", "(blank)", Replacer.ReplaceValue, {"Sold AS (Item ID)"})
in
    #"Blanks replaced with \"(blank)\""
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                               | Example           |
|-------------------------|-----------|-----------------------------------------------------------|-------------------|
| `Customer ID`           | Text      | Unique identifier for the customer                        | CUST1001          |
| `Customer Name`         | Text      | Name of the customer                                      | ABC Corp.         |
| `Invoice/CM #`          | Text      | Invoice number or Credit Memo reference                   | INV-2023-10001    |
| `Date`                  | Date      | Date of the sales transaction                             | 2024-01-20        |
| `Credit Memo`           | Text      | Indicates if the transaction is a credit memo             | Yes / No          |
| `Drop Ship`             | Text      | Indicates if item was drop-shipped directly               | Yes / No          |
| `Sold AS (Item ID)`     | Text      | Identifier for the sold item                              | ITEM56789         |
| `Sales_Quantity`        | Number    | Quantity of the item sold                                 | 150               |
| `G/L Account`           | Text      | General Ledger account associated with sales revenue      | 4000-Sales Rev.   |
| `Sales_Unit Price`      | Currency  | Selling price per unit of the item                        | 49.99             |
| `Sales_Amount`          | Currency  | Total sales amount (Quantity × Unit Price)                | 7498.50           |
| `Cost of Sales Account` | Text      | G/L account linked to cost of sales                       | 5000-Cost of Sales|
| `Cost of Sales Amount`  | Currency  | Total cost of the items sold                              | 4000.00           |

#### Usage and Analytical Value
This table forms the backbone of sales analytics, allowing the following analyses:

- **Revenue Analysis:** Evaluating total revenue by period, item, customer, and region.
- **Profitability Analysis:** Determining margins, profitability per item, and cost management effectiveness.
- **Trend Analysis:** Monitoring sales trends to support forecasting and strategic planning.
- **Customer Insights:** Understanding customer buying behavior and segmentation.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via the `Date` column for robust temporal analysis.
- **Linked to Item Information Table (`Item_Info`)** via `Sold AS (Item ID)` for enriched item-level analytics.
- **Linked to Customer Information** via `Customer ID` for customer demographic and segmentation insights.
- **Linked to General Ledger Accounts (`COA_CONS`)** via `G/L Account` to enable detailed financial analytics.

