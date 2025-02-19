# `dataRetrieval` user package testing


## Introduction

This script is part of user testing for the United States Geological
Survey [dataRetrieval](https://doi-usgs.github.io/dataRetrieval/) R
package. As part of user testing, users are asked to conduct an analysis
that addresses this prompt:

*You are working on an annual report of discharge and water temperature
at the* *Milwaukee River near Cedarburg, Wisconsin, site
\#USGS-05340500. Your tasks* *are: a) download the daily data for stream
discharge and water temperature* *between Oct 1, 2022 and Sept 30, 2023;
b) create time-series plots of the* *data, one for each variable.*

``` r
site <- "05340500"

start_date <- lubridate::as_date(x = "2022-10-01")
end_date <- lubridate::as_date(x = "2023-09-30")
```

## Data exploration

I started by exploring the parameter and statistic codes that are
relevant to this prompt. To retrieve relevant parameter codes, I:

- visited `readNWISdv()` [function
  documentation](https://doi-usgs.github.io/dataRetrieval/reference/readNWISdv.html)
- clicked the link provided for [daily value
  documentation](https://waterservices.usgs.gov/docs/dv-service/daily-values-service-details)
  and navigated to the section that addresses parameter codes
- clicked the link provided for the [parameter codes
  list](https://help.waterdata.usgs.gov/codes-and-parameters/parameters)
- clicked the link provided for the [html page of physical parameter
  codes](https://help.waterdata.usgs.gov/parameter_code?group_cd=PHY)

I then extracted a list of all parameter codes associated with discharge
and water temperature that I might use to answer this prompt.

``` r
parameter_codes <- tibble::tribble(
  ~parameter_code, ~description,
  "00010",         "Temperature, water, degrees Celsius",
  "00011",         "Temperature, water, degrees Fahrenheit",
  "00060",         "Discharge, cubic feet per second, mean, 1 day",
  "00061",         "Discharge, instantaneous, cubic feet per second",
  "30208",         "Discharge, cubic meters per second, mean, 1 day",
  "30209",         "Discharge, instantaneous, cubic meters per second",
  "50042",         "Discharge, gallons per minute",
  "72243",         "Discharge, cubic feet per day"
)
```

Next, I called `whatNWISdata()` to explore the parameters available for
the specified site and filtered the returned dataframe to retain only
those observations with parameter codes of interest.

``` r
available_data <- dataRetrieval::whatNWISdata(siteNumber = site)

available_data <- available_data |>
  dplyr::select(site_no, data_type_cd, parm_cd, stat_cd, begin_date, end_date, count_nu) |>
  dplyr::filter(parm_cd %in% parameter_codes$parameter_code) |>
  dplyr::left_join(parameter_codes, by = c("parm_cd" = "parameter_code"))

knitr::kable(available_data, format = "pipe")
```

| site_no | data_type_cd | parm_cd | stat_cd | begin_date | end_date | count_nu | description |
|:---|:---|:---|:---|:---|:---|---:|:---|
| 05340500 | dv | 00010 | 00001 | 2000-01-21 | 2025-02-18 | 8648 | Temperature, water, degrees Celsius |
| 05340500 | dv | 00010 | 00002 | 2000-01-21 | 2025-02-18 | 8648 | Temperature, water, degrees Celsius |
| 05340500 | dv | 00010 | 00003 | 2000-01-21 | 2025-02-18 | 8747 | Temperature, water, degrees Celsius |
| 05340500 | dv | 00060 | 00003 | 1902-01-01 | 2025-02-18 | 43317 | Discharge, cubic feet per second, mean, 1 day |
| 05340500 | qw | 00010 | NA | 1968-06-03 | 1968-06-03 | 1 | Temperature, water, degrees Celsius |
| 05340500 | qw | 00010 | NA | 1966-11-06 | 2003-10-16 | 280 | Temperature, water, degrees Celsius |
| 05340500 | qw | 00060 | NA | 1966-11-06 | 1968-06-03 | 3 | Discharge, cubic feet per second, mean, 1 day |
| 05340500 | qw | 00060 | NA | 1966-11-06 | 1999-02-12 | 32 | Discharge, cubic feet per second, mean, 1 day |
| 05340500 | qw | 00061 | NA | 1974-10-02 | 2003-10-16 | 279 | Discharge, instantaneous, cubic feet per second |
| 05340500 | qw | 30208 | NA | 1966-11-06 | 1968-06-03 | 3 | Discharge, cubic meters per second, mean, 1 day |
| 05340500 | qw | 30208 | NA | 1966-11-06 | 1999-02-12 | 32 | Discharge, cubic meters per second, mean, 1 day |
| 05340500 | qw | 30209 | NA | 1974-10-02 | 2003-10-16 | 279 | Discharge, instantaneous, cubic meters per second |
| 05340500 | uv | 00010 | NA | 2007-10-01 | 2025-02-19 | 6351 | Temperature, water, degrees Celsius |
| 05340500 | uv | 00060 | NA | 1982-10-05 | 2025-02-19 | 15478 | Discharge, cubic feet per second, mean, 1 day |

Looking at this dataframe of available data, it appears that mean daily
water temperature and discharge data are available. Since I plan on
using the function `readNWISdv()`, which assumes the statistic code
“00003,” I don’t feel the need to look more specifically at statistic
codes.

## Data pull

Next, I pulled daily mean discharge in cubic feet per second using the
`readNWISdv()` function and the appropriate parameter codes. I could
have combined the flow and temperature pulls, but to be clear for the
purposes of this analysis, I pulled each parameter of interest
separately.

``` r
flow_parameter_code <- "00060" # daily mean discharge in cfs
temp_parameter_code <- "00010" # daily mean water temperature in degrees celsius

flow <- dataRetrieval::readNWISdv(
  siteNumbers = site,
  parameterCd = flow_parameter_code,
  startDate = start_date,
  endDate = end_date
)

temp <- dataRetrieval::readNWISdv(
  siteNumbers = site,
  parameterCd = temp_parameter_code,
  startDate = start_date,
  endDate = end_date
)
```

Let’s view the flow data pull results first:

``` r
dplyr::glimpse(flow)
```

    Rows: 365
    Columns: 5
    $ agency_cd        <chr> "USGS", "USGS", "USGS", "USGS", "USGS", "USGS", "USGS…
    $ site_no          <chr> "05340500", "05340500", "05340500", "05340500", "0534…
    $ Date             <date> 2022-10-01, 2022-10-02, 2022-10-03, 2022-10-04, 2022…
    $ X_00060_00003    <dbl> 2090, 1970, 2070, 2100, 2110, 2060, 1970, 1940, 1940,…
    $ X_00060_00003_cd <chr> "A", "A", "A", "A", "A", "A", "A", "A", "A", "A", "A"…

Next, let’s view the water temperature data pull results.

``` r
dplyr::glimpse(temp)
```

    Rows: 365
    Columns: 5
    $ agency_cd        <chr> "USGS", "USGS", "USGS", "USGS", "USGS", "USGS", "USGS…
    $ site_no          <chr> "05340500", "05340500", "05340500", "05340500", "0534…
    $ Date             <date> 2022-10-01, 2022-10-02, 2022-10-03, 2022-10-04, 2022…
    $ X_00010_00003    <dbl> 13.9, 14.4, 14.9, 15.6, 16.4, 16.4, 15.5, 13.9, 12.6,…
    $ X_00010_00003_cd <chr> "A", "A", "A", "A", "A", "A", "A", "A", "A", "A", "A"…

## Cleaning

The data don’t need much in the way of cleaning, but let’s fix column
names for clarity.

``` r
flow <- flow |>
  dplyr::rename(
    agency_code = agency_cd,
    date = Date,
    flow_cfs = X_00060_00003,
    qual_code = X_00060_00003_cd
  )

temp <- temp |>
  dplyr::rename(
    agency_code = agency_cd,
    date = Date,
    temp_c = X_00010_00003,
    qual_code = X_00010_00003_cd
  )
```

## Visualization

Lastly, I visualized the data as time series, as the prompt requested.

``` r
# In this script I have defaulted to fully qualifying function calls for
# clarity, but that's annoying with ggplot2 syntax, so let's load ggplot2

library(ggplot2)

ggplot(data = flow) +
  geom_line(aes(x = date, y = flow_cfs)) +
  scale_x_date(
    name = "Date",
    minor_breaks = "1 month",
    date_labels = "%b %Y"
  ) +
  scale_y_continuous(
    name = "Mean daily discharge (cfs)",
    labels = scales::comma
  ) +
  labs(
    title = glue::glue("Mean daily discharge: USGS site #{site}"),
    subtitle = "Water year 2022-2023"
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", margin = margin(b = 7.5)),
    plot.subtitle = element_text(face = "italic", margin = margin(b = 10)),
    axis.title.x = element_text(face = "bold", margin = margin(t = 10)),
    axis.title.y = element_text(face = "bold", margin = margin(r = 10))
  )
```

<img src="README_files/figure-commonmark/visualize-1.png"
style="width:100.0%" />

``` r
ggplot(data = temp) +
  geom_line(aes(x = date, y = temp_c)) +
  scale_x_date(
    name = "Date",
    minor_breaks = "1 month",
    date_labels = "%b %Y"
  ) +
  scale_y_continuous(
    name = "Mean daily water temperature (°C)",
    breaks = scales::pretty_breaks(n = 10)
  ) +
  labs(
    title = glue::glue("Mean water temperature: USGS site #{site}"),
    subtitle = "Water year 2022-2023"
  ) +
  theme_minimal() +
  theme(
    plot.title = element_text(face = "bold", margin = margin(b = 7.5)),
    plot.subtitle = element_text(face = "italic", margin = margin(b = 10)),
    axis.title.x = element_text(face = "bold", margin = margin(t = 10)),
    axis.title.y = element_text(face = "bold", margin = margin(r = 10))
  )
```

<img src="README_files/figure-commonmark/visualize-2.png"
style="width:100.0%" />
