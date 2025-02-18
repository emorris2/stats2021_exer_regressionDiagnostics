# Preface

There is a lot here and this R notebook is busy. I was trying to compare
the advice and examples available from different sources. There is a lot
to be said about regression diagnostics. Needless to say, don’t worry
about trying to understand all of this in one week. See if you can
connect a couple dots between what is said in the texts and what is in
this notebook and call it good.

That being said, this is a great resource for future learning and
analysis should you choose to take it on. Keep it as an aid for
understanding regression in the future.

# Regression Diagnostics

The regression diagnostics available in Jamovi are still limited. It
would be a good idea to augment what is available in Jamovi with some
regression diagnostics in R. This R markdown document provides an
example of performing a multiple regression using the lm() function in R
and following up with some regression diagnostics to evaluate bias and
generalizability of the model.

#### Additional resources

-   Field has a good example of fitting a regression model and getting
    regression diagnostics in his book Discovering Statistics Using R.
    <https://archive.org/details/discoveringstatisticsusingr/page/n307/mode/2up>
-   Applied Statistics with R Ch. 13 (optional) available at:
    <https://daviddalpiaz.github.io/appliedstats/model-diagnostics.html>
-   R for Researchers OLS (optional) available at:
    <https://ssc.wisc.edu/sscc/pubs/RFR/RFR_Regression.html>
-   R for Researchers regression diagnostics (optional) available at:
    <https://ssc.wisc.edu/sscc/pubs/RFR/RFR_Diagnostics.html>
-   Diagnostics in multiple linear regression (optional) available at:
    <https://web.stanford.edu/class/stats191/notebooks/Diagnostics_for_multiple_regression.html>
-   Introduction to regression diagnostics (optional) available at:
    <https://stats.idre.ucla.edu/wp-content/uploads/2019/02/R_reg_part2.html#(1)>
-   Regression model diagnostics (optional) available at:
    <http://www.sthda.com/english/articles/39-regression-model-diagnostics/161-linear-regression-assumptions-and-diagnostics-in-r-essentials/>
-   Regression diagnostics (optional) available at:
    <https://www.statmethods.net/stats/rdiagnostics.html>
-   Outlier detection with Mahalanobis distance (optional) available at:
    <https://www.r-bloggers.com/outlier-detection-with-mahalanobis-distance/>
-   Multivariate outlier (optional) available at:
    <https://en.wikiversity.org/wiki/Multivariate_outlier>

## Package management in R

``` r
# keep a list of the packages used in this script
packages <- c("tidyverse","rio","jmv","boot","car","QuantPsyc","lmtest","faraway","broom")
```

This next code block has eval=FALSE because you don’t want to run it
when knitting the file. Installing packages when knitting an R notebook
can be problematic.

``` r
# check each of the packages in the list and install them if they're not installed already
for (i in packages){
  if(! i %in% installed.packages()){
    install.packages(i,dependencies = TRUE)
  }
  # show each package that is checked
  print(i)
}
```

``` r
# load each package into memory so it can be used in the script
for (i in packages){
  library(i,character.only=TRUE)
  # show each package that is loaded
  print(i)
}
```

## Regression Diagnostics

Regression diagnostics should be performed when fitting a linear model
to assess for bias and generalizability of the model.

## Open data file

The rio package works for importing several different types of data
files. We’re going to use it in this class. There are other packages
which can be used to open datasets in R. You can see several options by
clicking on the Import Dataset menu under the Environment tab in
RStudio. (For a csv file like we have this week we’d use either From
Text(base) or From Text (readr). Try it out to see the menu dialog.)

``` r
# Using the file.choose() command allows you to select a file to import from another folder.
# dataset <- rio::import(file.choose())
dataset <- rio::import("Album Sales.sav")
```

## lm() function in R

Many linear models are calculated in R using the lm() function. We’ll
look at how to perform a multiple regression using the lm() function
since it’s so common.

#### Visualization

``` r
# This code creates a scatter matrix
library(GGally)
GGally::ggpairs(dataset, columns=c('Sales','Adverts','Airplay','Image'), lower = list(continuous = "smooth"))
```

``` r
# This code creates a scatterplot between a single pair of variables
ggplot(dataset, aes(x = Adverts, y = Sales)) +
  geom_point() +
  stat_smooth(method = lm)
```

#### Computation

Multiple models adding variables in steps can be created in R using the
lm() and update() functions.

``` r
model.1 <- lm(formula = Sales ~ Adverts, data = dataset)
model.2 <- update(model.1, .~. + Airplay)
model.3 <- update(model.2, .~. + Image)
```

#### Model assessment

``` r
summary(model.1)
summary(model.2)
summary(model.3)
```

