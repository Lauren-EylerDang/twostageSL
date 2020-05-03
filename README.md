
<!-- README.md is generated from README.Rmd. Please edit that file -->

# twostageSL: Two-stage Super Learner for predicting healthcare cost with zero-inflation

<!-- badges: start -->

<!-- badges: end -->

> Tools for analyzing healthcare cost data with zero inflation and
> skewed distribution via two stage super learner

**Author:** Ziyue Wu

## Motivation

The study of healthcare expenditures has become an important area in
epidemiological and public health researches as healthcare utilization
and associated costs have increased rapidly in recent years. Estimating
healthcare expenditures is challenging due to heavily skewed
distributions and zero-inflation. Myriad methods have been developed for
analyzing cost data, however, a priori determination of an appropriate
method is difficult. Super-learning, a technique that considers an
ensemble of methods for cost estimation, provides an interesting
alternative for modeling healthcare expenditures as it addresses the
difficulties in choosing the correct model form. The super learner has
shown benefits over a single method in recent studies. Nevertheless, it
is notable that none of the healthcare cost studies have explored the
use of super learner in the case of zero-inflated cost data. Here we
proposed a new method called the two-stage super learner that
implemented the super-learning in a two-part model framework. The
two-stage super learner had natural support for cases of zero healthcare
utilization and extended the standard super learner by providing an
additional layer of the ensemble, thus allowing for greater flexibility
and improving the cost prediction.

## Features

`twostageSL` is an R package that provides tools for analyzing health
care cost data useing the super learning technique in the two stage
framework.

  - Automatic optimal predictor ensembling via cross-validation with one
    line of code.
  - Dozens of algorithms: XGBoost, Random Forest, GBM, Lasso, SVM, BART,
    KNN, Decision Trees, Neural Networks, and additional algorithms
    specifically for heavily skewed zero-inflation data: Tweedie, Tobit,
    ZIP, Cox Hazard, Adaptive Hazard, Quantile Regression, Acclerated
    Failture Time.
  - Two options for conducting either a two stage super learner or a
    standard superlearner. Include two seperate libraries for prediction
    algorithms at two stages, accompanied with another library with
    prediction algorithms for standard superlearner.
  - Screen variables (feature selection) based on univariate
    association, Random Forest, Elastic Net, et al. or custom screening
    algorithms.
  - Includes framework to quickly add custom algorithms to the ensemble.

## Installation

twostageSL can be installed from Github using the following command:

``` r
# install.packages("devtools")

library(devtools)
devtools::install_github("wuziyueemory/twostageSL")

library(twostageSL)
```

## Example

To show an example on how to use this package, we simulated an example
with training data containing 1000 observations, 5 predictors and
testing data containing 100 observations, 5 predictions, all following a
normal distribution. The outcome is continuous and non-negative with
relatively high proportion of zeros.

Generate training and testing set.

``` r
library(twostageSL)
#> Loading required package: SuperLearner
#> Loading required package: nnls
#> Super Learner
#> Version: 2.0-25
#> Package created on 2019-08-05
#> Loading required package: slcost
set.seed(1)
## training set
n <- 1000
p <- 5
X <- matrix(rnorm(n*p), nrow = n, ncol = p)
colnames(X) <- paste("X", 1:p, sep="")
X <- data.frame(X)
Y <- rep(NA,n)
## probability of outcome being zero
prob <- plogis(1 + X[,1] + X[,2] + X[,1]*X[,2])
g <- rbinom(n,1,prob)
## assign zero outcome
ind <- g==0
Y[ind] <- 0
## assign non-zero outcome
ind <- g==1
Y[ind] <- 10 + X[ind, 1] + sqrt(abs(X[ind, 2] * X[ind, 3])) + X[ind, 2] - X[ind, 3] + rnorm(sum(ind))

## test set
m <- 100
newX <- matrix(rnorm(m*p), nrow = m, ncol = p)
colnames(newX) <- paste("X", 1:p, sep="")
newX <- data.frame(newX)
newY <- rep(NA,m)
## probability of outcome being zero
newprob <- plogis(1 + newX[,1] + newX[,2] + newX[,1]*newX[,2])
newg <- rbinom(n,1,newprob)
## assign zero outcome
newind <- newg==0
newY[newind] <- 0
## assign non-zero outcome
newind <- g==1
newY[newind] <- 10 + newX[newind, 1] + sqrt(abs(newX[newind, 2] * newX[newind, 3])) + newX[newind, 2] - X[newind, 3] + rnorm(sum(newind))
```

The training data looks like

