![collaboratory logo](../../Misc-files/collaboratory2.png)

## [Collaboratory@Columbia](http://collaboratory.columbia.edu/)
### Columbia [Data Science Institute](http://datascience.columbia.edu/) and [Columbia Entrepreneurship](http://entrepreneurship.columbia.edu/)
## Winter 2018-2019 Data Science Bootcamp
### Day 2 Statistical Methods for Causal Reasoning with Observational Data
### Lab 2: Causal Inference Methods

A standard dataset in causal inference was produced by Lalonde ("Evaluating the Econometric Evaluations of Training Programs with Experimental Data", 1986) to evaluate the effectiveness of a job training program. In it, a randomized experiment was paired with an observational study so that it is possible to know the "true" treatment effect (although it may still not be possible to recover that estimate from the observational component due to confounding). The dataset we'll use here is the a smaller subset in Dehejia, R.H. and Wahba, S. (Causal Effects in Nonexperimental Studies: Reevaluating the Evaluation of Training Programs," 1999) Journal of the American Statistical Association 94, 1053-1062. Many causal inference packages in `R` come with the dataset; the one we'll use is in the `MatchIt` package. It takes 185 treated observations from the randomized experiment and combines them with 429 controls from a non-experimental comparison group. The variables are:

1. `treat`: 1 = treatment group, 0 = control group
2. `age`: age in years
3. `educ`: years of schooling
4. `black`: 1 if black, 0 otherwise
5. `hispan`: 1 if Hispanic, 0 otherwise
6. `married`: 1 if married, 0 otherwise
7. `nodegree`: 1 if high school diploma, 0 otherwise
8. `re74`, `re75`, `re78`: real earnings in the respective year

The treatment took place during 1976 to 1977s, so that `re74`, `re75` are not colliders. The outcome of interest is often assumed to be `re78`. The randomized experiment estimated the true effect to be $1,794. Proceed by entering the following code and responding to a prompt if you feel like it.

#### Preliminaries

Install the `MatchIt` package to perform various matching functions and the `cobalt` package to visualize balance:

```R
if (require(MatchIt, quietly = TRUE) == FALSE) {
  install.packages("MatchIt")
  require(MatchIt, quietly = TRUE)
}

if (require(cobalt, quietly = TRUE) == FALSE) {
  install.packages("cobalt")
  require(cobalt, quietly = TRUE)
}

data(lalonde, package = "MatchIt")
```

#### First Look, Naive Estimates

```R
# generate a naive treatment effect estimate
with(lalonde, mean(re78[treat == 1]) - mean(re78[treat == 0]))

# control for obvious confounders, but do a regression model
# with a constant treatment effect (the model just contains "treat"), the effect
# itself is just the regression coefficient and can be read off of the summary
summary(lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
           re74 + re75, data = lalonde))
```

Is there evidence that the response varies non-linearly by age or that there are interactions with any of the other variables? In their analysis Dehejia and Wahba included a quadratic term for age, which you can do in `R` by adding to the formula the term `I(age^2)`.

#### Look at Imbalance

```R
# are there imbalances in distributions?
with(lalonde, t.test(age[treat == 1], age[treat == 0]))

# one way to assess imbalance is to use the cobalt package
balanceSub <- lalonde[,!(colnames(lalonde) %in% c("treat", "re78"))]
pre.imbalance <- bal.tab(balanceSub, treat = lalonde$treat, estimand = "ate")
pre.imbalance
plot(pre.imbalance)
```

`bal.tab` comes from the `cobalt` package, and includes a number of options for conducting tests of balance. It's not very useful right now since we only have the unadjusted differences, but it's output could be modified to include tests of differences in distributions. See `?bal.tab` for info. For example, adding `ks.threshold = 0.05` to the call would cause the software to compute Komolgorov-Smirnov statistics.

#### Exact Match

The function `matchit` comes from the `MatchIt` package and has a number of options, including `method` ("exact", "full", "genetic", "nearest", "optimal", and "subclass") and `distance` ("logit", "mahalanobis", a numeric vector, and others).

```R
# perform an exact match
exact.match <-
  MatchIt::matchit(treat ~ age + educ + black + hispan + married + nodegree +
                   re74 + re75, data = lalonde, method = "exact")

# note how few observations there are
summary(exact.match)

exact.data <- match.data(exact.match)
head(exact.data)
nrow(exact.data)

# compute a regression estimate using just the exact matches
summary(lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
           re74 + re75, data = exact.data, weights = exact.data$weights))
```

