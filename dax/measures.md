# DAX measures — showcase

The model has 150+ measures organized into display folders (Summary, Efficiency, Agents, Topics, Drillthrough). This file walks through the most representative ones rather than listing all 150 — full definitions are in the `.pbix`/TMDL source.

---

## Core KPIs

```dax
Total Calls = COUNTROWS(fact_callcenter)

Answered Calls = CALCULATE([Total Calls], fact_callcenter[IsAnswered] = 1)

Resolved Calls = CALCULATE([Total Calls], fact_callcenter[IsResolved] = 1)

FCR % =
DIVIDE(
    CALCULATE([Total Calls], fact_callcenter[IsResolved] = 1, fact_callcenter[IsAnswered] = 1),
    [Answered Calls],
    0
)

CSAT Avg = AVERAGE(fact_callcenter[Satisfaction rating])

AHT (Average Handle Time) = DIVIDE(SUM(fact_callcenter[TalkSeconds]), [Answered Calls] * 60, 0)
```

These all follow the same defensive pattern — `DIVIDE()` with an explicit zero-fallback — so no measure throws a divide-by-zero error when a slicer selection returns an empty answered-calls set.

---

## Time-intelligence: the period-comparison engine

The Summary page's MoM/QoQ/YoY toggle is driven by a single disconnected table (`Period Selector`) rather than three separate sets of visuals. Every "current vs. previous period" KPI card reads `SELECTEDVALUE('Period Selector'[Period])` and branches accordingly:

```dax
Calls_Curr_Total_Value =
VAR _Period = SELECTEDVALUE('Period Selector'[Period])
VAR _MaxDate = MAX(dim_calendar[Date])
RETURN
    SWITCH(
        TRUE(),
        _Period = "MoM", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], _MaxDate, -1, MONTH)),
        _Period = "QoQ", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], _MaxDate, -1, QUARTER)),
        _Period = "YoY", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], _MaxDate, -1, YEAR))
    )

Calls_Prev_Total_Value =
VAR _Period = SELECTEDVALUE('Period Selector'[Period])
VAR _MaxDate = MAX(dim_calendar[Date])
RETURN
    SWITCH(
        TRUE(),
        _Period = "MoM", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], EDATE(_MaxDate, -1), -1, MONTH)),
        _Period = "QoQ", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], EDATE(_MaxDate, -3), -1, QUARTER)),
        _Period = "YoY", CALCULATE([Total Calls], DATESINPERIOD(dim_calendar[Date], EDATE(_MaxDate, -12), -1, YEAR))
    )

Calls_Growth_% = DIVIDE([Calls_Curr_Total_Value] - [Calls_Prev_Total_Value], [Calls_Prev_Total_Value], 0)
```

This same three-measure pattern (`_Curr_`, `_Prev_`, `_Growth_%`) repeats for eight KPIs: Calls, AHT, Answered, Resolved, Abandoned, CSAT, FCR, and Utilization. It's the largest single block of DAX in the model — and also the most repetitive. A more DRY version would pass the base measure as a parameter into one shared function rather than duplicating the `SWITCH`/`DATESINPERIOD` scaffold eight times; that's the explicit tradeoff documented in the main README.

---

## Workforce efficiency

```dax
ASA Seconds = AVERAGE(fact_callcenter[Speed of answer in seconds])

ASA SLA % =
DIVIDE(
    CALCULATE([Total Calls], fact_callcenter[Speed of answer in seconds] <= 60, fact_callcenter[IsAnswered] = 1),
    [Answered Calls],
    0
)

Occupancy Est % = DIVIDE(SUM(fact_callcenter[TalkSeconds]), [Available Seconds Est], 0)

zShrinkage Est % = 1 - [Occupancy Est %]
```

`_ASAInsight` generates the plain-language sentence shown on the Efficiency page ("ASA performance is within target"). The comparison direction matters here — ASA is a wait time, so *lower* is better, which the final logic reflects.

---

## SVG-rendered visuals

Several KPI elements on the dashboard — the CSAT/FCR/Resolution rings, the star-category labels, the topic bar chart, and the shift-performance bars — are not native Power BI visuals. They're single measures that return a `data:image/svg+xml` string, displayed via an Image-type card.

```dax
'SVG CSAT Ring' =
VAR _csat = [CSAT Avg]
VAR _pct = DIVIDE(_csat, 5, 0)
VAR _angle = _pct * 360
VAR _color = SWITCH(TRUE(), _csat >= 4, "#1D9E75", _csat >= 3, "#EF9F27", "#E24B4A")
RETURN
    "data:image/svg+xml;utf8," &
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'>" &
        "<circle cx='50' cy='50' r='40' stroke='#E5E5E5' stroke-width='10' fill='none' />" &
        "<circle cx='50' cy='50' r='40' stroke='" & _color & "' stroke-width='10' fill='none' " &
            "stroke-dasharray='" & (_pct * 251.2) & " 251.2' transform='rotate(-90 50 50)' />" &
    "</svg>"
```

```dax
'SVG Topic Bar Chart' =
VAR _Topics = VALUES(dim_topics[Topic])
VAR _MaxCalls = MAXX(_Topics, CALCULATE([Total Calls]))
RETURN
    "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 400 200'>" &
    CONCATENATEX(
        _Topics,
        VAR _calls = CALCULATE([Total Calls])
        VAR _width = DIVIDE(_calls, _MaxCalls, 0) * 350
        RETURN "<rect x='10' width='" & _width & "' height='18' fill='#378ADD' />",
        "",
        [Total Calls], DESC
    ) &
    "</svg>"
```

Building visuals this way trades native Power BI conditional-formatting convenience for full control over shape, color, and layout — and it's what makes the rings and bars on the Summary and Topics pages look custom rather than default.

---

## Drillthrough and decomposition

```dax
'Abandon Calls Top Agent' =
TOPN(1, VALUES(dim_agents[Agent]), CALCULATE([Abandon Calls Count]), DESC)

D_Abandon_Drillthrough_Label =
"Top abandon topic: " & [Worst Abandon Topic] & " — handled most by " & [Abandon Calls Top Agent]
```

The Abandon Calls Analysis page's decomposition tree and summary table both read from these label-building measures, so the drillthrough updates its narrative text automatically as the topic/agent filters change — not just the numbers.
