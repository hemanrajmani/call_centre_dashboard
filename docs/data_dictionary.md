# Data dictionary

Column-level reference for every table in the model. Grain, types, and relationships reflect the final TMDL model.

---

## fact_callcenter
**Grain:** one row per call.

| Column | Type | Description |
|---|---|---|
| Date | Date | Calendar date of the call. Joins to `dim_calendar[Date]`. |
| Agent | Text | Agent who handled the call. Joins to `dim_agents[Agent]`. |
| Topic | Text | Call category. Joins to `dim_topics[Topic]`. |
| TimeId | Whole number | Hour of day (0–23) the call started. Joins to `dim_time[Value]`. |
| Satisfaction rating | Whole number | 1–5 customer satisfaction score. Joins to `dim_ratings[Value]`. |
| IsAnswered | Whole number (0/1) | Pre-computed in Power Query — 1 if the call was answered, 0 if abandoned. |
| IsResolved | Whole number (0/1) | Pre-computed in Power Query — 1 if the call was resolved on first contact. |
| TalkSeconds | Whole number | Talk duration in seconds. |
| Speed of answer in seconds | Whole number | Time from call entering queue to being answered. |
| Talk_Time_SLA_Bucket | Text | SLA bucket label (e.g., Excellent / Average / Poor). Sourced into `dim_Talk_SLA`. |

## dim_calendar
Standard date dimension, one row per calendar day across the 4-year range.

| Column | Type | Description |
|---|---|---|
| Date | Date | Primary key. |
| Year | Whole number | Calendar year. |
| Quarter | Text | Q1–Q4 label. |
| Month | Text | Month name. |
| MonthNo | Whole number | Month number, used for sorting. |
| Weekday_Name | Text | Day of week name, used in the heatmap. |

## dim_agents
One row per agent.

| Column | Type | Description |
|---|---|---|
| Agent | Text | Agent name. Primary key. |
| Agent_Key | Whole number | Surrogate key (not currently used in any relationship). |

## dim_topics
One row per call topic.

| Column | Type | Description |
|---|---|---|
| Topic | Text | Topic name. Primary key. |
| Topic_Key | Whole number | Surrogate key (not currently used in any relationship). |
| Topic Abbreviation | Text | Calculated column — short form used in scatter chart legends. |

## dim_time
One row per hour of day, generated via `GENERATESERIES`.

| Column | Type | Description |
|---|---|---|
| Value | Whole number | Hour, 0–23. Primary key. |
| Hour | Text | Display label (e.g., "08:00"). |
| Shift | Text | Morning / Evening / Night classification. |

## dim_ratings
One row per star rating, generated via `GENERATESERIES`.

| Column | Type | Description |
|---|---|---|
| Value | Whole number | 1–5. Primary key. |
| Star_Value | Text | Display column — number plus a star glyph (e.g., "4★"). |

## dim_Talk_SLA
Distinct list of SLA bucket labels, sourced from `fact_callcenter`. Currently disconnected — no relationship back to the fact table.

| Column | Type | Description |
|---|---|---|
| SLA | Text | SLA bucket label. |

## Period Selector
Disconnected table used purely as a slicer source.

| Column | Type | Description |
|---|---|---|
| Period | Text | One of "MoM", "QoQ", "YoY". |

## last_refresh
Single-row utility table.

| Column | Type | Description |
|---|---|---|
| Data_Last_Refreshed | Text | Timestamp captured at refresh time via `DateTime.LocalNow()`. |