#### Standardized residuals from lm()

You might notice lm() does not provide the standardized residuals. Those
must me calculated separately.

``` r
standardized = lm(scale(Sales) ~ scale(Adverts) + scale(Airplay) + scale(Image), data=dataset)
summary(standardized)
```

Another option for getting standardized residuals is in the QuantPsyc
package.

``` r
QuantPsyc::lm.beta(model.3)
```

#### Confidence intervals for the parameters

The lm() function didn’t provide confidence intervals for the model
parameters. This can be done using the confint() function.

``` r
confint(model.3)
```

#### AIC & BIC

From <https://ssc.wisc.edu/sscc/pubs/RFR/RFR_Regression.html>

``` r
# Smaller values indicate a better model
AIC(model.1)
AIC(model.2)
AIC(model.3)
BIC(model.1)
BIC(model.2)
BIC(model.3)
```

#### Compare models

Hierarchal models can be compared using ANOVA.

``` r
anova(model.1, model.2, model.3)
```

## Evaluating for bias in the model

#### Add observation statistics to data (missing in Jamovi)

``` r
# add ID column
dataset$id = 1:nrow(dataset)

# Outliers
dataset$m3resid = round(resid(model.3), digits = 3)
dataset$m3stdres = round(rstandard(model.3), digits = 3)
dataset$m3studres = round(rstudent(model.3), digits = 3)

# Influential cases
dataset$m3cooks = round(cooks.distance(model.3), digits = 3)
dataset$m3dfbetas = round(dfbetas(model.3), digits = 3)
dataset$m3dffits = round(dffits(model.3), digits = 3)
dataset$m3lev = round(hatvalues(model.3), digits = 3)
dataset$m3covrat = round(covratio(model.3), digits = 3)
dataset$m3mahal = mahalanobis(dataset[,c("Adverts","Airplay","Image")],colMeans(dataset[,c("Adverts","Airplay","Image")]),cov(dataset[,c("Adverts","Airplay","Image")]))

# fitted values
dataset$fitted <- round(model.3$fitted.values, digits = 3)
```

After I created a lot of this code, I ran across a function to get a lot
of regression diagnostics quickly. The site seems to give some examples
using more tidyverse notation. From
<http://www.sthda.com/english/articles/39-regression-model-diagnostics/161-linear-regression-assumptions-and-diagnostics-in-r-essentials/>

``` r
model.diag.metrics <- broom::augment(model.3)
head(model.diag.metrics)
```

#### Flag cases for investigation

``` r
# studentized residuals greater than 2 (or 3)
# No more than 5% of cases above 2. No more than 1% above 2.5.
print("Studentized residuals greater than 2 or 3")
dataset$largeRes <- dataset$m3studres > 3 | dataset$m3studres < -3
dataset[dataset$largeRes, c("id")]

# Cook's greater than 1
# Any case
print("Cook's value greater than 1")
dataset$largeCook <- dataset$m3cooks > 1 | dataset$m3cooks < -1
dataset[dataset$largeCook, c("id")]

# Mahalanobis distance greater than chi-square critical value df=number predictors
# related to leverage
print("Mahalanobis distance greater than chi-square cutoff")
dataset$largeMahal <- dataset$m3mahal > qchisq(.99, df=3)
dataset[dataset$largeMahal, c("id")]

# leverage greater than 2 (or 3) times average 
print("Leverage greater than 2 or 3 times the average")
averageLeverage = (3 + 1)/200
dataset$largeLev <- dataset$m3lev > 3*averageLeverage
dataset[dataset$largeLev, c("id")]

# covariance ratio out of bounds (1 +/- (k*(K+1)/n))
print("Covariance ratio out of bounds")
dataset$COVcheck <- dataset$m3covrat > (1 + (3*(3+1)/200)) | dataset$m3covrat < (1 - (3*(3+1)/200))
dataset[dataset$COVcheck, c("id")]

# standardized dfbeta greater than 1 - b change when exclude cases
print("Standardized dfbeta greater than 1")
dataset$largedfbeta <- dataset$m3dfbetas > 1 | dataset$m3dfbetas < -1
dataset[dataset$largedfbeta, c("id")]

# standardized dffit greater than 1 - predicted values change when exclude cases
print("Standardized dffit greater than 1")
dataset$largedffit <- dataset$m3dffits > 1 | dataset$m3dffits < -1
dataset[dataset$largedffit, c("id")]
```

There is another function in R to help flag potentially influential
cases

``` r
# https://web.stanford.edu/class/stats191/notebooks/Diagnostics_for_multiple_regression.html
influence.measures(model.3)
```

Some more plots looking for outliers.

