![collaboratory logo](../../Misc-files/collaboratory2.png)

## [Collaboratory@Columbia](http://collaboratory.columbia.edu/)
### Columbia [Data Science Institute](http://datascience.columbia.edu/) and [Columbia Enterpreneuship](http://entrepreneurship.columbia.edu/)
## Winter 2018-2019 Data Science Bootcamp
### Day 2 Statistical Methods for Causal Reasoning with Observational Data
### Lab 1: Causal Inference EDAV

The 2018 Atlantic Causal Inference [Data Challenge](https://www.synapse.org/#!Synapse:syn11294478/wiki/486304) was a competition to evaluate different automated methods for causal inference in a simulated environment. One issue that arose over the course of the challenge was that not every data set met the foundational assumptions for causal inference: SUTVA, ignorability, and overlap. In this lab, you'll explore the 2018 sample files through visualizations. By the end you should be able to understand the fundamental problem of causal inference as it relates to data, identify obvious violations of foundational assumptions from the potential outcomes, and identify various sorts of causal variables.

The files for this lab can be found [here](https://stat.columbia.edu/~vincent/bootcamp). Each file contains a practice simulation from the contest and columns:

1. `z` - treatment variable, 0 is control and 1 is treated
2. `y` - the observed response variable
3. `y0` - the response for unit i under the control condition
4. `y1` - the response for unit i under the treated condition
5. `p.val` - the probability that unit i receives the treatment (this was not given and has been estimated)
6. a number of potential confounders

Pair up with someone and pick any file to explore, but I recommend the following (in order) 22, 8, 10, 19, 4. 

1. Download any file and load it into R or python
2. Look at the first few rows of the data (`head`)
3. Produce a plot of the potential outcomes, `y0` and `y1`, using a different plot symbol or color to differentiate between the treated and control observations (`z == 1`)
4. Produce a plot of the treatment effect `y1 - y0` over the probability of treatment `p.val`
  * Does the treatment effect appear to be constant?
  * For every treated unit, is there a comparable control? For every control, is there a comparable treated?
  * Relatedly, are there any observations whose probability of being treated is close to 0 or 1?
  * Does the effect seem estimable, and would you need any further assumptions to recover it?
  * Are all of the fundamental assumptions necessary for causal inference met?

Think about how the above would look if you didn't have access to the potential outcomes (there is no right answer here). A typical analysis would proceed by trying to identify which of the covariates are useful. A very rough way to do so is to calculate the univariate correlations between every variable and threatment/response, but note that uncorrelated does not mean independent! If you're using `R`, the following code snippet might be helpful:

```R
getCorrelations <- function(data) {
  res <- t(apply(data[,!(colnames(data) %in% c("z", "y", "y0", "y1", "p.val"))], 2, {
    function(col) c(cor(col, data$z), cor(col, data$y), cor(col, data$y0), cor(col, data$y1))} ))
  colnames(res) <- c("z", "y", "y0", "y1")
  res
}
```

Try to classify the variables as confounders, instruments, predictors, or unrelated. If there are no confounders, even if the potential outcomes look viable the problem cannot be solved!

1. For any variable of interest, produce a plot of the observed outcomes against it

For a "solution", see [here](lab_thoughts.md)