``` r
head(X)
#>           X1          X2          X3         X4         X5
#> 1 -0.6264538  1.13496509 -0.88614959  0.7391149 -1.1346302
#> 2  0.1836433  1.11193185 -1.92225490  0.3866087  0.7645571
#> 3 -0.8356286 -0.87077763  1.61970074  1.2963972  0.5707101
#> 4  1.5952808  0.21073159  0.51926990 -0.8035584 -1.3516939
#> 5  0.3295078  0.06939565 -0.05584993 -1.6026257 -2.0298855
#> 6 -0.8204684 -1.66264885  0.69641761  0.9332510  0.5904787
```

The proportion of zeros in outcome Y.

``` r
mean(Y==0)
#> [1] 0.347
```

The distribution of outcome Y, which is zero-inflated and heavily
skewed.

``` r
hist(Y,breaks = 100,freq = FALSE,main="Distribution of Y")
```

<img src="man/figures/README-example 4-1.png" width="100%" />

`twostageSL` is the core function to fit the two stage super learner. At
a minimum for implementation, you need to specify the predictor matrix
X, outcome variable Y, library of prediction algorithms at two stages,
boolean for two stage superlearner or standard superlearner,
distribution at two stages and number of folds for cross-validation. The
package incldues all the prediction algorithms from package
`SuperLearner` and also includes additional prediction algorithms from
package `slcost` that specifically used at stage 2 to deal with heavily
skewed data.

Generate the library and run the two stage super learner

``` r
## generate the Library
twostage.library <- list(stage1=c("SL.glm","SL.earth","SL.randomForest"),
                       stage2=c("SL.glm","SL.earth","SL.randomForest","SL.coxph"))

onestage.library <- c("SL.glm","SL.earth","SL.randomForest")


## run the twostage super learner
two <- twostageSL(Y=Y,
                   X=X,
                   newX = newX,
                   library.2stage <- twostage.library,
                   library.1stage <- onestage.library,
                   twostage = TRUE,
                   family.1=binomial,
                   family.2=gaussian,
                   family.single=gaussian,
                   cvControl = list(V = 5))
#> Loading required package: quadprog
#> Loading required namespace: randomForest
#> Loading required namespace: earth
#> Loading required package: nloptr
two
#> 
#> Call:  
#> twostageSL(Y = Y, X = X, newX = newX, library.2stage = library.2stage <- twostage.library,  
#>     library.1stage = library.1stage <- onestage.library, twostage = TRUE,  
#>     family.1 = binomial, family.2 = gaussian, family.single = gaussian, cvControl = list(V = 5)) 
#> 
#> 
#> 
#>                                                       Risk       Coef
#> S1: SL.glm_All + S2: SL.glm_All                   20.86725 0.00000000
#> S1: SL.glm_All + S2: SL.earth_All                 20.93465 0.08384237
#> S1: SL.glm_All + S2: SL.randomForest_All          21.05839 0.00000000
#> S1: SL.glm_All + S2: SL.coxph_All                 20.84862 0.00000000
#> S1: SL.earth_All + S2: SL.glm_All                 19.11666 0.00000000
#> S1: SL.earth_All + S2: SL.earth_All               19.06665 0.15485605
#> S1: SL.earth_All + S2: SL.randomForest_All        19.06521 0.37516389
#> S1: SL.earth_All + S2: SL.coxph_All               19.09128 0.00000000
#> S1: SL.randomForest_All + S2: SL.glm_All          19.49827 0.00000000
#> S1: SL.randomForest_All + S2: SL.earth_All        19.35212 0.20370859
#> S1: SL.randomForest_All + S2: SL.randomForest_All 19.48207 0.01440149
#> S1: SL.randomForest_All + S2: SL.coxph_All        19.44992 0.13614912
#> Single: SL.glm_All                                21.12016 0.03187848
#> Single: SL.earth_All                              20.39265 0.00000000
#> Single: SL.randomForest_All                       19.99982 0.00000000
```

When setting `twostage` to FALSE, we get the result from standard super
learner

``` r
## run the standard super learner
one <- twostageSL(Y=Y,
                  X=X,
                  newX = newX,
                  library.2stage <- twostage.library,
                  library.1stage <- onestage.library,
                  twostage = FALSE,
                  family.1=binomial,
                  family.2=gaussian,
                  family.single=gaussian,
                  cvControl = list(V = 5))
one
#> 
#> Call:  
#> SuperLearner(Y = Y, X = X, family = family.single, SL.library = library.1stage,  
#>     method = method.CC_LS.scale, verbose = verbose, control = list(saveCVFitLibrary = T),  
#>     cvControl = cvControl) 
#> 
#> 
#>                         Risk      Coef
#> SL.glm_All          21.12016 0.1755772
#> SL.earth_All        20.39265 0.3737644
#> SL.randomForest_All 20.10009 0.4506584
```
