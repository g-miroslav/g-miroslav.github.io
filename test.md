## Introduction
The Power BI report and Power Query M code demonstrates how to split time data into days and shifts. In the example provided, the data is split into 3 shifts and each day starts at 6am. This ensures that data, such as duration between start and end times, is attributed accurately to each day and shift.

## Data
### schema
| Column        | Data type     |
| ------------- | ------------- |
| DEVICE_ID     | text          |
| STATUS_ID     | text          |
| tStart        | datetime      |
| tEnd          | datetime      |

### sample data
| DEVICE_ID     | STATUS_ID     | tStart        | tEnd          |
| ------------- | ------------- | ------------- | ------------- |
| 07            | 13            | 2023-02-13 06:00:00 | 2023-02-18 06:00:00 |
| 24            | 03            | 2023-02-14 07:57:23 | 2023-02-14 21:40:21 |
| 24            | 04            | 2023-02-14 07:16:27 | 2023-02-14 07:57:23 |
| 24            | 26            | 2023-02-15 05:07:23 | 2023-02-15 06:07:37 |
| 24            | 07            | 2023-02-14 01:06:31 | 2023-02-14 07:16:27 |
    
## Power Query M Code
[Power Query M Code](Power_Query_M_Code.txt)

<details>
    <summary>Details</summary>

#### 1. Load Data
```m
let
    Source = #"Device Status (Raw Data)",
```
#### 2. Add Total Shift Start
Identify the start of the first shift associated with each event.
```m
#"Added TotalShiftStart" = Table.AddColumn(Source, "TotalShiftStart", each 
    let result =
        if varStartTime >= #time(6, 0, 0) and varStartTime <#time(14, 0, 0) then #duration(0, 6, 0, 0)
        else if varStartTime >= #time(14, 0, 0) and varStartTime < #time(22, 0, 0) then #duration(0, 14, 0, 0)
        else if varStartTime >= #time(22, 0, 0) then #duration (0, 22, 0, 0)
        else if varStartTime < #time(6, 0, 0) then #duration(0, 22, 0, 0) - #duration(1, 0, 0, 0)
        else "error",

        varStartTime = DateTime.Time([tStart])

        in DateTime.From(Date.From([tStart])) + result, type datetime
    ),
```
#### 3. Add Total Shift End
Identify the end of the last shift associated with each event.
```m
#"Added TotalShiftEnd" = Table.AddColumn(#"Added TotalShiftStart", "TotalShiftEnd", each 
    let result =
        if varEndTime >= #time(6, 0, 0) and varEndTime < #time(14, 0, 0) then #duration(0, 14, 0, 0)
        else if varEndTime >= #time(14, 0, 0) and varEndTime < #time(22, 0, 0) then #duration(0, 22, 0, 0)
        else if varEndTime >= #time(22, 0, 0) then #duration(1, 6, 0, 0)
        else if varEndTime < #time(6, 0, 0) then #duration(0, 6, 0, 0)
        else "error",

        varEndTime = DateTime.Time([tEnd])

        in DateTime.From(Date.From([tEnd])) + result, type datetime
    ),
```
#### 4. Split Data into 3 shifts
This step creates a list for each row.
##### List.DateTimes syntax:
```m
List.DateTimes(start as datetime, count as number, step as duration) as list
```
```m
#"Added ShiftStart" = Table.AddColumn(#"Added TotalShiftEnd", "ShiftStart", each 
        List.DateTimes(
            [TotalShiftStart],
            Duration.TotalHours([TotalShiftEnd]-[TotalShiftStart])/8,
            #duration(0, 8, 0, 0)
        )
    ),
```
#### 5. Expand the lists into rows & change to datetime
```m
#"Expanded ShiftStart" = Table.ExpandListColumn(#"Added ShiftStart", "ShiftStart"),
#"Changed ShiftStart Type" = Table.TransformColumnTypes(#"Expanded ShiftStart",{{"ShiftStart", type datetime}}),
```
#### 6. Add Shift End
```m
#"Added ShiftEnd" = Table.AddColumn(#"Changed ShiftStart Type", "ShiftEnd", each
        [ShiftStart] + #duration(0, 8, 0, 0), type datetime
    ),
```
#### 7. Add Start and End
```m
#"Added Start" = Table.AddColumn(#"Added ShiftEnd", "Start", each List.Max({[tStart], [ShiftStart]}), type datetime),
#"Added End" = Table.AddColumn(#"Added Start", "End", each List.Min({[tEnd], [ShiftEnd]}), type datetime),
```
#### 8. Calculate Duration
```m
#"Added Duration" = Table.AddColumn(#"Added End", "Duration", each Duration.TotalHours([End] - [Start]), type number),
```
#### 8. Add Date
Since each starts at 6am, a date column is created from the ShiftStart column. This can be linked to a Calendar table.
```m
#"Inserted Date" = Table.AddColumn(#"Added Duration", "Date", each DateTime.Date([ShiftStart]), type date),
```
#### 9. Add Shift Number
```m
#"Added ShiftNumber" = Table.AddColumn(#"Inserted Date", "ShiftNumber", each 
    let result =
        if varShiftStart = #time(6, 0, 0) then 1
        else if varShiftStart = #time(14, 0, 0) then 2
        else if varShiftStart = #time(22, 0, 0) then 3
        else "error",

        varShiftStart = DateTime.Time([ShiftStart])

        in result, Int64.Type
    )
```
#### 10. End of M code
```
in
    #"Added ShiftNumber"
```
</details>

