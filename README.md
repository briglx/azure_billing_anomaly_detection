# Azure Billing Anomaly Detection

This project demonstrates how to find daily changes in Azure billing data using PowerBI

![Delta Daily Costs](docs/dailycostsdelta.jpg) "Delta Daily Costs")
dailycostsdelta

Workflow:

- Aggregate daily instance usage by resourge group 
- Create new Columns to calculate difference between daily costs.

The sample data is formated with tfh following columns.

|date                | resource group | instance id   | cost | 
|--------------------|----------------|---------------|------|
|2021-06-28 00:00:00 | RG1            | RG1/instance1 | 2.00 |
|2021-06-28 00:00:00 | RG1            | RG1/instance2 | 4.00 |
|2021-06-28 00:00:00 | RG1            | RG1/instance3 | 2.00 |
|...                 | ...            | ...           | ...  |
|2021-06-28 00:00:00 | RG2            | RG2/instance1 | 3.00 |
|2021-06-28 00:00:00 | RG2            | RG2/instance1 | 5.00 |
|...                 | ...            | ...           | ...  |

The goal is to end up with the following

|date                | resource group | cost  | prevous cost | daily delta | daily precent change |  
|--------------------|----------------|-------|--------------|-------------|----------------------|
|2021-06-28 00:00:00 | RG1            | 42.00 | 32.00        | -10.00      | -.27                 |
|2021-06-28 00:00:00 | RG2            | 56.00 | 78.00        | 22.00       | .3284                |
|...                 | ...            | ...   | ...          | ...         | ...                  |


## Aggregate Daily Instance Usage

Use Power Query Editor to aggregate the data.

```
# Table.Group example
Table.Group(table as table, key as any, aggregatedColumns as list, optional groupKind as nullable number, optional comparer as nullable function) as table 

#"aggregate table" = Table.Group(#"original table", {"date", "resource group"}, {{"cost", each List.Sum([cost]), type nullable number}})
```

## Create new Columns

With the aggregated data, create new columns with DAX queries.

```
# Current Row
Current Row = RANKX( ALL('usage'), 'usage'[date], , ASC, Dense)

# Previous Row
Previous Row = 
VAR CurrentRow = 'usage'[Current Row]
VAR PreviousRow = CALCULATE(
    MAX('usage'[Current Row]), 
    ALL('usage'),
    'usage'[resource group] = EARLIER('usage'[resource group]),
    'usage'[date] < EARLIER('usage'[date])
)
RETURN IF( PreviousRow <> BLANK(), PreviousRow, CurrentRow )

# Previous Cost
Previous Cost = 
CALCULATE(
    MAX('usage'[cost]), 
    ALL('usage'), 
    'usage'[resource group] = EARLIER('usage'[resource group]), 
    'usage'[Current Row] = EARLIER('usage'[Previous Row])
)

# Daily Delta Cost
Daily Delta Cost = usage[cost] - usage[Previous Cost] 

# Daily Cost Precent Difference
DailyCostPrecentDiff = if(usage[Previous Cost] >0, (usage[cost] - usage[Previous Cost]  ) / ((usage[Previous Cost] + usage[cost]) / 2), 0)

```

# Setup

This setup will create a Power BI workbook that uses the sample data.

- Open the PowerBI template `/project/path/docs/AzureBillingAnomalyDetectionReport.pbit`
- Select the sample data set `/project/path/docs/usage.csv`

# References

- https://stackoverflow.com/questions/69074965/how-to-get-previous-row-value-in-dax-power-bi
- Power M Language https://docs.microsoft.com/en-us/powerquery-m/
- Table Group https://docs.microsoft.com/en-us/powerquery-m/table-group
- RankX function https://docs.microsoft.com/en-us/dax/rankx-function-dax