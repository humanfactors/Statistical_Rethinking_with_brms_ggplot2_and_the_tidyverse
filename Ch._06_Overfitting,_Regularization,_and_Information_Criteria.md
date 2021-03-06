---
title: "Ch. 6 Overfitting, Regularization, and Information Criteria"
author: "A Solomon Kurz"
date: "2018-04-16"
output:
  html_document:
    code_folding: show
    keep_md: TRUE
---

### 6.1.1. More parameters always impove fit.

We'll start of by making the data with brain size and body size for seven `species`.


```r
library(tidyverse)

(
  d <- 
  tibble(species = c("afarensis", "africanus", "habilis", "boisei",
                     "rudolfensis", "ergaster", "sapiens"), 
         brain = c(438, 452, 612, 521, 752, 871, 1350), 
         mass = c(37.0, 35.5, 34.5, 41.5, 55.5, 61.0, 53.5))
  )
```

```
## # A tibble: 7 x 3
##   species     brain  mass
##   <chr>       <dbl> <dbl>
## 1 afarensis     438  37.0
## 2 africanus     452  35.5
## 3 habilis       612  34.5
## 4 boisei        521  41.5
## 5 rudolfensis   752  55.5
## 6 ergaster      871  61.0
## 7 sapiens      1350  53.5
```

Here's our version of Figure 6.2.


```r
# install.packages("ggrepel", depencencies = T)
library(ggrepel)

set.seed(438) #  This makes the geom_text_repel() part reproducible
d %>%
  ggplot(aes(x =  mass, y = brain, label = species)) +
  theme_classic() +
  geom_point(color = "plum") +
  geom_text_repel(size = 3, color = "plum4", family = "Courier") +
  coord_cartesian(xlim = 30:65) +
  labs(x = "body mass (kg)",
       y = "brain volume (cc)",
       subtitle = "Average brain volume by body mass\nfor six hominin species") +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

I’m not going to bother with models `m6.1` through `m6.7` They were all done with the frequentist `lm()` function, which yields model estimates with OLS. Since the primary job of this manuscript is to convert rethinking code to brms code, those models provide nothing to convert.

Onward. 

## 6.2. Information theory and model performance

### 6.2.2. Information and uncertainty.

##### Overthinking: Computing deviance.

Here is how to compute deviance with brms.


```r
# data manipulation
d <-
  d %>%
  mutate(mass.s = (mass - mean(mass))/sd(mass))

library(brms)

# Here we specify our starting values
Inits <- list(Intercept = mean(d$brain),
              mass.s = 0,
              sigma = sd(d$brain))

InitsList <-list(Inits, Inits, Inits, Inits)