## T-SQL query
[T-SQL](Split_Timeline.sql)

<details>
    <summary>Details</summary>

#### 1. ShiftBoundaries_CTE
Identify the boudaries of shifts for each event. The `ROW_NUMBER` function creates a unique ID for each row that is used in the following CTE.
```tsql
WITH ShiftBoundaries_CTE as (
SELECT
    ROW_NUMBER() OVER (ORDER BY DEVICE_ID, tStart) as ID 
    , DEVICE_ID
    , STATUS_ID
    , tStart
    , tEnd
    , CASE
        WHEN CAST(tStart as time) >= '06:00' AND CAST(tStart as time) < '14:00' THEN DATEADD(hour, 6, CAST(CAST(tStart as date) as datetime))
        WHEN CAST(tStart as time) >= '14:00' AND CAST(tStart as time) < '22:00' THEN DATEADD(hour, 14, CAST(CAST(tStart as date) as datetime))
        WHEN CAST(tStart as time) >= '22:00' THEN DATEADD(hour, 22, CAST(CAST(tStart as date) as datetime))
        WHEN CAST(tStart as time) < '06:00' THEN DATEADD(hour, 22, CAST(CAST(tStart - 1 as date) as datetime))
    END as TotalShiftStart
    , CASE
        WHEN CAST(tEnd as time) >= '06:00' AND CAST(tEnd as time) < '14:00' THEN DATEADD(hour, 14, CAST(CAST(tEnd as date) as datetime))
        WHEN CAST(tEnd as time) >= '14:00' AND CAST(tEnd as time) < '22:00' THEN DATEADD(hour, 22, CAST(CAST(tEnd as date) as datetime))
        WHEN CAST(tEnd as time) >= '22:00' THEN DATEADD(hour, 6, CAST(CAST(tEnd + 1 as date) as datetime))
        WHEN CAST(tEnd as time) < '06:00' THEN DATEADD(hour, 6, CAST(CAST(tEnd as date) as datetime))
    END as TotalShiftEnd
FROM 
	Timeline
)
```
#### 2. Recursive_CTE
Split each event into individual shifts using a recursive CTE.
```tsql
, Recursive_CTE as (
SELECT
    *
    , TotalShiftStart as ShiftStart
FROM 
    ShiftBoundaries_CTE
UNION ALL
SELECT
    t.ID
    , t.DEVICE_ID
    , t.STATUS_ID
    , t.tStart
    , t.tEnd
    , t.TotalShiftStart
    , t.TotalShiftEnd
    , DATEADD(hour, 8, Recursive_CTE.ShiftStart) as ShiftStart
FROM 
    ShiftBoundaries_CTE as t INNER JOIN Recursive_CTE
        ON t.ID = Recursive_CTE.ID
WHERE
     DATEADD(hour, 8, Recursive_CTE.ShiftStart) < t.TotalShiftEnd
)
```
#### 3. EventStartEnd_CTE
Determine the Start and End datetimes for each event within a shift.
```tsql
, EventStartEnd_CTE as (
SELECT 
    *
    , CASE WHEN tStart > ShiftStart THEN tStart ELSE ShiftStart END as [Start]
    , CASE WHEN tEnd < DATEADD(hour, 8, ShiftStart) THEN tEnd ELSE DATEADD(hour, 8, ShiftStart) END as [End]
FROM
    Recursive_CTE
)
```
#### 4. Final output table
Create duration (hours), date, and shift number for each event.
```tsql
SELECT
    DEVICE_ID
    , STATUS_ID
    , tStart
    , tEnd
    , TotalShiftStart
    , TotalShiftEnd
    , DATEDIFF(second, [Start], [End])/60.0/60.0 as Duration
    , CAST(ShiftStart as date) as [Date]
    , CASE
        WHEN CAST(ShiftStart as time) = '06:00' THEN 1
        WHEN CAST(ShiftStart as time) = '14:00' THEN 2
        WHEN CAST(ShiftStart as time) = '22:00' THEN 3
    END as [Shift]
FROM
    EventStartEnd_CTE
```
</details>

