# dbt_metrics
<!--This table of contents is automatically generated. Any manual changes between the ts and te tags will be overridden!-->
<!--ts-->
* [dbt_metrics](#dbt_metrics)
* [About](#about)
   * [A note on refs](#warning-a-note-on-refs)
* [Usage](#usage)
* [Secondary calculations](#secondary-calculations)
   * [Period over Period (<a href="/macros/secondary_calculations/secondary_calculation_period_over_period.sql">source</a>)](#period-over-period-source)
   * [Period to Date (<a href="/macros/secondary_calculations/secondary_calculation_period_to_date.sql">source</a>)](#period-to-date-source)
   * [Rolling (<a href="/macros/secondary_calculations/secondary_calculation_rolling.sql">source</a>)](#rolling-source)
* [Customisation](#customisation)
   * [Calendar](#calendar)
   * [Time Grains](#time-grains)
   * [Custom aggregations](#custom-aggregations)
   * [Secondary calculation column aliases](#secondary-calculation-column-aliases)
* [Experimental behaviour](#-experimental-behaviour)
   * [Dimensions on calendar tables](#dimensions-on-calendar-tables)

<!-- Added by: runner, at: Wed Feb  9 05:28:08 UTC 2022 -->

<!--te-->

# About
This dbt package generates queries based on [metrics](https://docs.getdbt.com/docs/building-a-dbt-project/metrics), introduced to dbt Core in v1.0. 

## :warning: A note on `ref`s
To enable the dynamic referencing of models necessary for macro queries through the dbt Server, queries generated by this package do not participate in the DAG and `ref`'d nodes will not necessarily be built before they are accessed. Refer to the docs on [forcing dependencies](https://docs.getdbt.com/reference/dbt-jinja-functions/ref#forcing-dependencies) for more details.

# Usage
Access metrics [like any other macro](https://docs.getdbt.com/docs/building-a-dbt-project/jinja-macros#using-a-macro-from-a-package): 
```sql
select * 
from {{ metrics.metric(
    metric_name='new_customers',
    grain='week',
    dimensions=['plan', 'country'],
    secondary_calculations=[
        period_over_period(comparison_strategy="ratio", interval=1, alias="pop_1wk"),
        period_over_period(comparison_strategy="difference", interval=1),

        period_to_date(aggregate="average", period="month", alias="this_month_average"),
        period_to_date(aggregate="sum", period="year"),

        rolling(aggregate="average", interval=4, alias="avg_past_4wks"),
        rolling(aggregate="min", interval=4)
    ]
) }}
```

# Secondary calculations
Secondary calculations are window functions which act on the primary metric. You can use them to compare a metric's value to an earlier period and calculate year-to-date sums or rolling averages.

Calculations are passed into a list of dictionary elements which can be created manually:
```python
[
    {"calculation": "period_over_period", "interval": 1, "comparison_strategy": "difference", "alias": "pop_1mth"},
    {"calculation": "rolling", "interval": 3, "aggregate": "sum"}
]
```
or by using the convenience [constructor](https://en.wikipedia.org/wiki/Constructor_(object-oriented_programming)) macros.

Column aliases are [automatically generated](#secondary-calculation-column-aliases), but you can override them by setting `alias`. 

## Period over Period ([source](/macros/secondary_calculations/secondary_calculation_period_over_period.sql))

Constructor: `period_over_period(comparison_strategy, interval [, alias])`

- `comparison_strategy`: How to calculate the delta between the two periods. One of [`"ratio"`, `"difference"`]. Required
- `interval`: The number of periods to look back. Required
- `alias`: The column alias for the resulting calculation. Optional

## Period to Date ([source](/macros/secondary_calculations/secondary_calculation_period_to_date.sql))

Constructor: `period_to_date(aggregate, period [, alias])`

- `aggregate`: The aggregation to use in the window function. Options vary based on the primary aggregation and are enforced in [validate_aggregate_coherence()](/macros/secondary_calculations/validate_aggregate_coherence.sql). Required
- `period`: The time grain to aggregate to. One of [`"day"`, `"week"`, `"month"`, `"quarter"`, `"year"`]. Must be at equal or lesser granularity than the metric's grain (see [Time Grains](#time-grains) below). Required
- `alias`: The column alias for the resulting calculation. Optional

## Rolling ([source](/macros/secondary_calculations/secondary_calculation_rolling.sql))

Constructor: `rolling(aggregate, interval [, alias])`

- `aggregate`: The aggregation to use in the window function. Options vary based on the primary aggregation and are enforced in [validate_aggregate_coherence()](/macros/secondary_calculations/validate_aggregate_coherence.sql). Required
- `interval`: The number of periods to look back. Required
- `alias`: The column alias for the resulting calculation. Optional


# Customisation
Most behaviour in the package can be overridden or customised.

## Calendar 
The package comes with a [basic calendar table](/models/dbt_metrics_default_calendar.sql), running between 2010-01-01 and 2029-12-31 inclusive. You can replace it with any custom calendar table which meets the following requirements:
- Contains a `date_day` column. 
- Contains the following columns: `date_week`, `date_month`, `date_quarter`, `date_year`, or equivalents. 
- Additional date columns need to be prefixed with `date_`, e.g. `date_4_5_4_month` for a 4-5-4 retail calendar date set. Dimensions can have any name (see [dimensions on calendar tables](#dimensions-on-calendar-tables)).

To do this, set the value of the `dbt_metrics_calendar_model` variable in your `dbt_project.yml` file: 
```yaml
#dbt_project.yml
config-version: 2
[...]
vars:
    dbt_metrics_calendar_model: ref('my_custom_table')
```

## Time Grains 
The package protects against nonsensical secondary calculations, such as a month-to-date aggregate of data which has been rolled up to the quarter. If you customise your calendar (for example by adding a [4-5-4 retail calendar](https://nrf.com/resources/4-5-4-calendar) month), you will need to override the [`get_grain_order()`](/macros/secondary_calculations/validate_grain_order.sql) macro. In that case, you might remove `month` and replace it with `month_4_5_4`. All date columns must be prefixed with `date_` in the table. Do not include the prefix when defining your metric, it will be added automatically.

## Custom aggregations 
To create a custom primary aggregation (as exposed through the `type` config of a metric), create a macro of the form `metric_my_aggregate(expression)`, then override the [`aggregate_primary_metric()`](/macros/aggregate_primary_metric.sql) macro to add it to the dispatch list. The package also protects against nonsensical secondary calculations such as an average of an average; you will need to override the [`get_metric_allowlist()`](/macros/secondary_calculations/validate_aggregate_coherence.sql)  macro to both add your new aggregate to to the existing aggregations' allowlists, and to make an allowlist for your new aggregation:
```
    {% do return ({
        "average": ['max', 'min'],
        "count": ['max', 'min', 'average', 'my_new_aggregate'],
        [...]
        "my_new_aggregate": ['max', 'min', 'sum', 'average', 'my_new_aggregate']
    }) %}
```

To create a custom secondary aggregation (as exposed through the `secondary_calculations` parameter in the `metric` macro), create a macro of the form `secondary_calculation_my_calculation(metric_name, dimensions, calc_config)`, then override the [`perform_secondary_calculations()`](/macros/secondary_calculations/perform_secondary_calculation.sql) macro. 

## Secondary calculation column aliases
Aliases can be set for a secondary calculation. If no alias is provided, one will be automatically generated. To modify the existing alias logic, or add support for a custom secondary calculation, override [`generate_secondary_calculation_alias()`](/macros/secondary_calculations/generate_secondary_calculation_alias.sql).

# 🧪 Experimental behaviour 
:warning: This behaviour is subject to change in future versions of dbt Core and this package.

## Dimensions on calendar tables
You may want to aggregate metrics by a dimension in your custom calendar table, for example `is_weekend`. _In addition to_ the primary `dimensions` list, add the following `meta` properties to your metric:
```yaml
version: 2 
metrics:
  - name: new_customers
    [...]
    dimensions: 
      - plan
      - country

    meta: 
      dimensions:  
        - type: model
          columns:
            - plan
            - country
        - type: calendar
          columns: 
            - is_weekend
```

You can then access the additional dimensions as normal:
```sql
select * 
from {{ metrics.metric(
    metric_name='new_customers',
    grain='week',
    dimensions=['plan', 'country', 'is_weekend'],
    secondary_calcs=[]
) }}
```