# The model
b6.8 <- 
  brm(data = d, family = gaussian,
      brain ~ 1 + mass.s,
      prior = c(set_prior("normal(0, 1000)", class = "Intercept"),
                set_prior("normal(0, 1000)", class = "b"),
                set_prior("cauchy(0, 10)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4,
      inits = InitsList)  # Here we put our start values in the brm() function

print(b6.8)
```

```
##  Family: gaussian 
##   Links: mu = identity; sigma = identity 
## Formula: brain ~ 1 + mass.s 
##    Data: d (Number of observations: 7) 
## Samples: 4 chains, each with iter = 2000; warmup = 1000; thin = 1; 
##          total post-warmup samples = 4000
##     ICs: LOO = NA; WAIC = NA; R2 = NA
##  
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## Intercept   706.03    103.16   493.67   913.79       2853 1.00
## mass.s      221.23    114.61   -17.64   442.83       2069 1.00
## 
## Family Specific Parameters: 
##       Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## sigma   262.18     90.82   147.35   493.34       1592 1.00
## 
## Samples were drawn using sampling(NUTS). For each parameter, Eff.Sample 
## is a crude measure of effective sample size, and Rhat is the potential 
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

**Details about `inits`**: You don’t have to specify your `inits` lists outside of the `brm()` function the way we did, here. This is just how I currently prefer to do it. When you specify start values for the parameters in your Stan models, you need to do so with a list of lists. You need as many lists as HMC chains—four in this example. And then you put your—in this case—four lists inside a list. Lists within lists. Also, we were lazy and specified the same start values across all our chains. You can mix them up across chains if you want.

Anyway, the brms function `log_lik()` returns a matrix. Each occasion gets a column and each HMC chain iteration gets a row.


```r
dfLL <-
  b6.8 %>%
  log_lik() %>%
  as_tibble()

dfLL %>%
  glimpse()
```

```
## Observations: 4,000
## Variables: 7
## $ V1 <dbl> -6.283169, -6.283169, -7.584661, -7.379721, -6.696407, -6.8...
## $ V2 <dbl> -6.059161, -6.059161, -7.294890, -7.111944, -6.627079, -6.8...
## $ V3 <dbl> -6.171100, -6.171100, -6.244421, -6.264807, -6.517780, -6.9...
## $ V4 <dbl> -6.410820, -6.410820, -7.271515, -7.203928, -6.642947, -6.8...
## $ V5 <dbl> -7.207767, -7.207767, -6.691354, -6.910962, -6.561653, -6.9...
## $ V6 <dbl> -7.292158, -7.292158, -6.377044, -6.649427, -6.517617, -6.9...
## $ V7 <dbl> -9.628800, -9.628800, -9.094889, -8.337271, -8.511515, -7.6...
```

Deviance is the sum of the occasion-level LLs multiplied by -2.


```r
dfLL <-
  dfLL %>%
  mutate(sums     = rowSums(.),
         deviance = -2*sums)
```

Because we used HMC, deviance is a distribution rather than a single number.


```r
quantile(dfLL$deviance, c(.025, .5, .975))
```

```
##      2.5%       50%     97.5% 
##  95.14044  97.55311 105.02463
```

```r
ggplot(data = dfLL, aes(x = deviance)) +
  theme_classic() +
  geom_density(fill = "plum", size = 0) +
  geom_vline(xintercept = quantile(dfLL$deviance, c(.025, .5, .975)),
             color = "plum4", linetype = c(2, 1, 2)) +
  scale_x_continuous(breaks = quantile(dfLL$deviance, c(.025, .5, .975)),
                     labels = c(95, 98, 105)) +
  scale_y_continuous(NULL, breaks = NULL) +
  labs(title = "The deviance distribution",
       subtitle = "The dotted lines are the 95% intervals and\nthe solid line is the median.") +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

## 6.3. Regularization

In case you were curious, here's how you might do Figure 6.8 with ggplot2. All the action is in the `geom_line()` portions. The rest is window dressing.


```r
tibble(x = seq(from = - 3.5, 
               to = 3.5, 
               by = .01)) %>%
  
  ggplot(aes(x = x)) +
  theme_classic() +
  geom_ribbon(aes(ymin = 0, ymax = dnorm(x, mean = 0, sd = 1)), 
              fill = "plum", alpha = 1/8) +
  geom_ribbon(aes(ymin = 0, ymax = dnorm(x, mean = 0, sd = 0.5)), 
              fill = "plum", alpha = 1/8) +
  geom_ribbon(aes(ymin = 0, ymax = dnorm(x, mean = 0, sd = 0.2)), 
              fill = "plum", alpha = 1/8) +
  geom_line(aes(y = dnorm(x, mean = 0, sd = 1)), 
            linetype = 2, color = "plum4") +
  geom_line(aes(y = dnorm(x, mean = 0, sd = 0.5)), 
            size = .25, color = "plum4") +
  geom_line(aes(y = dnorm(x, mean = 0, sd = 0.2)), 
            color = "plum4") +
  scale_y_continuous(NULL, breaks = NULL) +
  labs(x = "parameter value") +
  coord_cartesian(xlim = c(-3, 3)) +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

### 6.4.1. DIC.

##### Overthinking: WAIC calculation. 

Here is how to fit the pre-WAIC model in brms.


```r
data(cars)

b <- 
  brm(data = cars, family = gaussian,
      dist ~ 1 + speed,
      prior = c(set_prior("normal(0, 100)", class = "Intercept"),
                set_prior("normal(0, 10)", class = "b"),
                set_prior("uniform(0, 30)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4)

print(b) 
```

```
##  Family: gaussian 
##   Links: mu = identity; sigma = identity 
## Formula: dist ~ 1 + speed 
##    Data: cars (Number of observations: 50) 
## Samples: 4 chains, each with iter = 2000; warmup = 1000; thin = 1; 
##          total post-warmup samples = 4000
##     ICs: LOO = NA; WAIC = NA; R2 = NA
##  
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## Intercept   -17.44      6.89   -30.69    -3.89       2056 1.00
## speed         3.92      0.42     3.07     4.73       1971 1.00
## 
## Family Specific Parameters: 
##       Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## sigma    15.76      1.65    12.93    19.32       2149 1.00
## 
## Samples were drawn using sampling(NUTS). For each parameter, Eff.Sample 
## is a crude measure of effective sample size, and Rhat is the potential 
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

In brms, you get the loglikelihood with `log_lik()`.


```r
dfLL <-
  b %>%
  log_lik() %>%
  as_tibble()
```

Computing the lppd, the "Bayesian deviance", takes a bit of leg work.


```r
dfmean <-
  dfLL %>%
  exp() %>%
  summarise_all(mean) %>%
  gather(key, means) %>%
  select(means) %>%
  log()

(
  lppd <-
  dfmean %>%
  sum()
)
```

```
## [1] -206.6634
```

Comupting the effective number of parameters, *p*~WAIC~, isn't much better.


```r
dfvar <-
  dfLL %>%
  summarise_all(var) %>%
  gather(key, vars) %>%
  select(vars) 

pwaic <-
  dfvar %>%
  sum()

pwaic
```

```
## [1] 3.226673
```

```r
dfvar
```

```
## # A tibble: 50 x 1
##      vars
##     <dbl>
##  1 0.0213
##  2 0.0721
##  3 0.0208
##  4 0.0458
##  5 0.0126
##  6 0.0199
##  7 0.0126
##  8 0.0127
##  9 0.0275
## 10 0.0163
## # ... with 40 more rows
```

Finally, here's what we've been working so hard for: our hand calculated WAIC value. Compare it to the value returned by the brms `waic()` function.


```r
-2*(lppd - pwaic)
```

```
## [1] 419.7801
```

```r
waic(b)
```

```
## 
## Computed from 4000 by 50 log-likelihood matrix
## 
##           Estimate   SE
## elpd_waic   -209.9  6.4
## p_waic         3.2  1.2
## waic         419.8 12.7
```

Here's how we get the WAIC standard error.


```r
dfmean %>%
  mutate(waic_vec = -2*(means - dfvar$vars)) %>%
  summarise(waic_se = (var(waic_vec)*nrow(dfmean)) %>% sqrt())
```

```
## # A tibble: 1 x 1
##   waic_se
##     <dbl>
## 1    12.7
```

### 6.5.1. Model comparison. 

Getting the `milk` data from earlier in the text.


```r
library(rethinking)

data(milk)
d <- 
  milk %>%
  filter(complete.cases(.))
rm(milk)

d <-
  d %>%
  mutate(neocortex = neocortex.perc/100)
```

The dimensions of `d` are:


```r
dim(d)
```

```
## [1] 17  9
```

Here are our competing `kcal.per.g` models.


```r
detach(package:rethinking, unload = T)
library(brms)

Inits <- list(Intercept = mean(d$kcal.per.g),
              sigma = sd(d$kcal.per.g))

InitsList <-list(Inits, Inits, Inits, Inits)

b6.11 <- 
  brm(data = d, family = gaussian,
      kcal.per.g ~ 1,
      prior = c(set_prior("uniform(-1000, 1000)", class = "Intercept"),
                set_prior("uniform(0, 100)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4,
      inits = InitsList)

Inits <- list(Intercept = mean(d$kcal.per.g),
              neocortex = 0,
              sigma = sd(d$kcal.per.g))

b6.12 <- 
  brm(data = d, family = gaussian,
      kcal.per.g ~ 1 + neocortex,
      prior = c(set_prior("uniform(-1000, 1000)", class = "Intercept"),
                set_prior("uniform(-1000, 1000)", class = "b"),
                set_prior("uniform(0, 100)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4,
      inits = InitsList)

Inits <- list(Intercept = mean(d$kcal.per.g),
              `log(mass)` = 0,
              sigma = sd(d$kcal.per.g))

b6.13 <- 
  brm(data = d, family = gaussian,
      kcal.per.g ~ 1 + log(mass),
      prior = c(set_prior("uniform(-1000, 1000)", class = "Intercept"),
                set_prior("uniform(-1000, 1000)", class = "b"),
                set_prior("uniform(0, 100)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4,
      inits = InitsList)

Inits <- list(Intercept = mean(d$kcal.per.g),
              neocortex = 0,
              `log(mass)` = 0,
              sigma = sd(d$kcal.per.g))

b6.14 <- 
  brm(data = d, family = gaussian,
      kcal.per.g ~ 1 + neocortex + log(mass),
      prior = c(set_prior("uniform(-1000, 1000)", class = "Intercept"),
                set_prior("uniform(-1000, 1000)", class = "b"),
                set_prior("uniform(0, 100)", class = "sigma")),
      chains = 4, iter = 2000, warmup = 1000, cores = 4,
      inits = InitsList)
```

#### 6.5.1.1. Comparing WAIC values.

In brms, you can get a model's WAIC value with either `WAIC()` or `waic()`.


```r
WAIC(b6.14)
```

```
## 
## Computed from 4000 by 17 log-likelihood matrix
## 
##           Estimate  SE
## elpd_waic      8.4 2.6
## p_waic         3.1 0.8
## waic         -16.8 5.2
```

```
## Warning: 2 (11.8%) p_waic estimates greater than 0.4. We recommend trying
## loo instead.
```

```r
waic(b6.14)
```

```
## 
## Computed from 4000 by 17 log-likelihood matrix
## 
##           Estimate  SE
## elpd_waic      8.4 2.6
## p_waic         3.1 0.8
## waic         -16.8 5.2
```

```
## Warning: 2 (11.8%) p_waic estimates greater than 0.4. We recommend trying
## loo instead.
```

Note the warning messages. Statisticians have made notable advances in Bayesian information criteria since McElreath published *Statistical Rethinking*. I won’t go into detail here, but the "We recommend trying loo instead" part of the message is designed to prompt us to use a different information criteria, the Pareto smoothed importance-sampling leave-one-out cross-validation (PSIS-LOO; aka, the LOO).  In brms, this is available with the `loo()` function, which you can learn more about in [this vignette](https://cran.r-project.org/web/packages/loo/vignettes/loo2-example.html) from the makers of the [loo package](https://cran.r-project.org/web/packages/loo/index.html). For now, back to the WAIC.

There are two basic ways to compare WAIC values from multiple models. In the first, you add more model names into the `waic()` function.


```r
waic(b6.11, b6.12, b6.13, b6.14)
```

```
##                 WAIC   SE
## b6.11          -8.64 3.69
## b6.12          -7.11 3.29
## b6.13          -8.99 4.20
## b6.14         -16.78 5.17
## b6.11 - b6.12  -1.53 1.12
## b6.11 - b6.13   0.35 2.36
## b6.11 - b6.14   8.14 4.93
## b6.12 - b6.13   1.88 3.02
## b6.12 - b6.14   9.67 5.07
## b6.13 - b6.14   7.78 3.50
```

Alternatively, you first save each model's `waic()` output in its own object, and then feed to those objects into `compare_ic()`.


```r
w.b6.11 <- waic(b6.11)
w.b6.12 <- waic(b6.12)
w.b6.13 <- waic(b6.13)
w.b6.14 <- waic(b6.14)

compare_ic(w.b6.11, w.b6.12, w.b6.13, w.b6.14)
```

```
##                 WAIC   SE
## b6.11          -8.64 3.69
## b6.12          -7.11 3.29
## b6.13          -8.99 4.20
## b6.14         -16.78 5.17
## b6.11 - b6.12  -1.53 1.12
## b6.11 - b6.13   0.35 2.36
## b6.11 - b6.14   8.14 4.93
## b6.12 - b6.13   1.88 3.02
## b6.12 - b6.14   9.67 5.07
## b6.13 - b6.14   7.78 3.50
```

If you want to get those WAIC weights, you can use the `brms::model_weights()` function like so:


```r
model_weights(b6.11, b6.12, b6.13, b6.14, 
              weights = "waic") %>% 
  round(digits = 2)
```

```
## b6.11 b6.12 b6.13 b6.14 
##  0.02  0.01  0.02  0.96
```

That last `round()` line was just to limit the decimal-place precision. If you really wanted to go through the trouble, you could make yourself a little table like this:


```r
model_weights(b6.11, b6.12, b6.13, b6.14, 
              weights = "waic") %>% 
  as_tibble() %>% 
  rename(weight = value) %>% 
  mutate(model = c("b6.11", "b6.12", "b6.13", "b6.14"),
         weight = weight %>% round(digits = 2)) %>% 
  select(model, weight) %>% 
  arrange(desc(weight))
```

```
## # A tibble: 4 x 2
##   model weight
##   <chr>  <dbl>
## 1 b6.14 0.960 
## 2 b6.11 0.0200
## 3 b6.13 0.0200
## 4 b6.12 0.0100
```

I'm not aware of a convenient way to plot the WAIC comparisons of brms models the way McElreath does with rethinking. However, one can get the basic comparison plot with a little data processing. It helps to examine the structure of your WAIC objects. For example:


```r
glimpse(w.b6.11)
```

```
## List of 9
##  $ estimates   : num [1:3, 1:2] 4.32 1.346 -8.64 1.845 0.308 ...
##   ..- attr(*, "dimnames")=List of 2
##   .. ..$ : chr [1:3] "elpd_waic" "p_waic" "waic"
##   .. ..$ : chr [1:2] "Estimate" "SE"
##  $ pointwise   : num [1:17, 1:3] 0.266 0.149 0.565 -0.17 -0.433 ...
##   ..- attr(*, "dimnames")=List of 2
##   .. ..$ : NULL
##   .. ..$ : chr [1:3] "elpd_waic" "p_waic" "waic"
##  $ elpd_waic   : num 4.32
##  $ p_waic      : num 1.35
##  $ waic        : num -8.64
##  $ se_elpd_waic: num 1.84
##  $ se_p_waic   : num 0.308
##  $ se_waic     : num 3.69
##  $ model_name  : chr "b6.11"
##  - attr(*, "dims")= int [1:2] 4000 17
##  - attr(*, "class")= chr [1:3] "ic" "waic" "loo"
```

We can index the point estimate for model `b6.11`'s WAIC as `w.b6.11$estimates["waic", "Estimate"]` and the standard error as `w.b6.11$estimates["waic", "SE"]`.


```r
w.b6.11$estimates["waic", "Estimate"]
```

```
## [1] -8.640175
```

```r
w.b6.11$estimates["waic", "SE"]
```

```
## [1] 3.689496
```

Alternatively, you could do `w.b6.11$estimates[3, 1]` and `w.b6.11$estimates[3, 2]`.

Armed with that information, we can make a data structure with those bits from all our models and then make a plot with the help of `ggplot2::geom_pointrange()`.


```r
tibble(model = c("b6.11", "b6.12", "b6.13", "b6.14"),
       waic = c(w.b6.11$estimates[3, 1], w.b6.12$estimates[3, 1], w.b6.13$estimates[3, 1], w.b6.14$estimates[3, 1]),
       se = c(w.b6.11$estimates[3, 2], w.b6.12$estimates[3, 2], w.b6.13$estimates[3, 2], w.b6.14$estimates[3, 2])) %>%
  
  ggplot() +
  theme_classic() +
  geom_pointrange(aes(x = model, y = waic, 
                      ymin = waic - se, 
                      ymax = waic + se),
                  shape = 21, color = "plum4", fill = "plum") +
  coord_flip() +
  labs(x = NULL, y = NULL,
       title = "My custom WAIC plot") +
  theme(text = element_text(family = "Courier"),
        axis.ticks.y = element_blank())
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-24-1.png)<!-- -->

We briefly discussed the alternative information criteria, the LOO, above. Here’s how to use it in brms.


```r
LOO(b6.11)
```

```
## 
## Computed from 4000 by 17 log-likelihood matrix
## 
##          Estimate  SE
## elpd_loo      4.3 1.9
## p_loo         1.4 0.3
## looic        -8.6 3.7
## ------
## Monte Carlo SE of elpd_loo is 0.0.
## 
## All Pareto k estimates are good (k < 0.5).
## See help('pareto-k-diagnostic') for details.
```

```r
loo(b6.11)
```

```
## 
## Computed from 4000 by 17 log-likelihood matrix
## 
##          Estimate  SE
## elpd_loo      4.3 1.9
## p_loo         1.4 0.3
## looic        -8.6 3.7
## ------
## Monte Carlo SE of elpd_loo is 0.0.
## 
## All Pareto k estimates are good (k < 0.5).
## See help('pareto-k-diagnostic') for details.
```

The Pareto $k$ values are a useful model fit diagnostic tool, which we’ll discuss later. But for now, realize that brms uses functions from the [loo package](https://cran.r-project.org/web/packages/loo/index.html) to compute its WAIC and LOO values. In addition to the vignette, above, [this vignette](https://cran.r-project.org/web/packages/loo/vignettes/loo2-weights.html) demonstrates the LOO with these very same examples from McElreath's text. And if you'd like to dive a little deeper, check out [Aki Veharti's GPSS2017 workshop](https://www.youtube.com/watch?v=8_Su5Qo49Dg&t).

#### 6.5.1.2. Comparing estimates.

The brms package doesn't have anything like rethinking's `coeftab()` function. However, one can get that information with a little ingenuity. For this, we'll employ the [broom package](https://cran.r-project.org/web/packages/broom/index.html), which provides an array of convenience functions to convert statistical analysis summaries into tidy data objects. Here we'll employ the `tidy()` function, which will save the summary statistics for our model parameters. For example, this is what it will produce for the full model, `b6.14`.


```r
# install.packages("broom", dependendcies = T)
library(broom)

tidy(b6.14)
```

```
##          term     estimate  std.error       lower        upper
## 1 b_Intercept  -1.06972723 0.59410620  -2.0314058  -0.08565205
## 2 b_neocortex   2.76855714 0.92158083   1.2614717   4.24874470
## 3   b_logmass  -0.09584749 0.02824901  -0.1429289  -0.04967985
## 4       sigma   0.13910517 0.03006200   0.1002100   0.19341450
## 5        lp__ -19.17534178 1.66038200 -22.3662856 -17.28467262
```

Note, `tidy()` also grabs the log posterior (i.e., "lp__"), which we'll exclude for our purposes. With a `rbind()` and a little indexing, we can save the summaries for all four models in a single tibble.


```r
my_coef_tab <-
  rbind(tidy(b6.11), tidy(b6.12), tidy(b6.13), tidy(b6.14)) %>%
  mutate(model = c(rep("b6.11", times = nrow(tidy(b6.11))),
                   rep("b6.12", times = nrow(tidy(b6.12))),
                   rep("b6.13", times = nrow(tidy(b6.13))),
                   rep("b6.14", times = nrow(tidy(b6.14))))
         ) %>%
  filter(term != "lp__") %>%
  select(model, everything())

head(my_coef_tab)
```

```
##   model        term  estimate  std.error      lower     upper
## 1 b6.11 b_Intercept 0.6582530 0.04675229  0.5821289 0.7357594
## 2 b6.11       sigma 0.1894174 0.03822525  0.1388652 0.2568818
## 3 b6.12 b_Intercept 0.3500599 0.54150051 -0.5416017 1.2516576
## 4 b6.12 b_neocortex 0.4549890 0.79846163 -0.8580444 1.7646954
## 5 b6.12       sigma 0.1925202 0.03981026  0.1405420 0.2663468
## 6 b6.13 b_Intercept 0.7052856 0.05681952  0.6149343 0.7997907
```

Just a little more work and we'll have a table analogous to the one McElreath produced with his `coef_tab()` function.


```r
my_coef_tab %>%
  # Learn more about dplyr::complete() here: https://rdrr.io/cran/tidyr/man/expand.html
  complete(term = distinct(., term), model) %>%
  select(model, term, estimate) %>%
  mutate(estimate = round(estimate, digits = 2)) %>%
  spread(key = model, value = estimate)
```

```
##          term b6.11 b6.12 b6.13 b6.14
## 1 b_Intercept  0.66  0.35  0.71 -1.07
## 2   b_logmass    NA    NA -0.03 -0.10
## 3 b_neocortex    NA  0.45    NA  2.77
## 4       sigma  0.19  0.19  0.18  0.14
```

I'm also not aware of an efficient way in brms to reproduce Figure 6.12 for which McElreath nested his `coeftab()` argument in a `plot()` argument. However, one can build something similar by hand with a little data wrangling.


```r
p11 <- posterior_samples(b6.11)
p12 <- posterior_samples(b6.12)
p13 <- posterior_samples(b6.13)
p14 <- posterior_samples(b6.14)

# This block is just for intermediary information
# colnames(p11)
# colnames(p12)
# colnames(p13)
# colnames(p14)

# Here we put it all together
tibble(mdn = c(NA, median(p11[, 1]), median(p12[, 1]), median(p13[, 1]), median(p14[, 1]),
               NA, NA, median(p12[, 2]), NA, median(p14[, 2]),
               NA, median(p11[, 2]), NA, median(p13[, 2]), NA,
               NA, median(p11[, 2]), median(p12[, 3]), median(p13[, 3]), median(p14[, 4])),
       sd  = c(NA, sd(p11[, 1]), sd(p12[, 1]), sd(p13[, 1]), sd(p14[, 1]),
               NA, NA, sd(p12[, 2]), NA, sd(p14[, 2]),
               NA, sd(p11[, 2]), NA, sd(p13[, 2]), NA,
               NA, sd(p11[, 2]), sd(p12[, 3]), sd(p13[, 3]), sd(p14[, 4])),
       order = c(20:1)) %>%
  
  ggplot(aes(x = mdn, y = order)) +
  theme_classic() +
  geom_hline(yintercept = 0, color = "plum4", alpha = 1/8) +
  geom_pointrange(aes(x = order, y = mdn, 
                      ymin = mdn - sd, 
                      ymax = mdn + sd),
                  shape = 21, color = "plum4", fill = "plum") +
  scale_x_continuous(breaks = 20:1,
                     labels = c("intercept", 11:14,
                                "neocortex", 11:14,
                                "logmass", 11:14,
                                "sigma", 11:14)) +
  coord_flip() +
  labs(x = NULL, y = NULL,
       title = "My custom coeftab() plot") +
  theme(text = element_text(family = "Courier"),
        axis.ticks.y = element_blank())
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-29-1.png)<!-- -->

Making that plot entailed a lot of hand typing values in the tibble, which just begs for human error. If possible, it's better to use functions in a principled way to produce the results. Below is such an attempt.


```r
my_coef_tab <-
  my_coef_tab %>%
  complete(term = distinct(., term), model) %>%
  rbind(
     tibble(
       model = NA,
       term = c("b_logmass", "b_neocortex", "sigma", "b_Intercept"),
       estimate = NA,
       std.error = NA,
       lower = NA,
       upper = NA)) %>%
  mutate(axis = ifelse(is.na(model), term, model),
         model = factor(model, levels = c("b6.11", "b6.12", "b6.13", "b6.14")),
         term = factor(term, levels = c("b_logmass", "b_neocortex", "sigma", "b_Intercept", NA))) %>%
  arrange(term, model) %>%
  mutate(axis_order = letters[1:20],
         axis = ifelse(str_detect(axis, "b6."), str_c("      ", axis), axis))
  
ggplot(data = my_coef_tab,
       aes(x = axis_order,
           y = estimate,
           ymin = lower,
           ymax = upper)) +
  theme_classic() +
  geom_hline(yintercept = 0, color = "plum4", alpha = 1/8) +
  geom_pointrange(shape = 21, color = "plum4", fill = "plum") +
  scale_x_discrete(NULL, labels = my_coef_tab$axis) +
  ggtitle("My other coeftab() plot") +
  coord_flip() +
  theme(text = element_text(family = "Courier"),
        panel.grid = element_blank(),
        axis.ticks.y = element_blank(),
        axis.text.y = element_text(hjust = 0))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-30-1.png)<!-- -->

I'm sure there are better ways to do this. Have at it.

### 6.5.2. Model averaging.

Within the current brms framework, you can do model-averaged predictions with the `pp_average()` function. The default weighting scheme is with the LOO. Here we'll use the `weights = "waic"` argument to match McElreath's method in the text. Because `pp_average()` yields a matrix, we'll want to convert it to a tibble before feeding it into ggplot2.


```r
nd <- 
  tibble(neocortex = seq(from = .5, to = .8, length.out = 30),
         mass = rep(4.5, times = 30))

ftd <-
  fitted(b6.14, newdata = nd) %>%
  as_tibble() %>%
  bind_cols(nd)

pp_average(b6.11, b6.12, b6.13, b6.14,
           weights = "waic",
           method = "fitted",  # for new data predictions, use method = "predict"
           newdata = nd) %>%
  as_tibble() %>%
  bind_cols(nd) %>%
  
  ggplot(aes(x = neocortex, y = Estimate)) +
  theme_classic() +
  geom_ribbon(aes(ymin = `2.5%ile`, ymax = `97.5%ile`), 
              fill = "plum", alpha = 1/3) +
  geom_line(color = "plum2") +
  geom_ribbon(data = ftd, aes(ymin = `2.5%ile`, ymax = `97.5%ile`),
              fill = "transparent", color = "plum3", linetype = 2) +
  geom_line(data = ftd,
              color = "plum3", linetype = 2) +
  geom_point(data = d, aes(x = neocortex, y = kcal.per.g), 
             size = 2, color = "plum4") +
  labs(y = "kcal.per.g") +
  coord_cartesian(xlim = range(d$neocortex), 
                  ylim = range(d$kcal.per.g)) +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-31-1.png)<!-- -->

##### Bonus: $R^2$ talk

At the beginning of the chapter (pp. 167--168), McElreath briefly introduced $R^2$ as a popular way to assess the variance explained in a model. He pooh-poohed it because of its tendency to overfit. It's also limited in that it doesn't generalize well outside of the single-level Gaussian framework. However, if you should find yourself in a situation where $R^2$ suits your purposes, the brms `bayes_R2()` function might be of use. Simply feeding a model brm fit object into `bayes_R2()` will return the posterior mean, SD, and 95% intervals. For example:


```r
bayes_R2(b6.14) %>% round(digits = 3)
```

```
##    Estimate Est.Error 2.5%ile 97.5%ile
## R2    0.496     0.131   0.165    0.665
```

With just a little data processing, you can get a table of each of models' $R^2$ `Estimate`.


```r
rbind(bayes_R2(b6.11), 
      bayes_R2(b6.12), 
      bayes_R2(b6.13), 
      bayes_R2(b6.14)) %>%
  as_tibble() %>%
  mutate(model = c("b6.11", "b6.12", "b6.13", "b6.14"),
         r_square_posterior_mean = round(Estimate, digits = 2)) %>%
  select(model, r_square_posterior_mean)
```

```
## # A tibble: 4 x 2
##   model r_square_posterior_mean
##   <chr>                   <dbl>
## 1 b6.11                  0     
## 2 b6.12                  0.0700
## 3 b6.13                  0.150 
## 4 b6.14                  0.500
```

If you want the full distribution of the $R^2$, you’ll need to add a `summary = F` argument. Note how this returns a numeric vector.


```r
b6.13.R2 <- bayes_R2(b6.13, summary = F)

b6.13.R2 %>%
  glimpse()
```

```
##  num [1:4000, 1] 0.166 0.138 0.156 0.241 0.029 ...
##  - attr(*, "dimnames")=List of 2
##   ..$ : NULL
##   ..$ : chr "R2"
```

If you want to use these in ggplot2, you’ll need to put them in tibbles or data frames. Here we do so for two of our model fits.


```r
# model b6.13
b6.13.R2 <- 
  bayes_R2(b6.13, summary = F) %>%
  as_tibble() %>%
  rename(R2.13 = R2)

# model b6.14
b6.14.R2 <- 
  bayes_R2(b6.14, summary = F) %>%
  as_tibble() %>%
  rename(R2.14 = R2)

# Let's put them in the same data object
combined_R2s <-
  bind_cols(b6.13.R2, b6.14.R2) %>%
  mutate(dif = R2.14 - R2.13)

# A simple density plot
combined_R2s %>%
  ggplot(aes(x = R2.13)) +
  theme_classic() +
  geom_density(size = 0, fill = "plum1", alpha = 2/3) +
  geom_density(aes(x = R2.14),
               size = 0, fill = "plum2", alpha = 2/3) +
  scale_y_continuous(NULL, breaks = NULL) +
  coord_cartesian(xlim = 0:1) +
  labs(x = NULL,
       title = expression(paste(italic("R")^{2}, " distributions")),
       subtitle = "Going from left to right, these are for\nmodels b6.13 and b6.14.") +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-35-1.png)<!-- -->

If you do your work in a field where folks use $R^2$ change, you might do that with a simple difference score, which we computed above with `mutate(dif = R2.14 - R2.13)`. Here's the $R^2$ change (i.e., `dif`) plot:


```r
combined_R2s %>%
  ggplot(aes(x = dif)) +
  theme_classic() +
  geom_density(size = 0, fill = "plum") +
  geom_vline(xintercept = quantile(combined_R2s$dif, 
                                   probs = c(.025, .5, .975)),
             color = "white", size = c(1/2, 1, 1/2)) +
  scale_y_continuous(NULL, breaks = NULL) +
  labs(x = expression(paste(Delta, italic("R")^{2})),
       subtitle = "This is how much more variance, in terms\nof %, model b6.14 explained compared to\nmodel b6.13. The white lines are the\nposterior median and 95% percentiles.") +
  theme(text = element_text(family = "Courier"))
```

![](Ch._06_Overfitting,_Regularization,_and_Information_Criteria_files/figure-html/unnamed-chunk-36-1.png)<!-- -->

The brms package did not get these $R^2$ values by traditional method used in, say, ordinary least squares estimation. To learn more about how the Bayesian $R^2$ sausage is made, check out the paper by [Gelman, Goodrich, Gabry, and Ali](https://github.com/jgabry/bayes_R2/blob/master/bayes_R2.pdf).



Note. The analyses in this document were done with:

* R          3.4.4
* RStudio    1.1.442
* rmarkdown  1.9
* tidyverse  1.2.1
* ggrepel    0.7.0
* brms       2.2.0
* rethinking 1.59
* rstan      2.17.3
* broom      0.4.3

## Reference
McElreath, R. (2016). *Statistical rethinking: A Bayesian course with examples in R and Stan.* Chapman & Hall/CRC Press.
