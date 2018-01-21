# googleAnalyticsProphetR
Applying Facebook's prophet on Google Analytics data

# Motivation
One the problems we have in Digital Analytics is figuring out when something has stopped recording or fires more frequently that it should (you know ; fire once per page vs per event).

# Strategy
In this attempt we are taking a data-driven approach to detecting deviations from the "expected" (ref: remains to be defined). One of the most accesible ways to get a estimation of "expected" is by using Facebook's [prophet]() API which is available both in R and Python. The proposed strategy is to create daily the prediction for the previous day and compare it to the actual count of events in discussion.

In practice, prophet does really well in point estimation but we can also get upper and lower prediction bounds. Actually, we will trigger an alert when the actual value is outside these bounds.

# Under the hood
To create the we have wrapped somethings around the following functions that are originating from [googleAnalyticsR()]() and [prophet()]() :

- [`get_ga_data()`]()
- [`get_prophet_prediction()`]()
- [`get_prophet_prediction_graph()`]()

*Side note* : Actually there is another function that is based on Twitter's awesome [AnomalyDetection]() package (only for R).

# Example(s)
There is a sample RNotebook under the Reports folder ([report.rmd]()) that you can use with minimal configuration.

## Configuration
### Packages
As usual you will need to have all the packages mentioned on the [requirements.R]() file.

### Authentication
Then you will need to authenticate to Google via any method you like and is provide in [googleAuthR](), in the example I authenticate once and then reuse the `.httr-oauth`. A deeper explanation of authentication can be found [here]().

### Parameters
You will need to pass your `GA_VIEW_ID` for the API calls and your dimensions and metric of interest (default :  `totalEvents`). Note, that since we need to have a time series by the definition of the problem `date` is always added in the dimensions.

```R
## Define the ID of the VIEW we need to fetch
id <- "YOUR_VIEW_ID" # this is for the internal/legacy/YOU_NAME_IT...

## Build the event list we are interested
## in monitoring for the V1.0
events_category <- c(
  # YOUR_EVENTS_LIST
)

## Dimensions for breakdown
dimensions <- c(
  # YOUR_DIMENSIONS_LIST
)
```

## Acquire the data
Now, we are pulling the data from Google Analytics API. We are pushing the `events_category` as a paremeter to the `get_ga_data` and getting a dataframe back using purrr's `map_df()` ; which is awesome.

```R
## Get the data from GA
ga_data <- events_category %>%
  map_df(~ get_ga_data(id, start, end, .x, breakdown_dimensions = dimensions))
```
Now, we can check what we got data via a summary of the `ga_data`. You can use base [`summary`]() or [`skimr`](); I use the second one.

```R
# Summary of what we got from GA API
# Look for strange things in the 'n_unique' column of dimensions
# and 5-num summary of metrics (ie totalEvents)
ga_data %>%
  skimr::skim_to_wide()
```

### Interlude : The tricky part
You will need to do your own sanity check of inputs to the data that we pass to prophet object! This is out of the scope of the current implementation. So use the section below for passing over the constrains you'd like to, in other words create filters...

```R
data <- ga_data %>%
  filter(deviceCategory != "tablet")

## Let's keep the most important stuff
channel_groups <- c("Direct", "Non Brand SEO", "Brand SEO", "SEM Brand", "SEM Non Brand")
landing_groups <- c(
  # YOUR_LANDING_PAGE_GROUP_LIST
  )
```

## Inspect predictions
Let's check a random 10 rows of prediction along their actual value on the last day of the run.

```R
prophet_data %>%
  dplyr::select(-min, -max, -estimate, -data) %>%
  mutate_at(vars(starts_with("prophet_")), funs(round(., digits = 2))) %>%
  filter(prophet_lower_range > 0) %>% 
  dplyr::select(-prophet_lower_range, -prophet_upper_range) %>%
  sample_n(10)
```

## Get Alert
Next, we pull all the deviating cases.    
(*NOTE* : If this section is empty then we have no anomalous case)

```R
## Apply the prophet prediction to each group
alert_data <- prophet_data %>%
  rowwise() %>%
  filter(prophet_lower_range > 0) %>%
  mutate(flag = if_else(
    between(last_day, prophet_lower_range, prophet_upper_range),
    0,
    1
  )) %>%
  filter(flag > 0) %>%
  dplyr::select(-min, -max, -estimate, -data) %>%
  mutate_at(vars(starts_with("prophet_")), funs(round(., digits = 2)))
```
# Extension(s)
Now, you can push the above into Slack (using [`SlackR`]()) or send an email (using [`blastula`]() for example).