# Mixed Models with Rand Julia
A tutorial illustrating how to estimate linear mixed models using Julia in R.

# Overview

In this tutorial, I will show how to estimate linear-mixed models with `Julia` using `R`. I am assuming that the necessary libraries are already installed in Julia. Therefore, you should first install the following packages in Julia: `RCall` and `MixedModels`. More information can be found in this [link](https://rpubs.com/dmbates/377897).

# Outline


#### Prelim - Installing libraries used in this script (whenever is necessary).

```{r, echo=TRUE, warning=TRUE, results="hide", message=FALSE}
# This code chunk simply makes sure that all the libraries used here are installed.
packages <- c("lme4", "JuliaCall")
if ( length(missing_pkgs <- setdiff(packages, rownames(installed.packages()))) > 0) {
  message("Installing missing package(s): ", paste(missing_pkgs, collapse = ", "))
  install.packages(missing_pkgs)
}
```

#### Prelim - Loading libraries used in this script.

It is necessary to use the function `dyn.load` to load foreign function interfaces. Here, it is necessary to set the folder with the library `libopenlibm.DLL`.

```{r, echo=TRUE, warning=TRUE, results="hide", message=FALSE}
dyn.load("C:/Users/u0119584/AppData/Local/Programs/Julia 1.5.3/bin/libopenlibm.DLL")

library(JuliaCall)
library(lme4)
```

#### Prelim - Setting Julia.

Next, we need to call `Julia`. In some cases, it will only be necessary to set up `Julia` in `R` by using `julia_setup()`. In my case, I need to specify the folder in which Julia is located.

```{r, echo=TRUE, warning=TRUE, results="hide", message=FALSE}
julia.dir = julia_setup(JULIA_HOME = "C:\\Users\\u0119584\\AppData\\Local\\Programs\\Julia 1.5.3\\bin")
```


# 1. Estimate linear-mixed model.

### Calling Julia library MixedModels

First, we need to load the `Julia` library `MixedModels` using `julia.dir` and the function `library`:

```{r, echo=TRUE, warning=FALSE, eval=TRUE}
julia.dir$library("MixedModels")
```

### Model Specification.

As an illustration, I will use the *sleepstudy* dataset from the `lme4` library. To estimate a linear mixed effect model using `Julia` the variables defining the clustering (i.e., Subject) structure should be defined as a factor. 

```{r, echo=TRUE, warning=FALSE, eval=TRUE}
data(sleepstudy)
sleepstudy$Subject = as.factor(sleepstudy$Subject)
```

Second, we need to call the function in `R` to `Julia`, this can be done using the `Julia` function `assign`. With this function we call the data and the formula that specify the linear mixed-model we are interested in estimate:  


```{r, echo=TRUE, warning=FALSE, eval=TRUE}
julia.dir$assign("sleepstudy", sleepstudy)
julia.dir$assign("form", formula(Reaction ~ Days + (Days | Subject)))
```

### Model Estimation.

Finally, we estimate the model. To estimate the model we need to use the function `eval`. The first argument of the function includes the command to fit the linear mixed-model using the previously specified formula and dataset. The second argument states that the function will return the estimated results. Have in mind that the first time the following lines are run might take a few seconds. However, for the following analyses, the function will run faster. This is especially useful if you are planning to conduct simulations. 

```{r, echo=TRUE, warning=FALSE, eval=TRUE}
results = julia.dir$eval("res = fit(LinearMixedModel, form, sleepstudy)",need_return = c("Julia"))

# Get summary 
julia.dir$eval("res")

## Julia Object of type LinearMixedModel{Float64}.
## Linear mixed model fit by maximum likelihood
##  Reaction ~ 1 + Days + (1 + Days | Subject)
##    logLik   -2 logLik     AIC       AICc        BIC    
##   -875.9697  1751.9393  1763.9393  1764.4249  1783.0971
## 
## Variance components:
##             Column    Variance Std.Dev.   Corr.
## Subject  (Intercept)  565.51067 23.78047
##          Days          32.68212  5.71683 +0.08
## Residual              654.94145 25.59182
##  Number of obs: 180; levels of grouping factors: 18
## 
##   Fixed-effects parameters:
## --------------------------------------------------
##                 Coef.  Std. Error      z  Pr(>|z|)
## --------------------------------------------------
## (Intercept)  251.405      6.63226  37.91    <1e-99
## Days          10.4673     1.50224   6.97    <1e-11
## --------------------------------------------------

# Get the estimated fixed effects
beta.fixed = julia_eval("coef(res)")
beta.fixed 

## [1] 251.40510  10.46729

# Get the estimated random effects
beta.random = t(julia_eval("ranef(res)")[1])
beta.random

##             [,1]        [,2]
##  [1,]   2.815819   9.0755116
##  [2,] -40.048442  -8.6440794
##  [3,] -38.433064  -5.5133980
##  [4,]  22.832112  -4.6587173
##  [5,]  21.549840  -2.9444928
##  [6,]   8.815541  -0.2352007
##  [7,]  16.441908  -0.1588087
##  [8,]  -6.996671   1.0327272
##  [9,]  -1.037588 -10.5994157
## [10,]  34.666294   8.6323845
## [11,] -24.558026   1.0643762
## [12,] -12.334467   6.4716750
## [13,]   4.273998  -2.9553318
## [14,]  20.622181   3.5617128
## [15,]   3.258535   0.8717108
## [16,] -24.710141   4.6597008
## [17,]   0.723262  -0.9710526
## [18,]  12.118908   1.3106981
```
### Getting the summary of the linear-mixed models of Julia using R.

To get a summary in the same form as `lme4`, you can use the library `JellyMe4`. I am using here the code available [here](https://rdrr.io/github/palday/jlme/src/R/JellyMe4.R). First, we need to compile the function `jmer`:

```{r, echo=TRUE, warning=FALSE, eval=TRUE}
jmer <- function(formula, data, REML=TRUE){
    # to simplify maintainence here (in the hopes of turning this into a real
    # package), I'm depending on JellyMe4, which copies the dataframe back with
    # the model this is of course what you want if you're primarily working in
    # Julia and just using RCall for the the R ecosystem of extras for
    # MixedModels, but it does create an unnecessary copy if you're starting
    # with your data in R.
    #
    # Also, this means we suffer/benefit from the same level of compatibility in
    # the formula as in JellyMe4, i.e. currently no support for the ||

    jf <- deparse(formula,width = 500)
    jreml = ifelse(REML, "true", "false")

    julia_assign("jmerdat",data)
    julia_command(sprintf("jmermod = fit!(LinearMixedModel(@formula(%s),jmerdat),REML=%s);",jf,jreml))

    julia_eval("robject(:lmerMod, Tuple([jmermod,jmerdat]));",need_return="R")
}
```

We can estimate the linear mixed-model using `Julia` on `R`, and gettting the summary as an `lmer` object as follow:

```{r, echo=TRUE, warning=FALSE, eval=TRUE}
julia.dir$library("JellyMe4")
fit = jmer(formula(Reaction ~ Days + (Days | Subject)), sleepstudy, REML=TRUE)
summary(fit)

## Linear mixed model fit by REML ['lmerMod']
## Formula: Reaction ~ 1 + Days + (1 + Days | Subject)
##    Data: jellyme4_data
## Control: lmerControl(optimizer = "nloptwrap", optCtrl = list(maxeval = 1),  
##     calc.derivs = FALSE, check.nobs.vs.nRE = "warning")
## 
## REML criterion at convergence: 1743.6
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.9536 -0.4634  0.0231  0.4634  5.1793 
## 
## Random effects:
##  Groups   Name        Variance Std.Dev. Corr
##  Subject  (Intercept) 612.10   24.741       
##           Days         35.07    5.922   0.07
##  Residual             654.94   25.592       
## Number of obs: 180, groups:  Subject, 18
## 
## Fixed effects:
##             Estimate Std. Error t value
## (Intercept)  251.405      6.825  36.838
## Days          10.467      1.546   6.771
## 
## Correlation of Fixed Effects:
##      (Intr)
## Days -0.138
## optimizer (LN_BOBYQA) convergence code: 5 (fit with MixedModels.jl)
```

# 2. Conclusion.

This tutorial will hopefully be updated soon.

# 3. Acknowledgment.

I would like to thank Kristof Meers for suggesting me to use Julia and for kindly answer some silly and not so silly questions. 
