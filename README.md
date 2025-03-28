# Unified Power BI Semantic Model: Advanced Business Analytics with DAX & M

## Table of Contents

- [The Need for a Comprehensive Data Model](#the-need-for-comprehensive-data-model)

- [Tools](#tools)

- [Tables](#tables)
  - [Date Table](#date-table)

- [Regression](#regression)

- [Pricing](#pricing)

- [Results & Impact](#results--impact)

- [Next Steps](#next-steps)


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
## Table of Contents

- [Heading 1](#heading-1)
  - [Subheading 1.1](#subheading-11)
  - [Subheading 1.2](#subheading-12)
    - [Sub-subheading 1.2.1](#sub-subheading-121)
- [Heading 2](#heading-2)
  - [Subheading 2.1](#subheading-21)
- [Heading 3](#heading-3)

## Heading 1

Some text here.

### Subheading 1.1

More text.

### Subheading 1.2

More text here.

#### Sub-subheading 1.2.1

Detailed text here.

## Heading 2

Another section.

### Subheading 2.1

Additional text here.

## Heading 3

Final section.

