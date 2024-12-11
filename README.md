# DAX

## 1. Calendar and Time Inteligence

### 1.1 Calendar table 1
```
Calendar = 
ADDCOLUMNS (
CALENDAR (DATE(2020,1,1), DATE(2022,12,31)),
"DateAsInteger", FORMAT ( [Date], "YYYYMMDD" ),
"Date_text", FORMAT([Date],"DD/MM/YYYY"),
"DateISO", FORMAT ( [Date], "YYYY-MM-DD" ),
"Year", YEAR ( [Date] ),
"Monthnumber", FORMAT ( [Date], "MM" ),
"YearMonthnumber", FORMAT ( [Date], "YYYY/MM" ),
"YearMonthnumbershort", FORMAT ( [Date], "YYYYMM" ),
"YearMonthShort", FORMAT ( [Date], "YYYY/mmm" ),
"YearMonthShort2", FORMAT ( [Date], "YY/mmm" ),
"MonthNameShort", FORMAT ( [Date], "mmm" ),
"YearMonthNameLong",FORMAT ( [Date], "YYYY" )&"/"&FORMAT ( [Date], "mmmm" ),
"MonthNameLong", FORMAT ( [Date], "mmmm" ),
"DayOfWeekNumber", WEEKDAY ( [Date] ),
"DayOfWeek", FORMAT ( [Date], "dddd" ),
"DayOfWeekShort", FORMAT ( [Date], "ddd" ),
"Quarter", "Q" & FORMAT ( [Date], "Q" ),
"YearQuarter", FORMAT ( [Date], "YYYY" ) & "/Q" & FORMAT ( [Date], "Q" )
)
```
### 1.2 Calendar Table 2 

```
Date = 
VAR minDate = MINX(Appned_table,Appned_table[EndBranchprice.Date])
VAR maxDate = MAXX(Appned_table,Appned_table[EndBranchprice.Date])
RETURN
ADDCOLUMNS(
    CALENDAR(minDate,maxDate),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMMM"),
    "Month number", MONTH([Date]),
    "Calendar quarter", CONCATENATE("Q",QUARTER([Date])),
    "Calendar week", WEEKNUM([Date],2),
    "Day number", WEEKDAY([Date],2),
    "Day", FORMAT([Date],"DDD"),
    "Fiscal Year", SWITCH(
        TRUE,
        MONTH([Date])>=10,
        YEAR([Date])+1,
        YEAR([Date])),
    "Fiscal Quarter", SWITCH(
        TRUE,
        QUARTER([Date]) = 1, "Q2",
        QUARTER([Date]) = 2, "Q3",
        QUARTER([Date]) = 3, "Q4",
        "Q1")
        )
```

### 1.3 DateINTkey

```
DateINTkey = 
FORMAT ( [Date], "YYYYMMDD" )
```

### 1.4 Is Past
```
IsPast =

VAR _today = TODAY()

VAR LastSalesPY = EDATE(_today, -12)

RETURN

    Calendar[Date] <= LastSalesPY
```

### 1.5 Number of a day (1-365/366)
- without leap year
```
DayNoOfYear =
DATEDIFF (
    DATE ( YEAR ( Calendar[Date] ), 1, 1 ),
    Calendar[Date],
    DAY
) + 1
```

### 1.6 Number of a day (1-365)
- with leap year
```
DaysFromStartOfYear =

VAR BaseDate = DATE(YEAR([Date_PK]), 1, 1)

VAR CurrentDate = [Date_PK]

VAR DayOfYear = DATEDIFF(BaseDate, CurrentDate, DAY)+1

VAR IsLeapYear =

    IF(

        MOD(YEAR(CurrentDate), 4) = 0 && (MOD(YEAR(CurrentDate), 100) <> 0 || MOD(YEAR(CurrentDate), 400) = 0),

        TRUE(),

        FALSE()

    )

RETURN

    IF(

        IsLeapYear && MONTH(CurrentDate) > 2,

        DayOfYear - 1,

        DayOfYear

    )
```

### 1.7 YTD Status
```
YTD Status_CALC =

VAR _today_day =

    CALCULATE(

        MAX( Calendar_CALC[DayNoOfYear] ),

        Calendar_CALC[Date_PK] = TODAY( )

    )

VAR _ytd =

    IF(

        Calendar_CALC[DayNoOfYear] <= _today_day,

        "YTD",

        "NO YTD"

    )

RETURN

    _ytd
``` 

## 2. Pareto distribution
### 2.1 RankX
- rankX based on two level structure, prepared for drill down and for hierarchies
- Based on two measures
- If the user wants only one level of structure or one measure, just remove Level1/Level2 and one measure.

```
RankX Industry = 
VAR _industry =
    RANKX(
        ALLSELECTED(
            'TableX'[Level1],
            'TableX'[Level2]
        ),
        ( [Sales received] + [Sales waiting] )
    )
VAR _subindustry =
    RANKX(
        ALLSELECTED(
            'TableX'[Level1],
            'TableX'[Level2]
        ),
        ([Sales received] + [Sales waiting] )
    )
RETURN
    SWITCH(
        TRUE( ),
        ISINSCOPE( 'TableX'[Level1] ), _subindustry,
        ISINSCOPE( 'TableX'[Level2] ), _industry
    )
``` 

### 2.2 Running total %
- be awar of sorting by column. It can make some mess in the RankX, so it is better to not use sorting or put soritng to all selected

```
VAR _rank = [RankX Industry]
VAR _cumulative_industry =
    SUMX(
        FILTER(
            ALLSELECTED(
                'TableX'[Level1],
                'TableX'[Sorting]
            ),
            [RankX Industry] <= _rank
        ),
        ( [Sales received] + [Sales waiting] )
    )
VAR _total_industry =
    CALCULATE(
        SUMX(
            ALLSELECTED(
                ''TableX'[Level1],
                'TableX'[Sorting]
            ),
            ( [Sales received] + [Sales waiting] )
        )
    )
VAR _cumulative_sub =
    SUMX(
        FILTER(
            ALLSELECTED(
                'TableX'[Level2],
                'TableX'[Sorting]
            ),
            [RankX Industry] <= _rank
        ),
        ( [Sales received] + [Sales waiting] )
    )
VAR _total_sub =
    CALCULATE(
        SUMX(
            ALLSELECTED(
                'TableX'[Level2],
                'TableX'[Sorting]
            ),
            ( [Sales received] + [Sales waiting] )
        )
    )
VAR _industry = DIVIDE( _cumulative_industry, _total_industry)
VAR _sub = DIVIDE( _cumulative_sub, _total_sub )
RETURN
    SWITCH(
        TRUE( ),
        ISINSCOPE( 'TableX'[Level2] ), _sub,
        ISINSCOPE( 'TableX'[Level1] ), _industry
    )

```

## 2.3 % share by Some category

```
VAR _orders = [Sales received] + [Sales waiting]
VAR _total_industry =
    CALCULATE(
        SUMX(
            ALLSELECTED(
                'TableX'[Level1],
                'TableX'[Sorting]
            ),
            [Sales received] + [Sales waiting]
        )
    )
VAR _total_sub =
    CALCULATE(
        SUMX(
            ALLSELECTED(
                'TableX'[Level2],
                'TableX'[Sorting]
            ),
            [Sales received] + [Sales waiting]
        )
    )
VAR _industry = DIVIDE( _orders, _total_industry )
VAR _sub = DIVIDE( _orders, _total_sub )
RETURN
    SWITCH(
        TRUE( ),
        ISINSCOPE( 'TableX'[Level2] ), _sub,
        ISINSCOPE( 'TableX'[Level1] ), _industry
    )


```