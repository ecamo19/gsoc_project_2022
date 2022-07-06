CLM Sensitivity Analysis
================
Kristina Riemer

``` r
library(broom)
library(magrittr)
library(dplyr)
library(tidyr)
library(ggplot2)
library(cowplot)
```

Data are parameter values with corresponding response variable
values.

``` r
sens_data <- readr::read_csv("clm5_Equ_C4g_default_ens_e500_output.txt", 
                                 skip = 1)
colnames(sens_data) <- gsub(" ", "_", colnames(sens_data))
sens_data_ex <- sens_data %>% 
  select(1:10)
```

### Example of Sensitivity Analysis for Single Output Variable

#### 1\. Elasticity

Sensitivity is the change in the response variable given a change in an
input variable, which can be a parameter, initial condition, etc. This
is representated by the derivative dY/dX. This is approximated below as
the slope of the linear regression between the parameter values and
response variable. This is a quick-and-dirty variant of the
[`get.sensitivity`
function](https://github.com/PecanProject/pecan/blob/develop/modules/uncertainty/R/sensitivity.analysis.R#L73)
in PEcAn.

``` r
output_var_name <- tail(colnames(sens_data_ex), 1)

test_form <- as.formula(paste(output_var_name, "~", "."))

slopes <- tidy(lm(data = sens_data_ex, formula = test_form)) %>% 
  rename(sensitivity = estimate) %>% 
  filter(term != "(Intercept)")
```

Because the units of the derviative are determined by X’s units,
elasticity is used to standardize sensitivity across variables.
Elasticity is dY/dX \* (mean X/mean Y).

``` r
mean_response <- mean(sens_data_ex$TLAI)

mean_params <- sens_data_ex %>% 
  select(-TLAI) %>% 
  summarise_all(funs(mean)) %>% 
  gather(parameter, mean) %>% 
  mutate(elasticity_multiplier = mean / mean_response)
```

    ## Warning: funs() is soft deprecated as of dplyr 0.8.0
    ## Please use a list of either functions or lambdas: 
    ## 
    ##   # Simple named list: 
    ##   list(mean = mean, median = median)
    ## 
    ##   # Auto named with `tibble::lst()`: 
    ##   tibble::lst(mean, median)
    ## 
    ##   # Using lambdas
    ##   list(~ mean(., trim = .2), ~ median(., na.rm = TRUE))
    ## This warning is displayed once per session.

``` r
elasticity <- left_join(slopes, mean_params, by = c("term" = "parameter")) %>% 
  mutate(elasticity = sensitivity * elasticity_multiplier)
```

Elasticity values increasingly farther from zero mean change in x is
causing increasingly greater change in y. A change in x causes the same
change in y when elasticity is one.

``` r
ggplot(elasticity, aes(x = elasticity, y = term)) +
  geom_point() +
  geom_vline(xintercept = 0)
```

![](clm_sensitivity_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

#### 2\. Coefficient of Variation

CV is the normalized parameter variance of all the input parameters.

``` r
CVs <- sens_data_ex %>% 
  select(-TLAI) %>% 
  gather(parameter, value) %>% 
  group_by(parameter) %>% 
  summarize(mean = mean(value), 
            var = var(value)) %>% 
  mutate(sd = sqrt(var), 
         cv = (sd / mean) * 100)
```

``` r
ggplot(CVs, aes(x = cv, y = parameter)) +
  geom_point() +
  geom_vline(xintercept = 0) +
  xlab("CV (%)")
```

![](clm_sensitivity_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

#### 3\. Explained Standard Deviation

How much of the response variable uncertainty is explained by each of
the parameters. Incorporates both parameter variance and sensitivity.

``` r
SDs <- tidy(aov(TLAI ~ ., data = sens_data_ex)) %>% 
  mutate(sd = sqrt(sumsq)) %>% 
  select(term, sd)
```

``` r
ggplot(SDs, aes(x = sd, y = term)) +
  geom_point() +
  geom_vline(xintercept = 0)
```

![](clm_sensitivity_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

### Sensitivity Analysis for All Output Variables

``` r
for(output_var in colnames(sens_data)[10:17]){
  output_sens_data <- sens_data %>% 
    select(sla:root_dist, output_var)
  output_var_name <- tail(colnames(output_sens_data), 1)
  slopes_formula <- as.formula(paste(output_var_name, "~", "."))
  slopes <- tidy(lm(data = output_sens_data, formula = slopes_formula)) %>%
    rename(sensitivity = estimate) %>%
    filter(term != "(Intercept)")
  mean_response <- mean(data.matrix(output_sens_data[output_var_name]))
  mean_params <- output_sens_data %>%
    select(-output_var_name) %>%
    summarise_all(funs(mean)) %>%
    gather(parameter, mean) %>%
    mutate(elasticity_multiplier = mean / mean_response)
  elasticity <- left_join(slopes, mean_params, by = c("term" = "parameter")) %>%
    mutate(elasticity = sensitivity * elasticity_multiplier,
           parameter = case_when(term == "sla" ~ "Specific Leaf Area", 
                                 term == "fineroot2leaf" ~ "Fine Root to Leaf Biomass Ratio", 
                                 term == "c2n_leaf" ~ "Leaf C:N", 
                                 term == "flnr_in" ~ "Fraction Leaf N in Rubisco", 
                                 term == "target_c2n_froot" ~ "Fine Root C:N", 
                                 term == "stom_slope_g1" ~ "Stomatal Conductance Slope", 
                                 term == "min_stom_cond" ~ "Minimum Stomatal Conductance", 
                                 term == "root_dist" ~ "Root Distribution"))
  elasticity_plot <- ggplot(elasticity, aes(x = elasticity, y = parameter)) +
    geom_point() +
    geom_vline(xintercept = 0) +
    ggtitle(output_var)
  
  # CVs <- output_sens_data %>%
  #   select(-output_var_name) %>%
  #   gather(term, value) %>%
  #   group_by(term) %>%
  #   summarize(mean = mean(value),
  #             var = var(value)) %>%
  #   mutate(sd = sqrt(var),
  #          cv = (sd / mean) * 100, 
  #          parameter = case_when(term == "sla" ~ "Specific Leaf Area", 
  #                                term == "fineroot2leaf" ~ "Fine Root to Leaf Biomass Ratio", 
  #                                term == "c2n_leaf" ~ "Leaf C:N", 
  #                                term == "flnr_in" ~ "Fraction Leaf N in Rubisco", 
  #                                term == "target_c2n_froot" ~ "Fine Root C:N", 
  #                                term == "stom_slope_g1" ~ "Stomatal Conductance Slope", 
  #                                term == "min_stom_cond" ~ "Minimum Stomatal Conductance", 
  #                                term == "root_dist" ~ "Root Distribution"))
  # CV_plot <- ggplot(CVs, aes(x = cv, y = parameter)) +
  #   geom_point() +
  #   geom_vline(xintercept = 0) +
  #   xlab("CV (%)")
  
  aov_formula <- as.formula(paste(output_var_name, "~", "."))
  SDs <- tidy(aov(data = output_sens_data, formula = aov_formula)) %>%
    mutate(sd = sqrt(sumsq)) %>%
    select(term, sd) %>% 
    filter(term != "Residuals") %>% 
    mutate(parameter = case_when(term == "sla" ~ "Specific Leaf Area", 
                                 term == "fineroot2leaf" ~ "Fine Root to Leaf Biomass Ratio", 
                                 term == "c2n_leaf" ~ "Leaf C:N", 
                                 term == "flnr_in" ~ "Fraction Leaf N in Rubisco", 
                                 term == "target_c2n_froot" ~ "Fine Root C:N", 
                                 term == "stom_slope_g1" ~ "Stomatal Conductance Slope", 
                                 term == "min_stom_cond" ~ "Minimum Stomatal Conductance", 
                                 term == "root_dist" ~ "Root Distribution"))
  SD_plot <- ggplot(SDs, aes(x = sd, y = parameter)) +
    geom_point() +
    geom_vline(xintercept = 0)
  all_plots <- plot_grid(elasticity_plot, SD_plot, ncol = 1)
  print(all_plots)
}
```

![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-2.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-3.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-4.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-5.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-6.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-7.png)<!-- -->![](clm_sensitivity_files/figure-gfm/unnamed-chunk-10-8.png)<!-- -->