``` r
car::outlierTest(model.3)
qqPlot(model.3, main="QQ Plot")
leveragePlots(model.3)
```

#### Cook’s distance plot

This site is the worst to navigate, but they have some plots and tests
the others don’t.
<https://stats.idre.ucla.edu/wp-content/uploads/2019/02/R_reg_part2.html#(4)>

``` r
# https://stats.idre.ucla.edu/wp-content/uploads/2019/02/R_reg_part2.html#(4)
plot(model.3, which = 4, cook.levels = cutoff)
influencePlot(model.3, main="Influence Plot", sub="Circle size proportional to Cook's Distance")
infIndexPlot(model.3)

# standardized residuals with lines at cutoff points
res.std <- rstandard(model.3)
plot(res.std, ylab = "Standardized Residual", ylim=c(-3.5, 3.5))
abline(h = c(-3, 0, 3), lty=2)
index <- which(res.std > 3 | res.std < -3)
text(index-20, res.std[index], labels = dataset$id[index])
print(index)
```

Nice plot to identify high leverage values.

``` r
#a vector containing the diagonal of the 'hat' matrix
h <- influence(model.3)$hat
#half normal plot of leverage from package faraway
halfnorm(influence(model.3)$hat, ylab = "leverage")
```

#### Added variable plots

May be helpful in identifying influential cases.

``` r
# https://web.stanford.edu/class/stats191/notebooks/Diagnostics_for_multiple_regression.html
# If the partial regression relationship is linear, this plot should look linear.
avPlots(model.3, 'Adverts')
avPlots(model.3, 'Airplay')
avPlots(model.3, 'Image')
```

#### Component + residual plots

Similar to the Added Variable plot, but may be more helpful in
identifying non-linear relationships.

``` r
# https://web.stanford.edu/class/stats191/notebooks/Diagnostics_for_multiple_regression.html
# The green line is a non-parametric smooth of the scatter plot that may suggest
# relationship other than linear.
crPlots(model.3, 'Adverts')
crPlots(model.3, 'Airplay')
crPlots(model.3, "Image")
```

## Evaluating generalizability of the model

This is based on how well the model meets statistical assumptions.

#### Additivity and linearity

Check scatterplots for linear relationships of predictors with outcome.

``` r
# no curve in the graph
# first plot - plot fitted values against residuals
plot(model.3)
```

Residual plots can help check for linearity. From
<https://stats.idre.ucla.edu/wp-content/uploads/2019/02/R_reg_part2.html#(6)>

``` r
car::residualPlots(model.3)
```

#### Independent errors

``` r
# value less than 1 or greater than 3 problematic
# close to 2 is best
durbinWatsonTest(model.3)
dwt(model.3)
```

You can also plot residuals with ID From
<https://stats.idre.ucla.edu/wp-content/uploads/2019/02/R_reg_part2.html#(7)>

``` r
plot(model.3$resid ~ dataset$id)
```

#### Homoscedasticity

``` r
# no funneling
# first plot - plot fitted values against residuals
plot(model.3)

# Breusch-Pagan test https://daviddalpiaz.github.io/appliedstats/model-diagnostics.html
bptest(model.3)
```

#### Normally distributed errors

``` r
# Q-Q plot of residuals for normality
# second plot - points follow the line
plot(model.3)

# can also plot a histogram of the standardized residuals
hist(dataset$m3studres)

# Shapiro-Wilk test on the residuals 
# https://daviddalpiaz.github.io/appliedstats/model-diagnostics.html
shapiro.test(resid(model.3))
```

#### Predictors uncorrelated with external variables (that should have been included)

Variables which correlate with both the predictor and outcome that are
left out of the model will cause conclusions from the model to be
potentially less valid.

#### Correct variable types

All variables should be quantitative. Categorical variables should only
have 2 categories.The outcome should be continuous with no ceiling or
floor effects.

#### No perfect multicollinearity

You can check pair-wise correlations between predictors before fitting
the model. Correlations above .8 could be problematic.

``` r
# largest VIF greater than 10 problematic
# Average VIF greater than 1 regression may be biased
# tolerance less than .1 a serious problem
# tolerance less than .2 potential problems
vif(model.3)
1/vif(model.3)
mean(vif(model.3))
```

#### Non-zero variance

That can be checked with descriptive statistics before the model is fit.

#### Cross-validation

How well does the model predict outcome from a different sample?

## Save new data

Save the dataset with your new regression diagnostics variables.

``` r
# can use with .sav, .rds. .csv files and more
# see documentation for options https://www.rdocumentation.org/packages/rio/versions/0.5.16
rio::export(dataset, "Album Sales Diagnostics.sav")
```

Turn in your dataset with your new influence diagnostics variables with
your other assignment files.
