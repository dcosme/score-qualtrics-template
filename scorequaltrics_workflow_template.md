This script is a template workflow for scoring Qualtrics data using the
[`scorequaltrics`](https://github.com/jflournoy/qualtrics) package built
by [John Flournoy](https://github.com/jflournoy) and is a pared down
version of the tutorial he created for the TDS study.

Generate a credentials file
---------------------------

To pull data from Qualtrics, you need a credentials file with an API
token associated with your account. To create the file, follow these
steps.

1.  Generate an API token for Qualtrics. Follow the steps outlined
    [here](https://www.qualtrics.com/support/integrations/api-integration/overview/)

2.  Create `qualtrics_credentials.yaml` in the `credentialDir` and add
    API token information

``` bash
credentialDir='/Users/danicosme/' #replace with your path

if [ ! -f ${credentialDir}qualtrics_credentials.yaml ]; then
  cd ${credentialDir}
  touch qualtrics_credentials.yaml
  echo "token: Ik0XNN...." >> qualtrics_credentials.yaml #replace with your token information
  echo "baseurl: oregon.qualtrics.com" >> qualtrics_credentials.yaml
  echo "credential file created"
else
  echo "credential file already exists in this location"
fi
```

    ## credential file already exists in this location

Load packages
-------------

``` r
if (!require(tidyverse)) {
  install.packages('tidyverse')
}

if (!require(knitr)) {
  install.packages('knitr')
}

if (!require(devtools)) {
  install.packages('devtools')
}

if (!require(scorequaltrics)) {
  devtools::install_github('dcosme/qualtrics', ref = "dev/enhance")
}

if (!require(ggcorrplot)) {
  install.packages('ggcorrplot')
}
```

Define variables and paths
--------------------------

-   `cred_file_location` = path to your Qualtrics credential file.
    You’ll need to generate this via Qualtrics using the instructions
    above.
-   `keep_columns` = subject ID column name and any other columns in
    Qualtrics survey you want to keep in wide format (all others will be
    gathered into a key-value pair); can be a regular expression
-   `survey_name_filter` = regular expression to select surveys
-   `sid_pattern` = regular expression for participant IDs
-   `exclude_sid` = regular expression for participant IDs to exclude
    (e.g. test responses)
-   `identifiable_data` = identifiable data you do not want to include
    in the dataframe
-   `output_file_dir` = output file directory
-   `rubric_dir` = scoring rubric directory

``` r
cred_file_location = '~/qualtrics_credentials.yaml'
keep_columns = '(ResponseId|SID|ExternalReference|Finished)'
survey_name_filter = 'Freshman Project T.* Survey'
sid_pattern = 'FP[0-9]{3}'
exclude_sid = 'FP999' # subject IDs to exclude
identifiable_data = c('IPAddress', "RecipientEmail", "RecipientLastName", "RecipientFirstName",
                      "LocationLatitude", "LocationLongitude") # exclude when printing duplicates
output_file_dir = '~/Documents/code/score-qualtrics'
rubric_dir = '~/Documents/code/score-qualtrics/rubrics'
```

Access qualtrics data
---------------------

Filter available surveys based on the filter specified above.

``` r
# load credential file
credentials = scorequaltrics::creds_from_file(cred_file_location)

# filter
surveysAvail = scorequaltrics::get_surveys(credentials)
surveysFiltered = filter(surveysAvail, grepl(survey_name_filter, SurveyName))

knitr::kable(arrange(select(surveysFiltered, SurveyName), SurveyName))
```

| SurveyName                         |
|:-----------------------------------|
| Freshman Project T1 Survey (Pilot) |
| Freshman Project T2 Survey (Pilot) |
| Freshman Project T3 Survey (Pilot) |
| Freshman Project T4 Survey (Pilot) |

Cleaning and scoring data
-------------------------

### Get survey data

The `get_survey_data` function pulls the data from the surveys specified
in `surveysFiltered` and reshapes into the long format. Because the
example data also includes some identifying information, we also want to
filter those items out of our dataframe.

``` r
# get data
surveys_long = scorequaltrics::get_survey_data(surveysFiltered,
                                               pid_col = keep_columns) %>%
               filter(!item %in% identifiable_data) #filter out identifiable data
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |======================================================================| 100%
    ##   |                                                                              |                                                                      |   0%  |                                                                              |======================================================================| 100%
    ##   |                                                                              |                                                                      |   0%  |                                                                              |======================================================================| 100%
    ##   |                                                                              |                                                                      |   0%  |                                                                              |======================================================================| 100%

``` r
# print first 10 rows
head(select(surveys_long, -ResponseId), 10)
```

    ## # A tibble: 10 x 6
    ## # Rowwise: 
    ##    Finished ExternalReference item     value    survey_name                SID  
    ##       <dbl> <chr>             <chr>    <chr>    <chr>                      <chr>
    ##  1        1 <NA>              StartDa… 1447719… Freshman Project T2 Surve… <NA> 
    ##  2        1 <NA>              StartDa… 1447725… Freshman Project T2 Surve… <NA> 
    ##  3        1 <NA>              StartDa… 1447723… Freshman Project T2 Surve… <NA> 
    ##  4        1 <NA>              StartDa… 1447741… Freshman Project T2 Surve… <NA> 
    ##  5        1 <NA>              StartDa… 1447746… Freshman Project T2 Surve… <NA> 
    ##  6        1 <NA>              StartDa… 1447818… Freshman Project T2 Surve… <NA> 
    ##  7        1 <NA>              StartDa… 1447744… Freshman Project T2 Surve… <NA> 
    ##  8        1 <NA>              StartDa… 1447822… Freshman Project T2 Surve… <NA> 
    ##  9        1 <NA>              StartDa… 1447825… Freshman Project T2 Surve… <NA> 
    ## 10        1 FP018             StartDa… 1447914… Freshman Project T2 Surve… <NA>

### Load scoring rubrics

To automatically score the surveys, scoring rubrics with the following
format must be provided:

``` r
read.csv('examplerubric.csv', stringsAsFactors = FALSE, check.names = FALSE)
```

    ##                Data File Name Scale Name Column Name Reverse Min Max Transform
    ## 1  Freshman Project T1 Survey        BIS       BIS_1       0   1   4         0
    ## 2  Freshman Project T1 Survey        BIS       BIS_2       0   1   4         0
    ## 3  Freshman Project T1 Survey        BIS       BIS_3       0   1   4         0
    ## 4  Freshman Project T1 Survey        BIS       BIS_4       0   1   4         0
    ## 5  Freshman Project T1 Survey        BIS       BIS_5       0   1   4         0
    ## 6  Freshman Project T1 Survey        BIS       BIS_6       1   1   4         0
    ## 7  Freshman Project T1 Survey        BIS       BIS_7       1   1   4         0
    ## 8  Freshman Project T1 Survey        BIS       BIS_8       1   1   4         0
    ## 9  Freshman Project T1 Survey        BIS       BIS_9       1   1   4         0
    ## 10 Freshman Project T1 Survey        BIS      BIS_10       1   1   4         0
    ## 11 Freshman Project T1 Survey        BIS      BIS_11       0   1   4         0
    ## 12 Freshman Project T1 Survey        BIS      BIS_12       0   1   4         0
    ## 13 Freshman Project T1 Survey        BIS      BIS_13       1   1   4         0
    ## 14 Freshman Project T1 Survey        BIS      BIS_14       0   1   4         0
    ## 15 Freshman Project T1 Survey        BIS      BIS_15       0   1   4         0
    ##    Nonplanning_Impulsivity_T1 Motor_Impulsivity_T1 Attentional_Impulsivity_T1
    ## 1                           0                  sum                          0
    ## 2                           0                  sum                          0
    ## 3                           0                  sum                          0
    ## 4                           0                  sum                          0
    ## 5                           0                  sum                          0
    ## 6                         sum                    0                          0
    ## 7                         sum                    0                          0
    ## 8                         sum                    0                          0
    ## 9                         sum                    0                          0
    ## 10                        sum                    0                          0
    ## 11                          0                    0                        sum
    ## 12                          0                    0                        sum
    ## 13                          0                    0                        sum
    ## 14                          0                    0                        sum
    ## 15                          0                    0                        sum
    ##    Total
    ## 1    sum
    ## 2    sum
    ## 3    sum
    ## 4    sum
    ## 5    sum
    ## 6    sum
    ## 7    sum
    ## 8    sum
    ## 9    sum
    ## 10   sum
    ## 11   sum
    ## 12   sum
    ## 13   sum
    ## 14   sum
    ## 15   sum

Scoring rubrics should exist in `rubric_dir` and be named according to
the following convention: `[measure]_scoring_rubric.csv`

``` r
# specify rubric paths
scoring_rubrics = data.frame(file = dir(file.path(rubric_dir), 
                                        pattern = '.*scoring_rubric.*.csv',
                                        full.names = TRUE))

# read in rubrics
scoring_data_long = scorequaltrics::get_rubrics(scoring_rubrics,
                                                type = 'scoring')
# print the first 10 rows
head(scoring_data_long[, -1], 10)
```

    ## # A tibble: 10 x 9
    ##    data_file_name scale_name column_name reverse min   max   transform
    ##    <chr>          <chr>      <chr>       <chr>   <chr> <chr> <chr>    
    ##  1 Freshman Proj… BIS        BIS_1       0       1     4     0        
    ##  2 Freshman Proj… BIS        BIS_2       0       1     4     0        
    ##  3 Freshman Proj… BIS        BIS_3       0       1     4     0        
    ##  4 Freshman Proj… BIS        BIS_4       0       1     4     0        
    ##  5 Freshman Proj… BIS        BIS_5       0       1     4     0        
    ##  6 Freshman Proj… BIS        BIS_6       1       1     4     0        
    ##  7 Freshman Proj… BIS        BIS_7       1       1     4     0        
    ##  8 Freshman Proj… BIS        BIS_8       1       1     4     0        
    ##  9 Freshman Proj… BIS        BIS_9       1       1     4     0        
    ## 10 Freshman Proj… BIS        BIS_10      1       1     4     0        
    ## # … with 2 more variables: scored_scale <chr>, include <chr>

### Cleaning

-   exclude non-sub responses
-   convert missing values to NA
-   duplicates

First, exclude responses that are not subject responses.

In this dataset, some subjects have their ID in the `ExternalReference`
column only, so we’ll need to add that to the `SID` column before
filtering. There are also some test responses that match our SID
pattern, so we’ll want to exclude those using the `exclude_SID` pattern.

``` r
surveys_long_sub = surveys_long %>%
  mutate(SID = ifelse(is.na(SID), ExternalReference, SID)) %>%
  select(-ExternalReference) %>%
  filter(grepl(sid_pattern, SID)) %>%
  filter(!grepl(exclude_sid, SID)) %>%
  arrange(SID)

# print unique SIDs
unique(surveys_long_sub$SID)
```

    ##  [1] "FP001" "FP002" "FP003" "FP004" "FP005" "FP006" "FP007" "FP008" "FP009"
    ## [10] "FP010" "FP011" "FP012" "FP013" "FP014" "FP015" "FP016" "FP017" "FP018"
    ## [19] "FP019" "FP020" "FP021" "FP022" "FP023" "FP024" "FP025" "FP026" "FP027"
    ## [28] "FP028" "FP029" "FP030" "FP031" "FP032" "FP034" "FP035"

Convert missing values to NA.

``` r
surveys_long_na = surveys_long_sub %>%
  mutate(value = ifelse(value == "", NA, value))
```

Check for non-numeric items using the `get_uncoercibles()` function.

``` r
surveys_long_na %>%
  scorequaltrics::get_uncoercibles() %>%
  distinct(item, value) %>%
  arrange(item) %>%
  head(., 10)
```

    ##        item
    ##  1: CARE_EI
    ##  2: CARE_EI
    ##  3: CARE_EI
    ##  4: CARE_EI
    ##  5: CARE_EI
    ##  6: CARE_EI
    ##  7: CARE_EI
    ##  8: CARE_EI
    ##  9: CARE_EI
    ## 10: CARE_EI
    ##                                                                    value
    ##  1:                                                              5 weeks
    ##  2:                                                              1 month
    ##  3:                                                               1 year
    ##  4:                                                             7 months
    ##  5:                                                             4 months
    ##  6:                                                              2 years
    ##  7:                                                             16 weeks
    ##  8:  A regular partner is someone that I have dated for at least 8 weeks
    ##  9:                                                     3 months or more
    ## 10: A regular partner is someone that I have dated for at least 8 weeks.

Make manual edits before converting values to numeric during scoring

``` r
# save ethnicity information as a separate variable
CVS_3 = surveys_long_na %>%
  mutate(value = ifelse(item == "CVS_3", tolower(value), value)) %>%
  filter(item == "CVS_3")

# make manual edits and convert values to numeric
surveys_long_num = surveys_long_na %>%
  mutate(value = ifelse(SID == "FP007" & item == "CVS_1", "18",
                 ifelse(SID == "FP006" & item == "CVS_15", "3.47",
                 ifelse(SID == "FP002" & item == "CVS_16", "3",
                 ifelse(SID == "FP006" & item == "CVS_16", "3.7", value)))))
```

Check for duplicate responses. There is a `clean_dupes` function that
can do this, but since we have multiple waves with the same surveys,
we’re going to do this homebrew.

``` r
surveys_long_num %>%
  spread(item, value) %>%
  group_by(survey_name, SID) %>%
  summarize(n = n()) %>%
  arrange(desc(n)) %>%
  filter(n > 1)
```

    ## # A tibble: 1 x 3
    ## # Groups:   survey_name [1]
    ##   survey_name                        SID       n
    ##   <chr>                              <chr> <int>
    ## 1 Freshman Project T2 Survey (Pilot) FP002     2

Since FP002 appears to have taken the T2 survey twice, we’re simply
going to randomly select based on the qid.

``` r
surveys_long_clean = surveys_long_num %>%
  filter(!ResponseId == "R_11YpEE2pH9Ozqvk") %>%
  select(-ResponseId)
```

First, get only the items used in the scoring rubrics.

``` r
scoring = scorequaltrics::get_rubrics(scoring_rubrics, type = 'scoring')
```

### Score the questionnaires

``` r
scored = scorequaltrics::score_questionnaire(surveys_long_clean, scoring, SID = "SID", psych = FALSE)

# print first 200 rows
head(scored, 200)
```

    ## # A tibble: 200 x 8
    ## # Groups:   survey_name, scale_name, scored_scale [6]
    ##    survey_name    scale_name scored_scale SID   score   n_items n_missing method
    ##    <chr>          <chr>      <chr>        <chr> <I<chr>   <int>     <int> <chr> 
    ##  1 Freshman Proj… CVS        ethnicity_t… FP001 caucas…       1         0 I     
    ##  2 Freshman Proj… CVS        ethnicity_t… FP002 hispan…       1         0 I     
    ##  3 Freshman Proj… CVS        ethnicity_t… FP003 caucas…       1         0 I     
    ##  4 Freshman Proj… CVS        ethnicity_t… FP004 Caucas…       1         0 I     
    ##  5 Freshman Proj… CVS        ethnicity_t… FP005 white         1         0 I     
    ##  6 Freshman Proj… CVS        ethnicity_t… FP006 White         1         0 I     
    ##  7 Freshman Proj… CVS        ethnicity_t… FP007 Caucas…       1         0 I     
    ##  8 Freshman Proj… CVS        ethnicity_t… FP008 White         1         0 I     
    ##  9 Freshman Proj… CVS        ethnicity_t… FP009 white         1         0 I     
    ## 10 Freshman Proj… CVS        ethnicity_t… FP010 White         1         0 I     
    ## # … with 190 more rows

Plots
-----

### Distributions

#### Grouped by scale

``` r
scored %>%
  filter(!method == "I") %>% # filter out non-numeric data
  mutate(score = as.numeric(score)) %>%
  group_by(scale_name) %>%
    do({
      plot = ggplot(., aes(scored_scale, score)) +
        geom_boxplot() +
        geom_jitter(height = .01, width = .15, alpha = .5, color = "#2A908B") +
        labs(x = "", y = "score\n", title = sprintf("%s\n", .$scale_name[[1]])) + 
        theme_minimal(base_size = 16) +
        theme(text = element_text(family = "Futura Medium", colour = "black"),
              legend.text = element_text(size = 8),
              axis.text = element_text(color = "black"),
              axis.text.x = element_text(angle = 45, vjust = 1, hjust = 1),
              panel.grid.major = element_blank(),
              panel.grid.minor = element_blank(),
              panel.border = element_blank(),
              panel.background = element_blank(),
              plot.title = element_text(hjust = 0.5))
      print(plot)
      data.frame()
    })
```

![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist-1.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist-2.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist-3.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist-4.png)

    ## # A tibble: 0 x 1
    ## # Groups:   scale_name [0]
    ## # … with 1 variable: scale_name <chr>

#### Grouped by scored scale

``` r
scored %>%
  filter(!method == "I") %>% # filter out non-numeric data
  mutate(score = as.numeric(score)) %>%
  group_by(scale_name, scored_scale) %>%
    do({
      plot = ggplot(., aes(scored_scale, score)) +
        geom_boxplot() +
        geom_jitter(height = .01, width = .15, alpha = .5, color = "#2A908B") +
        labs(x = "", y = "score\n", title = sprintf("%s %s\n", .$scale_name[[1]], .$scored_scale[[1]])) + 
        theme_minimal(base_size = 16) +
        theme(text = element_text(family = "Futura Medium", colour = "black"),
              axis.text = element_text(color = "black"),
              axis.text.x = element_text(angle = 45, vjust = 1, hjust = 1),
              panel.grid.major = element_blank(),
              panel.grid.minor = element_blank(),
              panel.border = element_blank(),
              panel.background = element_blank(),
              plot.title = element_text(hjust = 0.5))
      print(plot)
      data.frame()
    })
```

![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-1.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-2.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-3.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-4.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-5.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-6.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-7.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-8.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotdist2-9.png)

    ## # A tibble: 0 x 2
    ## # Groups:   scale_name, scored_scale [0]
    ## # … with 2 variables: scale_name <chr>, scored_scale <chr>

### Proportion of missing data

``` r
scored %>%
  filter(!method == "I") %>% # filter out non-numeric data
  mutate(score = as.numeric(score)) %>%
  group_by(scale_name) %>%
    do({
      plot = ggplot(., aes(scored_scale, n_missing)) +
        geom_violin() +
        geom_jitter(height = .01, width = .15, alpha = .5, color = "#2A908B") +
        labs(title = sprintf("%s %s\n", .$scale_name[[1]], .$scored_scale[[1]])) + 
        labs(x = "", y = "score\n") + 
        theme_minimal(base_size = 16) +
        theme(text = element_text(family = "Futura Medium", colour = "black"),
              axis.text = element_text(color = "black"),
              axis.text.x = element_text(angle = 45, vjust = 1, hjust = 1),
              panel.grid.major = element_blank(),
              panel.grid.minor = element_blank(),
              panel.border = element_blank(),
              panel.background = element_blank(),
              plot.title = element_text(hjust = 0.5))
      print(plot)
      data.frame()
    })
```

![](scorequaltrics_workflow_template_files/figure-markdown_github/plotmissing-1.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotmissing-2.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotmissing-3.png)![](scorequaltrics_workflow_template_files/figure-markdown_github/plotmissing-4.png)

    ## # A tibble: 0 x 1
    ## # Groups:   scale_name [0]
    ## # … with 1 variable: scale_name <chr>

### Changes across time

For those variables that were measured more than once, plot changes.

``` r
scored %>%
  filter(!method == "I") %>% # filter out non-numeric data
  mutate(score = as.numeric(score)) %>%
  extract(survey_name, "wave", ".*([0-9]{1}).*", remove = FALSE) %>%
  group_by(scale_name, scored_scale) %>%
  mutate(nrow = n()) %>%
  filter(nrow > 34) %>%
    do({
      plot = ggplot(., aes(wave, score)) +
        geom_point(aes(group = SID), fill = "black", alpha = .05, size = 3) +
        geom_line(aes(group = SID), color = "black", alpha = .05, size = 1) +
        stat_summary(fun.data = "mean_cl_boot", size = 1.5, color = "#3B9AB2") +
        stat_summary(aes(group = 1), fun.y = mean, geom = "line", size = 1.5, color = "#3B9AB2") +
        labs(x = "\nwave", y = "score\n", title = sprintf("%s %s\n", .$scale_name[[1]], .$scored_scale[[1]])) + 
        theme_minimal(base_size = 16) +
        theme(text = element_text(family = "Futura Medium", colour = "black"),
              axis.text = element_text(color = "black"),
              axis.line = element_line(colour = "black"),
              panel.grid.major = element_blank(),
              panel.grid.minor = element_blank(),
              panel.border = element_blank(),
              panel.background = element_blank(),
              plot.title = element_text(hjust = 0.5))
      print(plot)
      data.frame()
    })
```

![](scorequaltrics_workflow_template_files/figure-markdown_github/plotchange-1.png)

    ## # A tibble: 0 x 2
    ## # Groups:   scale_name, scored_scale [0]
    ## # … with 2 variables: scale_name <chr>, scored_scale <chr>

### Correlations

``` r
scored %>%
  filter(!method == "I") %>% # filter out non-numeric data
  mutate(score = as.numeric(score)) %>%
  filter(!scale_name == "CVS") %>%
  extract(survey_name, "wave", ".*(T[0-9]{1}).*", remove = FALSE) %>%
  mutate(var.name = paste(scale_name, scored_scale, wave, sep = " ")) %>%
  ungroup() %>%
  select(var.name, score, SID) %>%
  spread(var.name, score) %>%
  filter(!is.na(SID)) %>%
  select(-SID) %>%
  cor(., use = "pairwise.complete.obs") %>%
  ggcorrplot(hc.order = TRUE, outline.col = "white", colors = c("#3B9AB2", "white", "#E46726")) + 
    geom_text(aes(label = round(value, 2)), size = 4, family = "Futura Medium") +
    labs(x = "", y = "") + 
    theme_minimal(base_size = 16) +
    theme(text = element_text(family = "Futura Medium", colour = "black"),
          legend.text = element_text(size = 8),
          axis.text = element_text(color = "black"),
          axis.text.x = element_text(angle = 45, vjust = 1, hjust = 1),
          panel.grid.major = element_blank(),
          panel.grid.minor = element_blank(),
          panel.border = element_blank(),
          panel.background = element_blank())
```

![](scorequaltrics_workflow_template_files/figure-markdown_github/plotcorr-1.png)
