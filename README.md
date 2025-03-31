# Unified Power BI Semantic Model: Advanced Business Analytics with DAX & M

## Table of Contents

- [The Need for a Comprehensive Data Model](#the-need-for-comprehensive-data-model)

- [Tools](#tools)

- [Tables](#tables)
  - [Date Table](#date-table)
  - [Item Info Table](#item-info-table)
  - [Customers Table](#customers-table)
  - [Sales Table](#sales-table)
  - [SO Table (Sales Orders)](#so-table-sales-orders)
  - [Purchases Table](#purchases-table)
  - [PO Table (Purchase Order)](#po-table-purchase-orders)
  - [Chart of Accounts Table](#chart-of-accounts-table)
  - [Sales Representatives Table (Reps)](#sales-representatives-table-reps)

...

## The Need for a Comprehensive Data Model

Company faced significant challenges due to fragmented and inconsistent data scattered across multiple sources, such as sales records, purchase orders, inventory details, customer interactions, and financial accounts. This fragmentation hindered the company’s ability to generate timely, accurate, and comprehensive insights necessary for strategic decision-making. Establishing a unified, structured, and comprehensive data analytics model became crucial to streamline data integration, ensure data integrity, and empower Suspensia with actionable insights for improved business performance and informed decision-making.

## Tools

- PowerBI
- SQL Server

## Tables

Below is a list of tables loaded and structured within Power BI. Each table serves a distinct purpose and is either linked to SQL servers, loaded as-is from flat files or transformed through Power Query (M language). Some tables are created dynamically (e.g., Date Table), while others are merged or cleaned versions of raw exports from the company’s accounting system or supporting databases.

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

---

### Item Info Table

#### Purpose
The **Item_Info** table serves as the central item dimension, containing key metadata used across sales, purchase, and inventory tables. It ensures product consistency and traceability, even when part numbers evolve over time due to OE changes or engineering updates — a common need in the automotive industry.

This table enables robust item-level analytics by standardizing descriptions, categories, and cross-references. It also merges external references and catalog data for enriched analysis.

#### Source and Creation Method
- **Primary Source:** `SQL Server` Catalog_Items
- **Supplementary Tables (SQL Server):**
  - `Number_Change_Track_Table`: Used during transformation to update obsolete part numbers.
  - `Buyersguide_Crossed`: Merged into the final result for catalog-based enrichment.

- **Creation Method:**
  - Loaded via Power Query.
  - Merged with `Number_Change_Track_Table` to replace superseded `First_SUS#` values with the latest `Final_SUS#`.
  - Enriched with automotive application data from `Buyersguide_Crossed`.

#### Special Handling: Superseded Part Numbers

In the automotive industry, it’s common for part numbers to be superseded over time. To manage this:

- The **`Number_Change_Track_Table`** includes:
  - `First_SUS#`: The old part number being replaced.
  - `ChangeDate`: When the change occurred.
  - `Final_SUS#`: The new replacement part number.

- This table is not loaded into the Power BI data model directly, but used in **Power Query transformations**.
- The merge ensures all outdated part numbers in `Item_Info` are updated to the most recent version, maintaining a consistent identifier (`Final_SUS#`) across time-series data and preventing fragmentation in analysis.

#### Additional Enrichment: Buyers Guide Merge

To provide rich automotive catalog context, `Item_Info` is further enriched with data from the **`Buyersguide_Crossed`** SQL table, which includes:

- Vehicle compatibility (Make, Model, Year)
- OEM and aftermarket cross-references
- Application-specific part descriptions
- Quantity required per vehicle
- Opposite-side part suggestions

This allows the model to answer real-world catalog questions such as:
- “What does this part fit?”
- “What is the OE number for this SKU?”
- “Is there an opposite side part I should stock?”

#### (M) Code to Form Item Info Table

This M code loads and transforms item metadata from a SQL Server table. It includes logic for superseded part number resolution and catalog enrichment. These transformations ensure SKU continuity and complete vehicle fitment context — which is critical for operations in the automotive aftermarket.

```powerquery-m
let
    // Step 1: Connect to SQL Server and load the item master table
    Source = Sql.Database("SERVER_NAME", "Suspensia_DB"),
    ItemMaster = Source{[Schema="dbo", Item="SUS_SAGE_ITEM_CROSS"]}[Data],

    // Step 2: Promote the first row to column headers (if needed)
    #"Promoted Headers" = Table.PromoteHeaders(ItemMaster, [PromoteAllScalars=true]),

    // Step 3: Set data types for core fields
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {
        {"Item ID", type text},
        {"Item Description", type text},
        {"Description for Sales", type text},
        {"Description for Purchases", type text},
        {"Suspensia Cross", type text},
        {"TYPE", type text}
    }),

    // Step 4: Rename columns for semantic clarity
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type", {
        {"Suspensia Cross", "Before_Superseed"},
        {"TYPE", "PL or SUS"},
        {"Item ID", "Sold As (Item ID)"}
    }),

    // Step 5: Merge with Number_Change_Track table to replace outdated part numbers
    #"Merged Number_Change_Track" = Table.NestedJoin(#"Renamed Columns", {"Before_Superseed"}, Number_Change_Track, {"FirstSus#"}, "Number_Change_Track", JoinKind.LeftOuter),

    // Step 6: Expand final part number from tracking table
    #"Expanded NumberChangeTrack_Q" = Table.ExpandTableColumn(#"Merged Number_Change_Track", "Number_Change_Track", {"FinalSus#"}, {"FinalSus#"}) ,

    // Step 7: Use final number if available, else fallback to original
    #"Added Conditional Column" = Table.AddColumn(#"Expanded NumberChangeTrack_Q", "SUS#", each if [#"FinalSus#"] = null then [Before_Superseed] else [#"FinalSus#"]),

    // Step 8: Remove temp fields
    #"Removed Columns" = Table.RemoveColumns(#"Added Conditional Column",{"Before_Superseed", "FinalSus#"}),

    // Step 9: Ensure consistent data type
    #"Changed Type1" = Table.TransformColumnTypes(#"Removed Columns",{{"SUS#", type text}}),

    // Step 10: Merge BuyersGuide catalog data from SQL
    #"Merged Queries" = Table.NestedJoin(#"Changed Type1", {"SUS#"}, BuyersGuide_Crossed, {"Sus"}, "BuyersGuide_Crossed", JoinKind.LeftOuter),

    // Step 11: Expand catalog fields
    #"Expanded BuyersGuide_Crossed_Q" = Table.ExpandTableColumn(#"Merged Queries", "BuyersGuide_Crossed", {"Category", "OppositeSide"}, {"Category", "OppositeSide"}),

    // Step 12: Remove duplicate item entries
    #"Removed Duplicates" = Table.Distinct(#"Expanded BuyersGuide_Crossed_Q", {"Sold As (Item ID)"})
in
    #"Removed Duplicates"
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                                  | Example             |
|-------------------------|-----------|--------------------------------------------------------------|---------------------|
| `Sold As (Item ID)`     | Text      | Primary identifier (includes resolved supersessions)         | ITEM54321           |
| `Item Description`      | Text      | Generic item name                                             | Rear Shock Absorber |
| `Description for Sales` | Text      | Label shown on sales documents or dashboards                 | Rear Shock - Std    |
| `Description for Purchases` | Text  | Used internally for procurement reference                    | Absorber Rear S/22  |
| `PL or SUS`             | Text      | Product line or brand group identifier                       | PL123               |
| `SUS#`                  | Text      | Original part number (may be updated by merge)               | SUS56789            |
| `Category`              | Text      | Item classification for filtering                            | Suspension          |
| `OppositeSide`          | Text      | Alternate item ID (e.g., right-side counterpart)             | ITEM54322           |
| `OEM_Crosses`           | Text      | Original equipment manufacturer (OE) references              | 12345678            |
| `Aftermarket_Crosses`   | Text      | Alternate brands with matching specs                         | AM4321              |
| `Application`           | Text      | Vehicles that use this part (e.g., Ford Focus 2015-2018)     | Toyota Corolla 18–21|
| `Qty_Per_Vehicle`       | Number    | Quantity needed per vehicle                                  | 2                   |

> **Note:** Not all enrichment fields may be used in the data model directly but are available for lookup and dynamic analysis when needed.

#### Usage and Analytical Value
- **SKU Performance Tracking:** Maintains SKU continuity even across part number changes.
- **Cross-Sell and Recommendations:** Enables pairing items like left/right-side parts.
- **Catalog Lookup:** Provides full visibility into vehicle fitment, crosses, and part functions.
- **Standardized Item Reporting:** Centralizes descriptions and categories for clean dashboards.

#### Relationships (Brief Overview)
- **Linked to Sales, Purchases, and PO Tables** via standardized `Item ID`
- **Joined with Number Change Tracker during transformation**
- **Enriched from Buyersguide_Crossed SQL table**

---

### Customers Table

#### Purpose
The **Customers Table** is a key dimension table used to provide context and metadata for sales transactions. It enables analysis of customer behaviors, segmentation, and performance. By linking each sales record to its associated customer, the model can generate insights into top clients, region-based performance, and purchasing patterns.

#### Source and Creation Method
- **Source File:** `ISC_Customers.xlsx`. This file is updated through scheduled exports through companys accounting system.
- **Creation Method:** Loaded into Power BI using Power Query. Certain customer IDs are anonymized to preserve privacy in exported or public reports.

### (M) Code Explanation – Customers Table

The following M code loads customer metadata from Excel, promotes headers, assigns correct data types, and prepares the table for use in customer-level reporting in Power BI.

```powerquery-m
let
    // Step 1: Load the Excel workbook from the network path
    Source = Excel.Workbook(File.Contents("V:\\[confidential].xlsx"), null, true),

    // Step 2: Select the worksheet named 'All_Cust_Info'
    All_Cust_Info_Sheet = Source{[Item="All_Cust_Info", Kind="Sheet"]}[Data],

    // Step 3: Temporarily assign text type to all columns before header promotion
    #"Changed Type" = Table.TransformColumnTypes(All_Cust_Info_Sheet, {
        {"Column1", type text},
        {"Column2", type text},
        {"Column3", type text},
        {"Column4", type text},
        {"Column5", type text}
    }),

    // Step 4: Promote the first row to headers
    #"Promoted Headers" = Table.PromoteHeaders(#"Changed Type", [PromoteAllScalars=true]),

    // Step 5: Assign correct data types to each column after promotion
    #"Changed Type1" = Table.TransformColumnTypes(#"Promoted Headers", {
        {"Customer ID", type text},
        {"Customer Name", type text},
        {"Customer Type", type text},
        {"Cust_Segment", type text},
        {"Cust_ID_Anonym", type text}
    })
in
    #"Changed Type1"
```

#### Detailed Column Descriptions

| Column Name        | Data Type | Description                                                       | Example            |
|--------------------|-----------|-------------------------------------------------------------------|--------------------|
| `Customer ID`      | Text      | Unique identifier used across all sales data                      | CUST1234           |
| `Customer Name`    | Text      | Full name of the customer or company                              | West Auto Supply   |
| `Customer Type`    | Text      | Type/category of customer (e.g., Distributor, Installer, Retailer)| Distributor         |
| `Cust_Segment`     | Text      | Segmentation label for grouping customers                         | Tier 1             |
| `Cust_ID_Anonym`   | Text      | Anonymized version of the customer ID used for external sharing   | CUST_A01           |

#### Usage and Analytical Value
- **Customer Segmentation:** Analyze behavior by segment (e.g., Tiers, Regions, Customer Types)
- **Top Customers Reporting:** Identify high-volume or high-margin clients.
- **Anonymized Reporting:** Safely share analytics without exposing real customer identities.
- **Sales Rep Performance:** If combined with sales rep table, helps attribute performance by rep-customer relationship.

#### Relationships (Brief Overview)
- **Linked to the Sales Table** via `Customer ID`
-  **Linked to the SO Table** via `Customer ID`

This table plays a vital role in enabling B2B-focused sales analysis, customer profiling, and performance reporting.

---

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

---

### SO Table (Sales Orders)

#### Purpose
The **SO Table** (Sales Orders) contains detailed records of open or historical sales orders, proposals, and quotes received from customers. It is primarily used to track future commitments, analyze sales pipelines, manage fulfillment expectations, and assess sales potential that hasn't yet been invoiced. This complements the Sales Table, which tracks completed transactions.

Sales Order data is exported from the accounting software to the company server in CSV format on a scheduled basis.

#### Source and Creation Method
- **Source Files:** `SO_SUS_2018_2023.csv`, `SO_SUS_2024.csv`, `SO_SUS_2025.csv`
- **Creation Method:** The CSVs are combined and transformed using Power Query (M). Unnecessary columns are removed, numeric values are type-corrected, and fields are renamed for clarity.

#### (M) Code to Form Sales Table

```m-language
let
    // Combine multiple year-based SO files into one table
    Source = Table.Combine({SO_SUS_2018_2023, SO_SUS_2024, SO_SUS_2025}),

    // Remove unnecessary or irrelevant columns from the combined data
    #"Removed Columns" = Table.RemoveColumns(Source, {
        "Ship By", "Proposal", "Proposal Accepted", "Closed", "Quote #",
        "Ship to Name", "Ship to Address-Line One", "Ship to Address-Line Two",
        "Ship to City", "Ship Via", "Sales Tax ID", "Invoice Note",
        "Note Prints After Line Items", "Statement Note", "Stmt Note Prints Before Ref",
        "Internal Note", "Number of Distributions", "SO/Proposal Distribution",
        "Weight", "U/M No. of Stocking Units", "Stocking Quantity", "Stocking Unit Price",
        "Sales Tax Agency ID", "Discount Amount", "Transaction Period",
        "Transaction Number", "U/M ID", "Tax Type", "Sales Order/Proposal #",
        "Ship to State", "Ship to Zipcode", "Ship to Country", "Customer PO",
        "Displayed Terms", "Accounts Receivable Account", "Accounts Receivable Amount",
        "UPC / SKU", "Job ID", "Description"
    }),

    // Ensure Amount and Unit Price columns are interpreted as Currency data types
    #"Changed Type" = Table.TransformColumnTypes(#"Removed Columns", {
        {"Amount", Currency.Type},
        {"Unit Price", Currency.Type}
    }),

    // Convert Amounts to negative values (likely to align with accounting standards or match invoice logic)
    #"Amount Multiplied by (-1)" = Table.TransformColumns(#"Changed Type", {
        {"Amount", each _ * -1, Currency.Type}
    }),

    // Rename columns to match semantic naming conventions used in the model
    #"Renamed Columns" = Table.RenameColumns(#"Amount Multiplied by (-1)", {
        {"Item ID", "Sold As (Item ID)"},
        {"Amount", "SO_Amount"},
        {"Quantity", "SO_Quantity"},
        {"Unit Price", "SO_Unit Price"}
    }),

    // Replace blank values in the Item ID column with the string "(blank)"
    #"Blanks replaced with \"(blank)\"" = Table.ReplaceValue(#"Renamed Columns", "", "(blank)", Replacer.ReplaceValue, {"Sold As (Item ID)"})
in
    #"Blanks replaced with \"(blank)\""
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                                | Example           |
|-------------------------|-----------|------------------------------------------------------------|-------------------|
| `Customer ID`           | Text      | Unique customer identifier                                 | CUST2034          |
| `Customer Name`         | Text      | Name of the customer placing the order                     | Beta Industries   |
| `Date`                  | Date      | Date when the sales order was entered                      | 2024-03-10        |
| `Drop Ship`             | Logical   | Indicates whether this is a drop-shipped order             | TRUE              |
| `Sales Representative ID` | Text    | Identifier of the responsible sales representative         | REP0003           |
| `Item ID` (renamed to `Sold As (Item ID)`) | Text | Identifier of the ordered item                 | ITEM32991         |
| `Quantity` (renamed to `SO_Quantity`) | Number | Quantity ordered                                 | 80                |
| `Unit Price` (renamed to `SO_Unit Price`) | Currency | Price per unit ordered                         | 45.00             |
| `Amount` (renamed to `SO_Amount`) | Currency | Total sales order value for the line item                 | 3600.00           |
| `G/L Account`           | Int64     | Financial ledger account linked to the order               | 4000              |

#### Usage and Analytical Value
- **Backlog Analysis:** Identifies committed revenue not yet realized.
- **Sales Pipeline Monitoring:** Tracks pending and open orders.
- **Demand Planning:** Helps forecast future inventory needs based on pending orders.
- **Performance Monitoring:** Evaluates quoting and proposal effectiveness over time.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via the `Date` field for tracking open orders over time.
- **Linked to the Item Table (`Item_Info`)** via `Sold As (Item ID)` to analyze product-level demand.
- **Linked to Customer Information** via `Customer ID` to evaluate outstanding commitments by customer.

This table complements completed sales data by providing visibility into expected future business and supporting operational and financial forecasting.

---

### Purchases Table

#### Purpose
The **Purchases Table** records historical procurement transactions, capturing detailed information about goods or services acquired from vendors. It supports analysis of purchasing behavior, cost tracking, vendor performance, and inventory planning.

The table serves as a counterpart to the Sales Table and is essential for cost control, margin analysis, and operational decision-making.

#### Source and Creation Method
- **Source Files:** `PJ_SUS_2018_2024.csv`, `PJ_SUS_2025.csv`
- **Creation Method:** Combined and transformed using Power Query (M). Extraneous columns are removed, numeric types are enforced, and standard naming conventions are applied.

#### (M) Code to Form Purchases Table

Below is a breakdown of the M code used to generate and transform the `Purchases` table in Power BI.

```powerquery-m
let
    // Step 1: Combine purchase records from two CSV sources (multiple years)
    Source = Table.Combine({PJ_SUS_2018_2024, PJ_SUS_2025}),

    // Step 2: Explicitly define data types for relevant columns
    #"Changed Type" = Table.TransformColumnTypes(Source,{
        {"Credit Memo", type logical},
        {"Date", type date},
        {"Drop Ship", type logical},
        {"Waiting on Bill", type logical},
        {"Date Due", type date},
        {"Discount Date", type date},
        {"Discount Amount", type number},
        {"Accounts Payable Account", type text},
        {"Accounts Payable Amount", Currency.Type},
        {"Note Prints After Line Items", type logical},
        {"Applied To Purchase Order", type logical},
        {"Number of Distributions", Int64.Type},
        {"Invoice/CM Distribution", Int64.Type},
        {"Apply to Invoice Distribution", Int64.Type},
        {"PO Distribution", Int64.Type},
        {"Quantity", type number},
        {"Stocking Quantity", type number},
        {"U/M No. of Stocking Units", Int64.Type},
        {"G/L Account", type text},
        {"GL Date Cleared in Bank Rec", type date},
        {"Unit Price", Currency.Type},
        {"Stocking Unit Price", type number},
        {"UPC / SKU", type text},
        {"Weight", type number},
        {"Amount", Currency.Type},
        {"Transaction Number", Int64.Type},
        {"Transaction Period", Int64.Type},
        {"Used for Reimbursable Expense", type logical}
    }),

    // Step 3: Remove unnecessary columns to clean and reduce the dataset
    #"Removed Columns" = Table.RemoveColumns(#"Changed Type",{
        "Apply to Invoice Number", "Customer SO #", "Waiting on Bill", "Customer ID", "Customer Invoice #",
        "Ship to Address-Line One", "Ship to Address-Line Two", "Ship to City", "Ship to State",
        "Ship to Country", "Date Due", "Discount Date", "Discount Amount", "Ship Via", "P.O. Note",
        "Note Prints After Line Items", "Beginning Balance Transaction", "AP Date Cleared in Bank Rec",
        "Applied To Purchase Order", "Number of Distributions", "Apply to Invoice Distribution", "PO Number",
        "PO Distribution", "Stocking Quantity", "Serial Number", "U/M ID", "U/M No. of Stocking Units",
        "GL Date Cleared in Bank Rec", "Stocking Unit Price", "UPC / SKU", "Weight", "Job ID",
        "Used for Reimbursable Expense", "Transaction Period", "Transaction Number", "Displayed Terms",
        "Return Authorization", "Row Type", "Recur Number", "Recur Frequency", "Ship to Name",
        "Description", "Accounts Payable Amount", "Invoice/CM Distribution", "Ship to Zipcode",
        "Accounts Payable Account"
    }),

    // Step 4: Rename key columns for semantic clarity and consistency
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{
        {"Amount", "Purchase_Amount"},
        {"Quantity", "Purchase_Quantity"},
        {"Unit Price", "Purchase_Unit Price"}
    })
in
    #"Renamed Columns"
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                                | Example           |
|-------------------------|-----------|------------------------------------------------------------|-------------------|
| `Vendor ID`             | Text      | Unique identifier for the vendor                           | VEND1021          |
| `Vendor Name`           | Text      | Name of the vendor or supplier                             | Acme Suppliers     |
| `Invoice/CM #`          | Text      | Invoice or Credit Memo number                              | PJ-2023-00213      |
| `Date`                  | Date      | Date of the purchase transaction                           | 2024-01-05         |
| `Credit Memo`           | Text      | Indicates if the transaction is a credit memo              | Yes / No           |
| `Drop Ship`             | Logical   | True if goods were drop-shipped directly                   | TRUE               |
| `Item ID` (renamed to `Purchased As (Item ID)`) | Text | Identifier of the purchased item         | ITEM89021          |
| `Purchase_Quantity`     | Number    | Number of units purchased                                  | 120                |
| `Purchase_Unit Price`   | Currency  | Cost per unit purchased                                    | 22.50              |
| `Purchase_Amount`       | Currency  | Total purchase amount (Quantity × Unit Price)              | 2700.00            |
| `G/L Account`           | Text      | General Ledger account for the purchase transaction        | 5000-Cost of Sales |

#### Usage and Analytical Value
- **Cost Analysis:** Evaluates cost per item and purchasing trends.
- **Vendor Analytics:** Tracks performance and purchase volume by vendor.
- **Margin Monitoring:** Used in combination with Sales data to calculate gross profit.
- **Inventory Planning:** Informs demand forecasting and restocking cycles.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via `Date` for time-based analysis.
- **Linked to the Item Table (`Item_Info`)** via `Purchased As (Item ID)` to understand item-level purchase behavior.
- **Linked to Vendor dimension** via `Vendor ID` for supplier-based analysis.
- **Linked to COA Table (`COA_CONS`)** via `G/L Account` for financial reporting and cost classification.

---

### PO Table (Purchase Orders)

#### Purpose
The **Purchase Orders Table** tracks procurement intents — orders placed with vendors for future delivery. Unlike the Purchases Table (which reflects completed transactions), this table captures **open commitments** and allows the business to manage expected inventory inflows, vendor reliability, and purchasing trends.

CSV files are automatically exported from the accounting system and uploaded to the company server as part of a scheduled integration routine.

#### Source and Creation Method
- **Source Files:** `POJ_2018_2024.csv`, `POJ_2025.csv`
- **Creation Method:** Combined using Power Query (M language), followed by removal of irrelevant fields and renaming for clarity and consistency.

#### M Code to Form PO Table

The following M code loads, cleans, and transforms the Purchase Orders data from two annual sources (`POJ_2018_2024` and `POJ_2025`). This prepares it for use in Power BI’s semantic model.

```powerquery-m
let
    // Step 1: Combine two PO data sources into one unified table
    Source = Table.Combine({POJ_2018_2024, POJ_2025}),

    // Step 2: Remove unnecessary columns to keep only relevant fields for analysis
    #"Removed Columns" = Table.RemoveColumns(Source,{
        "Remit to Address Line 1", "Remit to Address Line 2", "Remit to City", "Remit to State",
        "Remit to Zip", "Remit to Country", "Closed", "P.O. Good Thru Date", "Customer SO #",
        "Customer Invoice #", "Customer ID", "Ship to Name", "Ship to Address-Line One",
        "Ship to Address-Line Two", "Ship to City", "Ship to State", "Ship to Zipcode",
        "Ship to Country", "Discount Amount", "Displayed Terms", "Accounts Payable Account",
        "Accounts Payable Amount", "Ship Via", "P.O. Note", "Internal Note",
        "Note Prints After Line Items", "Number of Distributions", "PO Distribution",
        "Stocking Quantity", "U/M ID", "U/M No. of Stocking Units", "Stocking Unit Price",
        "UPC / SKU", "Weight", "Job ID", "Transaction Period", "Transaction Number",
        "Description"
    }),

    // Step 3: Assign correct data types to key columns
    #"Changed Type" = Table.TransformColumnTypes(#"Removed Columns",{
        {"Amount", Currency.Type},
        {"Unit Price", Currency.Type},
        {"Quantity", Int64.Type},
        {"PO #", type text}
    }),

    // Step 4: Rename columns to match the semantic model's naming conventions
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type",{
        {"Unit Price", "PO_Unit Price"},
        {"Amount", "PO_Amount"},
        {"Quantity", "PO_Quantity"}
    })
in
    #"Renamed Columns"
```

#### Detailed Column Descriptions

| Column Name       | Data Type | Description                                                | Example          |
|-------------------|-----------|------------------------------------------------------------|------------------|
| `Vendor ID`       | Text      | Unique identifier for the vendor                          | VEND0041         |
| `Vendor Name`     | Text      | Name of the vendor company                                | Global Parts Inc.|
| `PO #`            | Text      | Unique Purchase Order number                              | PO-2024-00891    |
| `Date`            | Date      | Date the purchase order was issued                        | 2024-03-15       |
| `Drop Ship`       | Logical   | Indicates if the order is to be shipped directly to customer | TRUE          |
| `Item ID`         | Text      | Identifier of the product ordered                         | ITEM47592        |
| `PO_Quantity`     | Number    | Number of units ordered                                   | 200              |
| `PO_Unit Price`   | Currency  | Unit cost of the ordered item                             | 11.25            |
| `PO_Amount`       | Currency  | Total order value (Quantity × Unit Price)                 | 2250.00          |
| `G/L Account`     | Int64     | General Ledger account used for PO financial tracking     | 5000             |

#### Usage and Analytical Value
- **Open Order Monitoring:** Understand what inventory is on the way and from which vendors.
- **Procurement Planning:** Measure purchasing frequency and lead times.
- **Vendor Performance:** Evaluate if suppliers fulfill POs as promised.
- **Cash Flow Planning:** Anticipate financial commitments before they're invoiced.

#### Relationships (Brief Overview)
- **Linked to the Date Table** via `Date` for time-based PO tracking.
- **Linked to the Item Table (`Item_Info`)** via `Item ID` to analyze product-level demand.
- **Linked to Vendor dimension** via `Vendor ID` to review supplier-based activity.
- **Linked to the COA Table (`COA_CONS`)** via `G/L Account` to associate POs with financial accounts.

---

### Chart of Accounts Table

#### Purpose
The **COA_CONS** table (Chart of Accounts - Consolidated) serves as a financial dimension table that maps each transaction in the sales and purchases data to its corresponding general ledger (G/L) account. It allows financial reporting to be segmented by revenue categories, cost categories, or any other financial structure defined by the organization.

This table is critical for aligning sales and purchase activity with financial reporting and margin analysis.

#### Source and Creation Method
- **Source File:** `COA_CONS.xlsx` (Excel)
- **Creation Method:** Loaded via Power Query and cleaned/standardized to align account IDs with transactional tables.

#### (M) Code Explanation – COA_CONS (Chart of Accounts)

The following M code is used to load and transform the chart of accounts data stored in an Excel file (`COA_CONS.xlsx`). This transformation prepares the data for use in financial analysis within the Power BI model.

```powerquery-m
let
    // Step 1: Load the Excel workbook
    Source = Excel.Workbook(File.Contents("V:\\[confidential].xlsx"), null, true),

    // Step 2: Access the specific sheet named 'COA_CONS'
    COA_CONS_Sheet = Source{[Item="COA_CONS",Kind="Sheet"]}[Data],

    // Step 3: Promote the first row as column headers
    #"Promoted Headers" = Table.PromoteHeaders(COA_CONS_Sheet, [PromoteAllScalars=true]),

    // Step 4: Apply proper data types to each column
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {
        {"Account ID", Int64.Type},
        {"Account Description", type text},
        {"Account Type", type text}
    }),

    // Step 5: Rename columns to match semantic naming standards
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type", {
        {"Account Description", "G/L Account Description"},
        {"Account Type", "G/L Account Type"}
    })
in
    #"Renamed Columns"
```

#### Detailed Column Descriptions

| Column Name             | Data Type | Description                                              | Example               |
|-------------------------|-----------|----------------------------------------------------------|-----------------------|
| `Account ID`            | Text      | Unique identifier of the G/L account                     | 4000                  |
| `G/L Account Description` | Text    | Human-readable description of the account                | Sales Revenue         |
| `G/L Account Type`      | Text      | High-level classification (e.g., Revenue, Expense, COGS) | Revenue               |

#### Usage and Analytical Value
- **Financial Reporting:** Enables grouping of transactions into financial categories for reporting.
- **Margin Analysis:** Links directly to Sales and Purchase tables for cost and revenue comparison.
- **Filtering and Aggregation:** Used in slicers and filters to control financial views by account group or type.

#### Relationships (Brief Overview)
- **Linked to the Sales Table** via `G/L Account`
- **Linked to the Purchases Table** via `G/L Account`
- **Linked to the PO Table** for aligning purchase commitments with accounting structure

This table allows the business to report on financial results by ledger classification, enabling cleaner income statements, cost of goods sold analysis, and profitability segmentation.

---


### Sales Representatives Table (`Reps`)

#### Purpose
The **Sales Representatives Table** tracks the individuals responsible for handling customer accounts and executing sales transactions. It plays a critical role in rep-level performance monitoring, accountability, and segmenting sales by division, territory, or internal structure.

This table supports both internal and external reporting use cases through a combination of real and anonymized identifiers.


#### Source and Creation Method

- **Dynamic Base Table:** Created via **DAX** using distinct rep IDs found in the Sales and SO tables.
- **Metadata Mapping:** Enriched with external metadata (e.g., anonymized ID and division) using Power Query from `ISC_Reps.xlsx`.


#### DAX Table Definition

The table is dynamically generated to include any rep ID present in the system, even if metadata hasn’t been defined yet:

```DAX
Sales_Reps = 
DISTINCT(
    UNION(
        SELECTCOLUMNS(Sales_SUS, "Rep_ID", Sales_SUS[Sales Representative ID]),
        SELECTCOLUMNS(SO_SUS, "X", SO_SUS[Sales Representative ID])
    )
)
```

This ensures:
- Newly added sales reps automatically appear in reports.
- Data integrity is maintained even if metadata is missing.
- Reduces maintenance compared to static tables.

---

#### Power Query Enrichment

Once the DAX table exists, rep metadata is added using Power Query from the Excel file `ISC_Reps.xlsx`.

```powerquery-m
let
    // Step 1: Load the Excel workbook
    Source = Excel.Workbook(File.Contents("V:\LIT\ISC\Sage\Customer\ISC_Reps.xlsx"), null, true),

    // Step 2: Access the worksheet (Sheet1)
    Sheet1_Sheet = Source{[Item="Sheet1", Kind="Sheet"]}[Data],

    // Step 3: Temporarily cast columns to text
    #"Changed Type" = Table.TransformColumnTypes(Sheet1_Sheet, {
        {"Column1", type text},
        {"Column2", type text},
        {"Column3", type text}
    }),

    // Step 4: Promote the first row to headers
    #"Promoted Headers" = Table.PromoteHeaders(#"Changed Type", [PromoteAllScalars=true]),

    // Step 5: Assign correct data types
    #"Changed Type1" = Table.TransformColumnTypes(#"Promoted Headers", {
        {"Rep_ID", type text},
        {"Rep_ID_Anonym", type text},
        {"Division", type text}
    })
in
    #"Changed Type1"
```


#### Column Descriptions

| Column Name      | Data Type | Description                                                       | Example     |
|------------------|-----------|-------------------------------------------------------------------|-------------|
| `Rep_ID`         | Text      | Unique internal identifier of the sales representative            | REP014      |
| `Rep_ID_Anonym`  | Text      | Anonymized ID used in public dashboards                           | R_ANON001   |
| `Division`       | Text      | Department or region the rep is assigned to                       | US Midwest  |


#### Usage and Analytical Value

- **Performance Dashboards:** Track revenue, profit, or sales quantity by rep.
- **Organizational Reporting:** View KPIs by rep division.
- **Data Governance:** Anonymized data ensures compliance and sharing flexibility.
- **Dynamic Updates:** No manual updates required when new reps are added to source systems.


#### Relationships Overview

- **Connected to Sales Table** via `Sales Representative ID`
- **May be joined to Customer Table** for segmenting customers by rep
- **Used in filters, slicers, and role-based visuals**

---