## Qlik Sense script
[Qlik Sense script](Qlik_Sense_script.txt)

<details>
    <summary>Details</summary>

Qlik Sense script was adapted from an answer by **Swuehl** in the Qlik Community forum - [Splitting the data - On the way to granularity](https://community.qlik.com/t5/QlikView-App-Dev/Splitting-the-data-On-the-way-to-granularity/td-p/468139)
	
#### 1. Load Data
Identify the boudaries of shifts for each event.
```
Input:
LOAD
    DEVICE_ID,
    STATUS_ID,
    tStart,
    tEnd
 FROM [lib://C/Sample Data.csv]
(txt, codepage is 28591, embedded labels, delimiter is ',', msq);
```
#### 2. Identify Shift boundaries
Identify the boudaries of shifts for each event.
```
ShiftBoundaries:
LOAD
    *,
    Timestamp(Floor(tStart, MakeTime(8), MakeTime(6))) as TotalShiftStart,
    Timestamp(Floor(tEnd, MakeTime(8), MakeTime(6)) + MakeTime(8)) as TotalShiftEnd
Resident Input;
```
#### 3. Split into 3 shifts
`WHILE` statement is used to split events into 3 shifts.
```
LOAD
    *,
    Timestamp(TotalShiftStart + (IterNo() - 1) * MakeTime(8)) as ShiftStart,
    Timestamp(TotalShiftStart + IterNo() * MakeTime(8)) as ShiftEnd
Resident ShiftBoundaries
WHILE TotalShiftStart + IterNo() * MakeTime(8) <= TotalShiftEnd;
```
#### 4. Start and End
Determine the Start and End datetimes for each event within a shift.
```
LOAD
    *,
    If(tStart > ShiftStart, tStart, ShiftStart) as Start,
    If(tEnd < ShiftEnd, tEnd, ShiftEnd) as End;
```
#### 5. Duration, Shift, Date
Create duration (hours), date, and shift number for each event.
```
LOAD
    *,
    Interval(End - Start) * 24 as Duration,
    Pick(Match(Time(Frac(ShiftStart)), MakeTime(6), MakeTime(14), MakeTime(22)), 1, 2, 3) as Shift,
    Date(Floor(ShiftStart)) as Date;
```
#### 6. Remove redundant tables
```
DROP TABLE Input;
DROP TABLE ShiftBoundaries;
```
</details>

### Credits
- **Antti Suanto** - `as Timeline` 1.5.1 custom visual for Power BI
- **Swuehl** - [Qlik Community - Splitting the data - On the way to granularity](https://community.qlik.com/t5/QlikView-App-Dev/Splitting-the-data-On-the-way-to-granularity/td-p/468139)