Take a look at the exact match data to see what was left over. It should look very different from the original data (but it has great balance!).

#### Propensity Score Estimation

```R
p.score <-
  glm(treat ~ age + educ + black + hispan + married + nodegree +
      re74 + re75, family = binomial, data = lalonde)$fitted.values

ate.weights <- with(lalonde, treat / p.score + (1 - treat) / (1 - p.score))

# see how it impacts balance
plot(bal.tab(balanceSub, treat = lalonde$treat, weights = ate.weights, method = "weighting", estimand = "ate"))

# iptw estimate
summary(lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
           re74 + re75, data = lalonde, weights = ate.weights))
```

Another concern is again how the estimates change if we complexify the model. Try adding `I(age^2)` or an interaction to one or other of the treatment assignment mechanism or the response surface model.


```R
# try again with stabilized weights
stabilized.weights <- with(lalonde, mean(treat) * treat / p.score + (1 - mean(treat)) * (1 - treat) / (1 - p.score))

plot(bal.tab(balanceSub, treat = lalonde$treat, weights = stabilized.weights, method = "weighting", estimand = "ate"))

summary(lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
           re74 + re75, data = lalonde, weights = stabilized.weights))
```

Note that the stabilized weights don't really seem to change the balance picture but do change the estimate.

One thing we haven't talked much about is what estimand is of interest. The above computes weights useful to compute the average treatment effect (ATE) across the sample, but we could also look at the average treatment on the treated (ATT). Because the treated units come from one population and the controls from another, this data set is ideal for trying to match controls to treated, or to compute the ATE. Compute those weights using the formula $$w = z + (1 - z) e / (1 - e)$$.

#### Nearest Neighbor Matching

Nearest neighbor matching can be used to compute the ATT by finding a single control observation for every treated.

```R
nearest.match <-
  matchit(treat ~ age + educ + black + hispan + married + nodegree +
          re74 + re75, data = lalonde, method = "nearest", distance = "logit", discard = "control")
nearest.imbalance <- bal.tab(nearest.match)
nearest.imbalance
plot(nearest.imbalance)
nearest.data <- match.data(nearest.match)
```

Nearest neighbor matching produces weights that are 0/1 unless we match the nearest K observations.

```R
## non-parametric estimate of the ATT
mean(nearest.data$re78[nearest.data$treat == 1]) -
     mean(nearest.data$re78[nearest.data$treat == 0])

## A model-based estimate of the ATT
summary(lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
           re74 + re75, data = nearest.data))
```

#### Further Options

At this point, it would be a good idea to play with the above. Some suggestions:

1. Explore the options for `matchit` and try different matching algorithms with different parameters. You can see those options by entering `?matchit`.
2. Try different ways of estimating the propensity score, for example using generalized additive models and the `mgcv` package.
3. Try a nonparametric approach such as using the package `tmle`.
4. Consider non-constant treatment effect models. You can modify any of the above by interacting `treat` with another variable (for example, adding `treat:married` to a formula). Calculating treatment effects becomes a bit harder, since you will then have to average the predictions across the sample population. For example:

```R
nonConstantFit <-
  lm(re78 ~ treat + age + educ + black + hispan + married + nodegree +
     re74 + re75 + treat:married, data = nearest.data)

nearest.data.trt <- nearest.data
nearest.data.trt$treat <- 1
nearest.data.ctl <- nearest.data
nearest.data.ctl$treat <- 0
mean(predict(nonConstantFit, nearest.data.trt) - predict(nonConstantFit, nearest.data.ctl))
```

#### Further Reading

The above was losely based off of:

* [Using the R MatchIt package for propensity score analysis by Matt Bogard](https://www.r-bloggers.com/using-the-r-matchit-package-for-propensity-score-analysis/)
* [Matching by Stephen Pettigrew](http://www.stephenpettigrew.com/teaching/gov2001/section11_2014.pdf)

However, there are a great number of analyses of the Lalonde data out there, including 

* [cobalt package vignette](https://cran.r-project.org/web/packages/cobalt/vignettes/cobalt_A0_basic_use.html)
* [MatchIt package vignette](https://cran.r-project.org/web/packages/MatchIt/vignettes/matchit.pdf)
* [Exploring propensity score matching and weighting by Peter's stats stuff](https://www.r-bloggers.com/exploring-propensity-score-matching-and-weighting/)

The last reference includes some useful code for analysis beyond matching.
