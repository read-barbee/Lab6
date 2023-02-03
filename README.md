WILD 562 Lab 6: Evaluating RSF Models
================
Mark Hebblewhite
February 03, 2023

# Lab 6: Evaluating RSF Models

Today we will cover the statistical evaluation of model goodness of fit
from a predictive perspective. First we will cover the statistical
approaches for evaluating used-unused designs. Second we will build on
the statistical model evaluation to the most important aspect of
evaluating RSF models – how well they actually predict observed animal
locations. We will build on previous labs statistical development to
create spatial layers of spatial variables that were in the top models
you selected last week (either prey- or habitat-based) top RSF models.
We will learn about the special challenge of evaluating logistic
regression models of binary response data. We will learn about
classification tables, sensitivity, specificity, and ROC curves to
evaluate the ‘optimal’ cutpoint of prediction in logistic models. Next,
we will use k-folds cross validation to conduct in-sample model
validation for the RSF. Finally, we will also explore how different
graphical display options for spatial predictions influence the
interpretation of the ‘habitat map’.

## 0.1 Preliminaries: setting packages

Today we are going to use many of the previous packages we have come to
rely on for spatial mapping, managing data, model selection and
graphing. As well as the new pacakges DescTools and a few other new
ones.
`packages <- c("ROCR", "tidyverse", "ks", "mapview", "colorRamps","rgeos", "VGAM", "AICcmodavg", "MuMIn", "corrgram", "GGally","caret", "DescTools", "car", "sf", "terra", "stars", "tmap")`

## 0.2 Saving and loading data and shapefile data of Kernel home range from Lab

Note, that we CANNOT just use an .RData file because of the Raster files
in this lab. See below where we export our raster stack and then
re-import it.

``` r
wolfkde <- read.csv(here::here("Data","wolfkde5.csv"), header=TRUE, sep = ",", na.strings="NA", dec=".")
wolfkde3 <-na.omit(wolfkde)
wolfkde3$usedFactor <-as.factor(wolfkde3$usedFactor) ## make sure usedFactor is a factor

kernelHR <- st_read(here::here("Data","homerangeALL.shp"))
```

    ## Reading layer `homerangeALL' from data source 
    ##   `C:\Users\Administrator.KJLWS11\Documents\Lab6\Data\homerangeALL.shp' 
    ##   using driver `ESRI Shapefile'
    ## Simple feature collection with 2 features and 2 fields
    ## Geometry type: POLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: 546836.6 ymin: 5662036 xmax: 612093.9 ymax: 5748911
    ## Projected CRS: NAD83 / UTM zone 11N

``` r
tmap_mode("plot")
```

    ## tmap mode set to plotting

``` r
tm_shape(kernelHR) + tm_polygons()
```

![](README_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

``` r
plot(kernelHR)
```

![](README_files/figure-gfm/unnamed-chunk-1-2.png)<!-- -->

``` r
ext(kernelHR)
```

    ## SpatExtent : 546836.576513259, 612093.865451867, 5662035.82104512, 5748911.12903735 (xmin, xmax, ymin, ymax)

``` r
kernels <- rast()
ext(kernels) <- c(xmin=546836, xmax=612093, ymin=5662036, ymax=5748911) 
```

## 0.4 Loading raster’s needed for mapping RSF models later

To map the predictions of an RSF model in space, we need a consistent
raster stack of the input covariates that the RSF models are based on.
For continuous covariates, this is simply the elevation raster. For
categorical covariates, though, such as landcover, we need to create a
‘dummy’ raster where 1 = burn, say, and 0 = everything else, for all
landcover categories. We will process these raster operations first this
week.

``` r
deer_w<-rast("Data/deer_w2.tiff")
moose_w<-rast("Data/moose_w2.tif")
elk_w<-rast("Data/elk_w2.tif") # already brought in above
sheep_w<-rast("Data/sheep_w2.tif")
goat_w<-rast("Data/goat_w2.tif")
wolf_w<-rast("Data/wolf_w2.tif")
elevation2<-rast("Data/Elevation2.tif") #resampled
disthumanaccess2<-rast("Data/DistFromHumanAccess2.tif") #resampled in lab 4
disthhu2<-rast("Data/DistFromHighHumanAccess2.tif") #resampled in lab 4
landcover2 <- rast("Data/landcover16.tif") ## resampled to same extent as lab 4
ext(landcover2)
```

    ## SpatExtent : 540827.834625105, 620387.834625105, 5642002.4200005, 5756332.4200005 (xmin, xmax, ymin, ymax)

``` r
plot(landcover2)
```

    ## Warning: [plot] unknown categories in raster values

``` r
plot(kernelHR, add=TRUE, col = NA) #note - col = NA maps just the polygon border
```

    ## Warning in plot.sf(kernelHR, add = TRUE, col = NA): ignoring all but the first
    ## attribute

![](README_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

``` r
cats(landcover2)
```

    ## [[1]]
    ##    value      HABITATTYPE
    ## 1      1     Open Conifer
    ## 2      2 Moderate Conifer
    ## 3      3   Closed Conifer
    ## 4      4        Deciduous
    ## 5      5     Mixed Forest
    ## 6      6     Regeneration
    ## 7      7       Herbaceous
    ## 8      8            Shrub
    ## 9      9            Water
    ## 10    10         Rock-Ice
    ## 11    11            Cloud
    ## 12    12      Burn-Forest
    ## 13    13   Burn-Grassland
    ## 14    14       Burn-Shrub
    ## 15    15      Alpine Herb
    ## 16    16     Alpine Shrub

Note these values correspond to the same numbers are lab 4, i.e., 1 =
open conifer, 2 = moderate conifer, etc. We will be creating separate
‘dummy’ variable raster layers below using these habitattypes values
later.

Next, note that the extents are all different for human access and
elc_habitat-derived layers, so we need to resample landcover and human
access layers to match extents.

``` r
elevation2 <- resample(elevation2, elk_w)
disthhu2 <- resample(disthhu2, elk_w)
disthumanaccess2 <- resample(disthumanaccess2, elk_w)
landcover2 <- resample(landcover2, elk_w)
```

Then we create and set values for landcover rasters based on landcover2
values of Habitat classification variables. Note below that we are using
the command `ifel()` which is the `terra` equivalent of `ifelse()`.

``` r
alpine <- ifel(landcover2 == 15 | landcover2 == 16, 1,ifel(is.na(landcover2),NA,0))
burn <- ifel(landcover2 == 12 | landcover2 == 13 |landcover2 == 14, 1, ifel(is.na(landcover2),NA,0))
closedConif <- ifel(landcover2 == 3, 1, ifel(is.na(landcover2),NA,0))
herb <- ifel(landcover2 == 7, 1, ifel(is.na(landcover2),NA,0))
mixed <- ifel(landcover2 == 5, 1, ifel(is.na(landcover2),NA,0))
rockIce <- ifel(landcover2 == 10, 1, ifel(is.na(landcover2),NA,0))
water <- ifel(landcover2 == 9, 1, ifel(is.na(landcover2),NA,0))
modConif <- ifel(landcover2 == 2, 1, ifel(is.na(landcover2),NA,0))
decid <- ifel(landcover2 == 10, 1, ifel(is.na(landcover2),NA,0))
plot(rockIce)
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

``` r
plot(closedConif)
```

![](README_files/figure-gfm/unnamed-chunk-5-2.png)<!-- -->

``` r
# note that open conifer is implicitly being defined as the intercept
```

## 0.6 Creating a Raster Stack

A Raster stack could help with mapping later, but, its not really needed
this lab. Recall all rasters must have same extent and resolution, which
will help.

``` r
all_rasters<-c(deer_w, moose_w, elk_w, sheep_w, goat_w, wolf_w,elevation2, disthumanaccess2, disthhu2, landcover2, alpine, burn, closedConif, modConif, herb, mixed, rockIce, water, decid)

plot(all_rasters) ## note limit of plotting 9 layers
```

    ## Warning: [plot] unknown categories in raster values

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

Note, the next two sets of commands give a way to export the processed
rasters, all of them, to a new folder in similar format.

``` r
names = c("deer_w", "moose_w", "elk_w", "sheep_w", "goat_w", "wolf_w","elevation2", "disthumanaccess2", "disthhu2", "landcover2", "alpine", "burn", "closedConif", "modConif", "herb", "mixed", "rockIce", "water", "decid")

writeRaster(all_rasters,"Output/lab6Stack.tif",names = 'names', overwrite = TRUE)
```

# Multiple Logistic Regression & Collinearity

We will first evaluate collinearity between Distance to High Human Use
and Elevation by re-running the ‘top’ models from Lab 5 for the ‘biotic’
and ‘environmental’ models.

``` r
## top Biotic model was model 41
top.biotic <- glm(used ~ DistFromHumanAccess2+deer_w2 + goat_w2, family=binomial(logit), data=wolfkde3)
summary(top.biotic)
```

    ## 
    ## Call:
    ## glm(formula = used ~ DistFromHumanAccess2 + deer_w2 + goat_w2, 
    ##     family = binomial(logit), data = wolfkde3)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -1.8345  -0.6246  -0.1888  -0.0306   3.1247  
    ## 
    ## Coefficients:
    ##                       Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept)          -3.553037   0.366553  -9.693  < 2e-16 ***
    ## DistFromHumanAccess2 -0.001422   0.000183  -7.770 7.86e-15 ***
    ## deer_w2               0.898069   0.077941  11.522  < 2e-16 ***
    ## goat_w2              -0.333540   0.048524  -6.874 6.26e-12 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 2040.9  on 2117  degrees of freedom
    ## Residual deviance: 1398.2  on 2114  degrees of freedom
    ## AIC: 1406.2
    ## 
    ## Number of Fisher Scoring iterations: 7

``` r
#double check the VIF for each final model, just to be sure
vif(top.biotic)
```

    ## DistFromHumanAccess2              deer_w2              goat_w2 
    ##             1.039378             1.020137             1.022622

Environmental Model - top model is model 11

``` r
top.env <- glm(used ~ Elevation2 + DistFromHighHumanAccess2 + openConif+modConif+closedConif+mixed+herb+shrub+water+burn, family=binomial(logit), data=wolfkde3)
summary(top.env)
```

    ## 
    ## Call:
    ## glm(formula = used ~ Elevation2 + DistFromHighHumanAccess2 + 
    ##     openConif + modConif + closedConif + mixed + herb + shrub + 
    ##     water + burn, family = binomial(logit), data = wolfkde3)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.0290  -0.5020  -0.1576  -0.0366   3.2732  
    ## 
    ## Coefficients:
    ##                            Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept)               9.570e+00  8.805e-01  10.869  < 2e-16 ***
    ## Elevation2               -6.782e-03  4.883e-04 -13.888  < 2e-16 ***
    ## DistFromHighHumanAccess2  1.867e-04  3.511e-05   5.317 1.05e-07 ***
    ## openConif                 8.457e-01  4.404e-01   1.920   0.0548 .  
    ## modConif                 -1.716e-02  3.836e-01  -0.045   0.9643    
    ## closedConif              -1.126e-01  3.944e-01  -0.286   0.7752    
    ## mixed                     1.325e+00  5.435e-01   2.438   0.0148 *  
    ## herb                      8.564e-01  5.525e-01   1.550   0.1212    
    ## shrub                     5.781e-01  4.486e-01   1.289   0.1974    
    ## water                     8.559e-01  6.389e-01   1.340   0.1804    
    ## burn                      1.861e+00  4.629e-01   4.021 5.80e-05 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 2040.9  on 2117  degrees of freedom
    ## Residual deviance: 1298.3  on 2107  degrees of freedom
    ## AIC: 1320.3
    ## 
    ## Number of Fisher Scoring iterations: 7

``` r
vif(top.env)
```

    ##               Elevation2 DistFromHighHumanAccess2                openConif 
    ##                 2.186429                 2.026714                 2.892384 
    ##                 modConif              closedConif                    mixed 
    ##                 7.720150                 5.317916                 1.857145 
    ##                     herb                    shrub                    water 
    ##                 1.778124                 2.733747                 1.515043 
    ##                     burn 
    ##                 2.322895

Note I acknowledge some of the problems of inclduing elevation and
distance from high human access in the model both because of the high
correlation between the two, r = 0.52, but also the potentially
important confounding. I also note the high VIF for modConif. However,
one could ‘use’ the upper limits of guidelines of including correlated
variables, r = 0.6 - 0.7, and, the VIF \< 10 as an argument to include
these 2 variables. I personally wouldnt find it that convincing, but
these are published guidelines, so we will go with it for now.
Personally, I would want to compare the fit (using AIC), coefficients,
and VIF’s of a top environmental with and without disthha in it . OR
test it in cross-validation. But, lets proceed. This is a nice, complex
enough environment model.

# Calculate model selection AIC table

``` r
models = list(top.biotic, top.env)
modnames = c("top biotic", "top env")
aictab(cand.set = models, modnames = modnames)
```

    ## 
    ## Model selection based on AICc:
    ## 
    ##             K    AICc Delta_AICc AICcWt Cum.Wt      LL
    ## top env    11 1320.41       0.00      1      1 -649.14
    ## top biotic  4 1406.25      85.85      0      1 -699.12

``` r
# aictab(models, modnames) ## short form. 
```

Ok, so the ‘top’ models from Lab 5 were the biotic and environmental
models but the environmental model is FAR better. We will see how it
fares below.

## Evaluating Predictions from Residual Diagnostic Plots

First, we will proceed using ‘normal’ regression diagostics of residual
plots proceeding as if this model was a linear model. We will start
first with the top.env model Review here in Lab
<http://www.statmethods.net/stats/rdiagnostics.html> for Linear
Regression (Normal regression).

``` r
par(mfrow = c(2,2))
plot(top.env)
```

![](README_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

This cylces through residual plots which look decidedly different from
‘normal’ residual plots. Why do they look so weird??? The response is
binomial, not gaussian. Thus, honeslty, residual plots are really
useless at evaluating model fit. We will return to evaluating goodness
of fit in lab 6.

Next, we will save the predictions from the top model using the fitted()
function and examine the predictions ourselves directly. We will then go
on to calculate the following 1) residuals 2) studentized residuals 3)
leverage statistics include the hat-value and Cooks distance .

``` r
##### Saving predictions manually, an example with the environment model
wolfkde3$fitted.top.env <- fitted(top.env)
#### this is the predicted probability from the model

wolfkde3$residuals.top.env <- residuals(top.env)
## these are the deviations from the predictions for each row (data point)

wolfkde3$rstudent.top.env <- rstudent(top.env)
## This is a standardized residual - the studentized residual

wolfkde3$hatvalues.top.env <- hatvalues(top.env)
#### this is the first of the leverage statistics, the larger hat value is, the bigger the influence on the fitted value

wolfkde3$cooks.distance.top.env <- cooks.distance(top.env)
#### this is the Cooks leverage statistic, the larger hat value is, the bigger the influence on the fitted value

wolfkde3$obsNumber <- 1:nrow(wolfkde3) ## just added a row number for plotting
```

Making manual residual verus predicted plots

``` r
ggplot(wolfkde3, aes(fitted.top.env, residuals.top.env)) + geom_point() + geom_text(aes(label = obsNumber, colour = used))
```

![](README_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

``` r
## Manual residual plot

ggplot(wolfkde3, aes(wolfkde3$residuals.top.env, wolfkde3$cooks.distance.top.env)) + geom_point() + geom_text(aes(label = obsNumber, colour = used))
```

![](README_files/figure-gfm/unnamed-chunk-13-2.png)<!-- -->

``` r
## shows us some points at high cooks values that might be having a big influence

ggplot(wolfkde3, aes(wolfkde3$cooks.distance.top.env, wolfkde3$hatvalues.top.env)) + geom_point() + geom_text(aes(label = obsNumber, colour = used))
```

![](README_files/figure-gfm/unnamed-chunk-13-3.png)<!-- --> This shows
us some points at high cooks values that might be having a big
influence. This helps identify some locations that have high leverage
that are used (1 - blue) and available (0=black) points. For example,
datapoint 16

``` r
## e.g., datapoint 16
wolfkde3[16,]
```

    ##    deer_w2 moose_w2 elk_w2 sheep_w2 goat_w2 wolf_w2 Elevation2
    ## 19       4        4      4        4       3       4   2086.334
    ##    DistFromHumanAccess2 DistFromHighHumanAccess2 landcover16 EASTING NORTHING
    ## 19             1289.253                 1595.905           1  570610  5710430
    ##        pack used usedFactor  habitatType    landcov.f closedConif modConif
    ## 19 Red Deer    1          1 Open Conifer Open Conifer           0        0
    ##    openConif decid regen mixed herb shrub water rockIce burn alpineHerb
    ## 19         1     0     0     0    0     0     0       0    0          0
    ##    alpineShrub alpine fitted.top.env residuals.top.env rstudent.top.env
    ## 19           0      0     0.03119327          2.633459         2.654942
    ##    hatvalues.top.env cooks.distance.top.env obsNumber
    ## 19       0.003644757              0.0103663        16

This is a red deer wolf used point at high elevtion in open conifer far
from human access. This does not seem to be a data entry error - a REAL
data point. But - look at the deviation between the predicted
probability of use, and the observed - given that it was a USED (1)
point. The predicted probability is 0.03, yet it was used = 1. We will
return to this point below.

Next we will explore evaluating model fit graphically. First lets make a
plot of Y against X predictions…

``` r
scatterplot(fitted.top.env~Elevation2, reg.line=lm, smooth=TRUE, spread=TRUE, boxplots='xy', span=0.5, xlab="elevation", ylab="residual", cex=1.5, cex.axis=1.4, cex.lab=1.4, data=wolfkde3)
```

![](README_files/figure-gfm/unnamed-chunk-15-1.png)<!-- -->

``` r
hist(wolfkde3$fitted.top.env, scale="frequency", breaks="Sturges", col="darkgray")
```

![](README_files/figure-gfm/unnamed-chunk-15-2.png)<!-- --> This last
histogram plot is VERY important - it is the predicted probability of a
location being a wolf used location given your top model. But how would
we divide this into wolf habitat and wolf available?

``` r
ggplot(wolfkde3, aes(x=wolfkde3$fitted.top.env, fill=usedFactor)) + geom_histogram(binwidth=0.05, position="identity", alpha=0.7) + xlab("Predicted Probability of Wolf Use") + theme(axis.title.x=element_text(size=16)) #+ facet_grid(pack ~ ., scales="free")
```

![](README_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

This plot shows that somewhere around 0.25 - 0.40 eyeballing it it looks
like we could ‘cut’ used and available points?

``` r
ggplot(wolfkde3, aes(x=fitted.top.env, y=..density.., fill=usedFactor)) + geom_histogram(binwidth=0.05, position="identity", alpha=0.7) + xlab("Predicted Probability of Wolf Use") + theme(axis.title.x=element_text(size=16)) + facet_grid(pack ~ ., scales="free")
```

    ## Warning: The dot-dot notation (`..density..`) was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `after_stat(density)` instead.

![](README_files/figure-gfm/unnamed-chunk-17-1.png)<!-- --> But note
this ‘cut’ point looks different for both wolf packs?

Regardless of between pack differences, this introduces the basic
problem of dividing continuous predictions from logistic regression into
categories of habitat (1) and available (0). There are a few things to
note here. First, the range of predicted values differs between the two
wolf packs. Second, the amount of ‘overlap’ between TRUE 1’s and 0’s
differ between the two different wolf packs. And third, the ‘asymmetry’
of predicted probabilities between the 0’s and 1’s are obviously
different. We will revisit all three of these observations during the
rest of the lab.

## Evaluating predictions from residual plots for top.biotic model

Next, we will do the same set of steps for the top Biotic model.

``` r
par(mfrow = c(2,2))
plot(top.biotic)
```

![](README_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

``` r
##### Saving predictions manually, an example with the environment model
wolfkde3$fitted.top.biotic <- fitted(top.biotic)
#### this is the predicted probability from the model

wolfkde3$residuals.top.biotic <- residuals(top.biotic)
## these are the deviations from the predictions for each row (data point)

wolfkde3$rstudent.top.biotic <- rstudent(top.biotic)
## This is a standardized residual - the studentized residual

wolfkde3$hatvalues.top.biotic <- hatvalues(top.biotic)
#### this is the first of the leverage statistics, the larger hat value is, the bigger the influence on the fitted value

wolfkde3$cooks.distance.top.biotic <- cooks.distance(top.biotic)
#### This isthe Cooks leverage statistic


## Making manual residual verus predicted plots
ggplot(wolfkde3, aes(wolfkde3$cooks.distance.top.biotic, wolfkde3$hatvalues.top.biotic)) + geom_point() + geom_text(aes(label = obsNumber, colour = used))
```

![](README_files/figure-gfm/unnamed-chunk-18-2.png)<!-- -->

Similarly to the Environment Model, this helps identify some locations
that have high leverage that are used (1 - blue) and available (0=black)
points. Similar to last time, datapoint 16. But also datapoint 30.

``` r
wolfkde3[30,]
```

    ##    deer_w2 moose_w2 elk_w2 sheep_w2 goat_w2 wolf_w2 Elevation2
    ## 35       3        4      4        4       4       4   2169.713
    ##    DistFromHumanAccess2 DistFromHighHumanAccess2 landcover16 EASTING NORTHING
    ## 35             1704.151                 9264.565           2  580310  5708450
    ##        pack used usedFactor      habitatType        landcov.f closedConif
    ## 35 Red Deer    1          1 Moderate Conifer Moderate Conifer           0
    ##    modConif openConif decid regen mixed herb shrub water rockIce burn
    ## 35        1         0     0     0     0    0     0     0       0    0
    ##    alpineHerb alpineShrub alpine fitted.top.env residuals.top.env
    ## 35          0           0      0     0.03129583          2.632212
    ##    rstudent.top.env hatvalues.top.env cooks.distance.top.env obsNumber
    ## 35         2.644415       0.002075902            0.005865753        30
    ##    fitted.top.biotic residuals.top.biotic rstudent.top.biotic
    ## 35       0.009800049             3.041502            3.053085
    ##    hatvalues.top.biotic cooks.distance.top.biotic
    ## 35         0.0006981523                0.01766003

This is another high wolf used point that is being classified as an
AVAILABLE location. **Misclassification !!! **

Next we will evaluate model fit graphically, first making a plot of Y
against X predictions… for Elevation.

``` r
scatterplot(fitted.top.biotic~Elevation2, reg.line=lm, smooth=TRUE, spread=TRUE, boxplots='xy', span=0.5, xlab="elevation", ylab="residual", cex=1.5, cex.axis=1.4, cex.lab=1.4, data=wolfkde3)
```

![](README_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

``` r
## next, the histogram of predicted probaiblities
hist(wolfkde3$fitted.top.biotic, scale="frequency", breaks="Sturges", col="darkgray")
```

![](README_files/figure-gfm/unnamed-chunk-20-2.png)<!-- -->

This plot is VERY important - it is the predicted probability of a
location being a wolf used location given your top model. But how would
we divide this into wolf habitat and wolf available?

``` r
ggplot(wolfkde3, aes(x=wolfkde3$fitted.top.biotic, fill=usedFactor)) + geom_histogram(binwidth=0.05, position="identity", alpha=0.7) + xlab("Predicted Probability of Wolf Use") + theme(axis.title.x=element_text(size=16)) #+ facet_grid(pack ~ ., scales="free")
```

![](README_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

``` r
#### This plot shows that somewhere around 0.25 - 0.40 eyeballing it it looks like we could 'cut' used and available points? 

ggplot(wolfkde3, aes(x=fitted.top.biotic, y=..density.., fill=usedFactor)) + geom_histogram(binwidth=0.05, position="identity", alpha=0.7) + xlab("Predicted Probability of Wolf Use") + theme(axis.title.x=element_text(size=16)) + facet_grid(pack ~ ., scales="free")
```

![](README_files/figure-gfm/unnamed-chunk-21-2.png)<!-- -->

``` r
#### But note this 'cut' point looks different for both wolf packs?
```

So, similar but slightly starker differences between the predictions
evaluated at the wolf pack level for the Biotic model, but the same 3
observations as for the top Environmental Models.

## Comparing the ‘Fit’ of Predictions From The Biotic and Environmental Models

``` r
ggplot(wolfkde3, aes(x=fitted.top.biotic, y=fitted.top.env)) + geom_point() + stat_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

This shows a strong positive correlation between predicted probabilities
of wolf use between the two models. Naturally this leads us to think of
things like averaging predictions across models, a point we will return
to much later in ths semester when we consider ensemble modeling.

What about the comparison of predictions from these two models when
split by wolf packs?

``` r
ggplot(wolfkde3, aes(x=fitted.top.biotic, y=fitted.top.env, fill = pack)) + geom_point() + stat_smooth(method="lm") 
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

There is quite a bit of scatter here in the predictions between the two
models, but in general, they are highly correlated. We also note the
truncated predictions for the Red deer pack, which are being driven by
the higher elevaitons in that pack’s home range, and thus, lower
predicted probabilities in our ‘average’ one size fits all model.

But what if we do not force a linear model through the predictions?

``` r
ggplot(wolfkde3, aes(x=fitted.top.biotic, y=fitted.top.env, fill = pack)) + geom_point() + stat_smooth()
```

    ## `geom_smooth()` using method = 'gam' and formula = 'y ~ s(x, bs = "cs")'

![](README_files/figure-gfm/unnamed-chunk-24-1.png)<!-- --> This shows
there is some evidence that the top environmental model (Y axis) is not
successfully predicting the ‘best’ wolf habitat especially in the bow
valley wolf pack compared to the top biotic model (X axis).

# Classification Tables

The excercises above where we plotted the predicted probabilities of use
as a function of whether the data were an actual true wolf location, or,
a random available location, illustrate the problem of classification.
And, Misclassification in the cross classication of a continuous
prediction against a discrete input data. Next we will learn about the
problem of classification for Binary logistic regression, and how to
calculate classification table tests.

[R Bloggers Post on Evaluating Logistic Regression
Models](https://www.r-bloggers.com/evaluating-logistic-regression-models/)
[Useful Vignette on Evaluating Multiple Logistic Regression
Models](https://rpubs.com/ryankelly/ml_logistic)

The basic problem of classification in logistic regression is that we
have a binary response variable, 1 = Used, 0 = Avail (or unused), and
yet a prediction that is a continuous predicted probability between 0
and 1. This is VERY different from a normal linear regression model
where the predicted values are in the same range and units of values as
the observed data. Consider the regression of height and weight against
each other where were are trying to predict weight based just on height.
We can use standard model goodness of fit measures such as the
Coefficient of Determination, R^2, for example, that measure the amount
of variation in Y explained by X. Here, with a binary input variable and
a continuous probability prediction, we MUST first classify our
predictions into whether they were a 0 or 1 first.

## Pseuod R-squared

Cox and Snell’s R^2 is based on the log likelihood for the model
compared to the log likelihood for a baseline model. However, with
categorical outcomes, it has a theoretical maximum value of less than 1,
even for a “perfect” model.

Nagelkerke’s R 2 is an adjusted version of the Cox and Snell’s R 2 that
adjusts the scale of the statistic to cover the full range from 0 to 1
(again except for models with categorical variables in it).

McFadden’s R 2 is another version, based on the log-likelihood kernels
for the intercept-only model and the full estimated model.

``` r
#require(DescTools)
PseudoR2(top.biotic, c("McFadden", "CoxSnell", "Nagel"))
```

    ##   McFadden   CoxSnell Nagelkerke 
    ##  0.3148914  0.2617169  0.4231606

The first thing to note about these PseudoR2 measures are they are not
directly comparable to R-squared from linear models, despite the
dangerous temptation. They do not always go from 0 - 1, theoretically,
and so its tough to gauge their value. None are very satisfying, nor,
tell us much useful about the model. Moreover, as Fielding and Bell
note, they do not necessarily apply to Used-Available designs.

## Classification Table for Top Biotic Model

First we will arbitrarily define the cutpoint between 1 and 0’s using p
= 0.5, which might make sense in the case of a fair coin flip. We will
see that the ‘optimal’ cutpoint varies for each individual dataset. The
baseline optimal cutpoint depends on the baseline probabiltiy of an
outcome, which is simply the ratio of the frequency of 1’s over the
count of 1’s + 0’s (i.e., the proportion of used locations in an RSF).
And second, that what we are ‘optimizing’ may itself be affected by the
research questions.

``` r
# First we will arbitrarily define the cutpoint between 1 and 0's using p = 0.5
ppused = wolfkde3$fitted.top.biotic>0.5
table(ppused,wolfkde3$used)
```

    ##        
    ## ppused     0    1
    ##   FALSE 1655  229
    ##   TRUE    67  167

What this shows is on the top, the data - 0 = available, and 1 = used.
On the ROWS, we see points classified as 0’s (FALSE) versus those
classified as TRUE (1). Correctly classified 0’s are 1655, and correctly
classifed 1’s are 167. But what are the cross diagonals?

Next, we will go through Hosmer and Lemeshow Chapter 5 to calculate our
classification success for 1’s?

``` r
167/(167+229)
```

    ## [1] 0.4217172

So when wolf telemetry locations were known = 1, the model classified
42% as ‘used’. *This is pretty terrible, don’t you think?* This is also
called the **Specificity of a Model, i.e., how well the model specifies
TRUTH when = 1**. Another synonym is the True Positive Rate (TPR)

Now lets do the correct classification rate of the true 0’s (i.e.,
avail)

``` r
1655/(1655+67)
```

    ## [1] 0.9610918

But when the points were really 0’s, available, we classified them 96%
of the time correctly. **This is called the Sensitivity of a model, the
correct classification of 0’s.** Otherwise known as the True Negative
Rate.

Now lets do this manually using cutpoints of 0.25 and 0.1

``` r
ppused = wolfkde3$fitted.top.biotic>0.25
table(ppused,wolfkde3$used)
```

    ##        
    ## ppused     0    1
    ##   FALSE 1376   92
    ##   TRUE   346  304

Now, what is specificity? (i.e., the probability of classifying the 1’s
correctly?)

``` r
304/(304+92)
```

    ## [1] 0.7676768

Specificity is about 76% - Great! We are doing better at classifying the
0’s with a lower ‘cutpoint’ probability.

But - what happened to our sensitivity (i.e., the probability of
classifying the 0’s correctly?)

``` r
1376 / (1376+346)
```

    ## [1] 0.7990708

So our probability of classifying 0’s correctly decreases to \~ 80% with
our sensitivity

Now lets try a p = 0.10

``` r
ppused = wolfkde3$fitted.top.biotic>0.10
table(ppused,wolfkde3$used)
```

    ##        
    ## ppused     0    1
    ##   FALSE 1001   39
    ##   TRUE   721  357

Now, what is specificity? (i.e., the probability of classifying the 1’s
correctly?)

``` r
357/(357+39)
```

    ## [1] 0.9015152

Our Specificity is about 90% - Even better! But, guess what happened to
our sensitivity (i.e., the probability of classifying the 0’s
correctly?)

``` r
1001 / (1001+721)
```

    ## [1] 0.5813008

Thus, our sensitivity, correct classification of 0’s, has decreased to
58%. Hmm. It seems tough to have it all, doesn’t it?

Finally, lets try a p = 0.70

``` r
ppused = wolfkde3$fitted.top.biotic>0.70
table(ppused,wolfkde3$used)
```

    ##        
    ## ppused     0    1
    ##   FALSE 1705  330
    ##   TRUE    17   66

What is specificity? (i.e., the probability of classifying the 1’s
correctly?)

``` r
66/(330+66)
```

    ## [1] 0.1666667

With this higher cutpoint probability, Specificity is about 17% -
Terrible! And you can guess what happened to our ability to classify
0’s.

``` r
17 / (1705+17)
```

    ## [1] 0.009872242

So our probability of classifying 0’s correctly decreases with our
sensitivity, but we observe for the first time that there is a trade off
between Sensitivity (true 1’s) and specificity (true 0’s) as we change
the cutpoint probability. We will return to how to pick the ‘optimal’
cutpoint shortly with ROC Curves - Receiver Operating Characteristic
Curves.

## Confusion Matrices

There is a simpler way, naturally, to calculate so called (well named)
confusion matrices. We will calculate the Confusion Matrix from the
package caret

``` r
#require(caret)
wolfkde3$pr.top.biotic.used <- ifelse(wolfkde3$fitted.top.biotic>0.5, 1, 0)
xtab1<-table(wolfkde3$pr.top.biotic.used, wolfkde3$used)
xtab1
```

    ##    
    ##        0    1
    ##   0 1655  229
    ##   1   67  167

``` r
#?confusionMatrix
confusionMatrix(xtab1) ## Note this is incorrectly specifying what the 1's are. 
```

    ## Confusion Matrix and Statistics
    ## 
    ##    
    ##        0    1
    ##   0 1655  229
    ##   1   67  167
    ##                                           
    ##                Accuracy : 0.8602          
    ##                  95% CI : (0.8447, 0.8747)
    ##     No Information Rate : 0.813           
    ##     P-Value [Acc > NIR] : 4.668e-09       
    ##                                           
    ##                   Kappa : 0.4544          
    ##                                           
    ##  Mcnemar's Test P-Value : < 2.2e-16       
    ##                                           
    ##             Sensitivity : 0.9611          
    ##             Specificity : 0.4217          
    ##          Pos Pred Value : 0.8785          
    ##          Neg Pred Value : 0.7137          
    ##              Prevalence : 0.8130          
    ##          Detection Rate : 0.7814          
    ##    Detection Prevalence : 0.8895          
    ##       Balanced Accuracy : 0.6914          
    ##                                           
    ##        'Positive' Class : 0               
    ## 

``` r
confusionMatrix(xtab1, positive = "1") ## Thanks to spring 2019 student Forest HAyes for finding this mistake :)
```

    ## Confusion Matrix and Statistics
    ## 
    ##    
    ##        0    1
    ##   0 1655  229
    ##   1   67  167
    ##                                           
    ##                Accuracy : 0.8602          
    ##                  95% CI : (0.8447, 0.8747)
    ##     No Information Rate : 0.813           
    ##     P-Value [Acc > NIR] : 4.668e-09       
    ##                                           
    ##                   Kappa : 0.4544          
    ##                                           
    ##  Mcnemar's Test P-Value : < 2.2e-16       
    ##                                           
    ##             Sensitivity : 0.42172         
    ##             Specificity : 0.96109         
    ##          Pos Pred Value : 0.71368         
    ##          Neg Pred Value : 0.87845         
    ##              Prevalence : 0.18697         
    ##          Detection Rate : 0.07885         
    ##    Detection Prevalence : 0.11048         
    ##       Balanced Accuracy : 0.69140         
    ##                                           
    ##        'Positive' Class : 1               
    ## 

This reveals a LOT of information - lets compare these values to the
table in the ? confusionMatrix help file, and here in this table.

A confusion matrix is an N X N matrix, where N is the number of classes
being predicted. For the problem in hand, we have N=2, and hence we get
a 2 X 2 matrix. Here are a few definitions, you need to remember for a
confusion matrix :

*Accuracy* : the proportion of the total number of predictions that were
correct. *Positive Predictive Value or Precision* : the proportion of
positive cases that were correctly identified. *Negative Predictive
Value* : the proportion of negative cases that were correctly
identified. *Sensitivity or Recall* : the proportion of actual positive
cases which are correctly identified. *Specificity* : the proportion of
actual negative cases which are correctly identified.

![Figure 6.1. Calculations of different quantities from a Confusion
Matrix. Note, Target represents the TRUTH, and Model represents the
PREDICTIONS.](Figures/ConfusionMatrix.png)

*Excercise: Redo for the top.env model* on your own.

## Receiver Operating Characterstic ROC curves

The receiver operating characteristic curve, i.e., ROC curve, is a
graphical plot that illustrates the diagnostic ability of a binary
classifier system as its discrimination threshold, the cutoff, is
varied.

The ROC curve is created by plotting the true positive rate (TPR)
against the false positive rate (FPR) at various threshold settings. The
true-positive rate is also known as *Sensitivity*, recall or probability
of detection in machine learning. *Specificity* is therefore the True
Negative Rate (TNR), that is, the classification success of true 0’s.

Conversely, the false-positive rate is also known as the fall-out or
probability of false alarm and can be calculated as (1 − *Specificity*).
It can also be thought of as a plot of the Power as a function of the
Type I Error of the decision rule (when the performance is calculated
from just a sample of the population, it can be thought of as estimators
of these quantities).

The ROC curve is thus the sensitivity as a function of fall-out. In
general, if the probability distributions for both detection and false
alarm are known, the ROC curve can be generated by plotting the
cumulative distribution function (area under the probability
distribution from − ∞ -to the discrimination threshold) of the detection
probability in the y-axis versus the cumulative distribution function of
the false-alarm probability on the x-axis.

ROC analysis provides tools to select possibly optimal models and to
discard suboptimal ones independently from (and prior to specifying) the
cost context or the class distribution. ROC analysis is related in a
direct and natural way to cost/benefit analysis of diagnostic decision
making.

[Wikipedia
Source](https://en.wikipedia.org/wiki/Receiver_operating_characteristic)

*Other References* [An Illustrated Guide to ROC and AUC for Logisitic
Regression
Models](https://www.r-bloggers.com/illustrated-guide-to-roc-and-auc/)

Today, we will use the ROCR package in R [ROCR
package](ROCR%20package%20help%20here:%20https://rocr.bioinf.mpi-sb.mpg.de)

As well as this journal paper here:

Sing, T., O. Sander, N. Beerenwinkel, and T. Lengauer. 2005. ROCR:
visualizing classifier performance in R. Bioinformatics 21:7881.

## Sensitivity and Specificity

Remember that sensitivity is the True Positive Rate, or, the
classification success of 1’s when they are truly 1. And that
Specificity is the Ture Negative rate. 1 - TNR is known as the False
Negative Rate, something we also need to think of.

``` r
#require(ROCR)
pp = predict(top.biotic,type="response")
pred = prediction(pp, wolfkde3$used)

perf3 <- performance(pred, "sens", x.measure = "cutoff")
plot(perf3)
```

![](README_files/figure-gfm/unnamed-chunk-39-1.png)<!-- --> Look at what
happens to our Sensitivity as we change the cutoff value. Remember that
sensitivity is the True Positive Rate, or, the classification success of
1’s when they are truly 1. Looking at the graph, we see that we classify
everything as a wolf used location when the cutoff is really low.

Next, lets examine the relationship between the cutoff value and
Specificity, or, the True Negative Rate - the rate we correctly classify
0’s.

``` r
perf4 <- performance(pred, "spec", x.measure = "cutoff")
plot(perf4)
```

![](README_files/figure-gfm/unnamed-chunk-40-1.png)<!-- --> Similarly,
if we use a really low threshold cutoff value between 0’s and 1’s, we
see that we have the lowest Specificity - because basically we are
calling everything a 1, and misclassifiying the true 0’s. As the cutoff
increases, we see that there is a sharp increase in our Specificity,
approaching 100% of all 0’s correctly classified by a cutoff of about
0.5.

Obviously, in most cases we want to maximize both Sensitivity and
Specificity for a model with what could be called the ‘optimal’
cutpoint.

## Estimating the Optimal Cutpoint

Next, we will calculate the Maximum for the sum of sensitivity and
specificity to calculate the optimal cutpoint probability.

``` r
perfClass <- performance(pred, "tpr","fpr") # change 2nd and/or 3rd arguments for other metrics
fpr <- perfClass@x.values[[1]]
tpr <- perfClass@y.values[[1]]
sum <- tpr + (1-fpr)
index <- which.max(sum)
cutoff <- perfClass@alpha.values[[1]][[index]]
cutoff
```

    ## [1] 0.2363158

Thus the cutpoint that maximizes the overall classification succes is
0.236. Note that this is VERY different from our naive starting value of
0.5.

Where does this value come from?

``` r
table(wolfkde3$used)
```

    ## 
    ##    0    1 
    ## 1722  396

``` r
396/(1722+396)
```

    ## [1] 0.1869688

Note that the baseline prevalence, 18% of used locations in our dataset,
is VERY close (but not precisely the same) as the optimal cutpoint of
the model. This is because of the strong effects of covariates in our
model. But this baseline prevalence will be a CLOSE guide to the Optimal
cutpoint, on average.

Now, lets overlay the sensitivity, specificity, and optimal cutoff
curves together.

``` r
plot(perf3, col="blue") # Sensitivity
plot(perf4, add = TRUE) # Specificity
abline(v=cutoff, col="red") ## optimal cutpoint
```

![](README_files/figure-gfm/unnamed-chunk-43-1.png)<!-- --> Note how
this cutpoint probabilty OPTIMIZES the correct classificaiton of 1’s
(Specificity) and Sensitivity (0’s).

## ROC Plot

Now we will put the TPR and FPR (1 - Specificity) together to estimate
the Receiver Operating Characteristic Curve (ROC plot). Receiver
Operating Characteristic(ROC) summarizes the model’s performance by
evaluating the trade offs between true positive rate (sensitivity) and
false positive rate (1- specificity). For plotting ROC, it is advisable
to assume p \> 0.5 since we are more concerned about success rate. ROC
summarizes the predictive power for all possible values of p \> 0.5. The
area under curve (AUC), referred to as index of accuracy(A) or
concordance index, is a perfect performance metric for ROC curve. Higher
the area under curve, better the prediction power of the model. Below is
a sample ROC curve. The ROC of a perfect predictive model has TP equals
1 and FP equals 0. This curve will touch the top left corner of the
graph.

``` r
plot(perfClass)
abline(a=0, b= 1)
```

![](README_files/figure-gfm/unnamed-chunk-44-1.png)<!-- -->

This plot shows the trade off between the True Positive Rate versus the
False Positive Rate for our top model. At every cutoff, the TPR and FPR
are calculated and plotted. The smoother the graph, the more data the
predictions have. We also plotted a 45-degree line, which represents, on
average, the performance of a Uniform(0, 1) random variable. Think of a
random coin flip, heads or tails, as the ‘null’ model against which we
have to evaluate whether a logistic model does better in predictions. We
are not just trying to do better than 0.0; we need to do better than a
random coin flip.

The further away from the diagonal line, the better. Overall, we see
that we see gains in sensitivity (true positive rate, (\> 80%)), trading
off a false positive rate (1- specificity), up until about 15% FPR.
After an FPR of 15%, we don’t see significant gains in TPR for a
tradeoff of increased FPR.

Next, we will proceed to calculate the area under the curve, or, the
AUC. Calculating the Area Under Curve gives us a measure of how well the
data are being predicted, overall, by the model. Note that the area of
entire square is 1\*1 = 1. Hence AUC itself is the ratio under the curve
and the total area. For the case in hand, we get AUC ROC as 96.4%.
Following are a few rules of thumb from Hosmer and Lemeshow, Fielding
and Bell, etc:

- 0.90-1 = excellent (A)
- 0.80-.90 = good (B)
- 0.70-.80 = fair (C)
- 0.60-.70 = poor (D)
- 0.50-.60 = fail (F) - barely better than guessing at random by
  flipping a coin.

``` r
BMauc <- performance(pred, measure="auc") 
str(BMauc)
```

    ## Formal class 'performance' [package "ROCR"] with 6 slots
    ##   ..@ x.name      : chr "None"
    ##   ..@ y.name      : chr "Area under the ROC curve"
    ##   ..@ alpha.name  : chr "none"
    ##   ..@ x.values    : list()
    ##   ..@ y.values    :List of 1
    ##   .. ..$ : num 0.868
    ##   ..@ alpha.values: list()

``` r
auc <- as.numeric(BMauc@y.values)
auc
```

    ## [1] 0.8678605

This is the sum of the area under the predicted performance curve we
just plotted, showing that about \~ 86% of the time, we are correctly
classifying the 1’s. But this does not capture the 0’s. For this, we
need to look at the entire ROC plot.

Next, we will Plot ROC Curve with the optimal cut point and AUC

``` r
plot(perfClass, colorize = T, lwd = 5, print.cutoffs.at=seq(0,1,by=0.1),
     text.adj=c(1.2,1.2),
     main = "ROC Curve")
text(0.5, 0.5, "AUC = 0.867")
abline(v=cutoff, col = "red", lwd = 3)
```

![](README_files/figure-gfm/unnamed-chunk-46-1.png)<!-- --> This plot
shows us the ROC curve with the AUC reported, and the optimal cutpoint,
0.23, that best discriminates true 1’s and true 0’s from each other.

Another cost measure that is popular is overall accuracy. This measure
optimizes the correct results, but may be skewed if there are many more
negatives than positives, or vice versa. Let’s get the overall accuracy
for the simple predictions and plot it:

``` r
acc.perf = performance(pred, measure = "acc")
plot(acc.perf)
```

![](README_files/figure-gfm/unnamed-chunk-47-1.png)<!-- --> But again,
we know that overall accuracy is masking changes in the different rates
of classification mistakes from our earlier work looking at how
classification success changes across a range of cutoff values.

## Manually Changing Cutoff Values

Now lets go back and use this cutoff to calculate the Confusion Matrix
ourself manually, p = 0.23 to see what is going on.

``` r
### now lets try a p = of our cutoff
ppused = wolfkde3$fitted.top.biotic>cutoff
table(ppused,wolfkde3$used)
```

    ##        
    ## ppused     0    1
    ##   FALSE 1344   76
    ##   TRUE   378  320

``` r
#### Now, what is specificity? (i.e., the probability of classifying the 1's correctly?)
320/(320+76)
```

    ## [1] 0.8080808

``` r
#### about 80% - Great! But - what happened to our sensitivity (i.e., the probability of classifying the 0's correctly?)
1344 / (1344+378)
```

    ## [1] 0.7804878

``` r
#### so our probability of classifying 0's correctly is about 78%
```

We see that the trade off between TPR and FPR at the optimal cutpoint
leads to a much higher rate of TPR, 1’s, but, at the expense of a
reduced rate of TNR, or, the true negative rates.

Lets look at the confusion matrix now for the optimal cutpoint.

``` r
wolfkde3$pr.top.biotic.used2 <- ifelse(wolfkde3$fitted.top.biotic>cutoff, 1, 0)
xtab2<-table(wolfkde3$pr.top.biotic.used2, wolfkde3$used)
xtab2
```

    ##    
    ##        0    1
    ##   0 1344   76
    ##   1  378  320

``` r
#?confusionMatrix
confusionMatrix(xtab2)
```

    ## Confusion Matrix and Statistics
    ## 
    ##    
    ##        0    1
    ##   0 1344   76
    ##   1  378  320
    ##                                          
    ##                Accuracy : 0.7856         
    ##                  95% CI : (0.7675, 0.803)
    ##     No Information Rate : 0.813          
    ##     P-Value [Acc > NIR] : 0.9993         
    ##                                          
    ##                   Kappa : 0.455          
    ##                                          
    ##  Mcnemar's Test P-Value : <2e-16         
    ##                                          
    ##             Sensitivity : 0.7805         
    ##             Specificity : 0.8081         
    ##          Pos Pred Value : 0.9465         
    ##          Neg Pred Value : 0.4585         
    ##              Prevalence : 0.8130         
    ##          Detection Rate : 0.6346         
    ##    Detection Prevalence : 0.6704         
    ##       Balanced Accuracy : 0.7943         
    ##                                          
    ##        'Positive' Class : 0              
    ## 

Now lets use this cutoff to classify used and avail locations into 1 and
0’s, and make a plot of where this cutoff is using geom_vline() in
ggplot

``` r
## this is our best model classifying used and avail locations into 1 and 0's. 
ggplot(wolfkde3, aes(x=wolfkde3$fitted.top.biotic, fill=usedFactor)) + geom_histogram(binwidth=0.05, position="identity", alpha=0.7) + xlab("Predicted Probability of Wolf Use") + theme(axis.title.x=element_text(size=16)) + geom_vline(xintercept = cutoff, col="red")
```

![](README_files/figure-gfm/unnamed-chunk-50-1.png)<!-- --> This graph
shows the optimal cutpoint based on our data, and illustrates the
problem of confusion and asymmetry between the 0’s and 1’s. Remember as
well that the \# of 0’s in our dataset was completely arbitrary, defined
by us, and chosen to be 1000 for starters (before filtering out NA’s,
etc). This limits our application of these measures, honestly, to
used-available designs, but the principles of cross-classification
success. Nonetheless, I still use these classificaiton tables to
understand the mechanics of how the models predictions are doing at
classifying the 1’s especially.

## EXCERCISE 2: Evaluating the top Environmental Model

Note this is an excellent excercise to do on your own and part of this
weeks homework .

# K-folds Cross Validation

ROC and AUC are useful measures for evaluating the performance of a true
logistic regression model, such as for used-unused designs in RSF
models. However, as Fielding and Bell, and Boyce et al. tell us, when we
use a USED-AVAILABLE model, ROC and AUC are no longer strictly speaking
correct in evaluating predictive performance of a logistic regression
model. Especially for the 0’s. As we will learn, this is because we are
not actually fitting a logistic regression model.

Fortunately, the tried and true method for used-available models, and in
someways, for all statistical models, at evaluating model fit is
cross-validation. In general, cross-validation measures the predictive
performance of a model by comparing the true data, or observations from
a model, to the predicted values.

There are two main types of cross-validation that we are considering
here. First is *Internal* cross validation. The Coefficient of
Determination is an example of this. This measures the ability of the
data that generated the model to be predicted by the mode - sort of a
circular test don’t you think? But - nonetheless, this is what R^2
measures, and, gives a useful sense of how good the model works.

The second, and ‘better’ and strongest model is to conduct truly out of
sample, *external* cross validation. This is more rarely done in
ecological studies, because it involves collecting a new dataset, or,
testing it against new data or regions to evaluate model performance.
But - ultimately, it is this kind of model validation that is the best.

A hybrid form of this that is still an *internal* cross validation
procedure, but attempts to measure how well a model would do in out of
sample cross validaiton is known as *k-folds Cross-Validation*. The
concept is well known in Data Science, dating back to the 1980’s at
least, but as applied to Habitat modeling, was first pioneered by Mark
Boyce in 2002 with his Ecological Modeling paper.

We are going to load the custom function kxv.R from source. See the
kxv.R details for information about this function. It was basically
programmed for Boyce for the 2002 paper

*References*

1.  Boyce, M. S., P. R. Vernier, S. E. Nielsen, and F. K. A.
    Schmiegelow. 2002. Evaluating resource selection functions.
    Ecological modelling 157:281-300.

2.  Wiens, T. S., B. C. Dale, M. S. Boyce, and G. P. Kershaw. 2008.
    Three Way K-Fold Cross-Validation of Resource Selection Functions.
    Ecological Modelling 212:244-255.

3.  Roberts, D. R., V. Bahn, S. Ciuti, M. S. Boyce, J. Elith, G.
    Guillera-Arroita, S. Hauenstein, J. J. Lahoz-Monfort, B.
    Schröder, W. Thuiller, D. I. Warton, B. A. Wintle, F. Hartig,
    and C. F. Dormann. 2017. Cross-validation strategies for data with
    temporal, spatial, hierarchical, or phylogenetic structure.
    Ecography 40:913-929.

## Evaluating the Top Environmental Model with k-folds

First, we will run a function from Source, that is, without loading it
directly from R code. This is the first time we have done this in the
class, but its an increasingly useful way to do things that you don’t
really care about looking at again, have already written R code for,
etc. To learn more about running code from source, type: `?source` Here,
we will ‘source’ an R function written by for the Boyce et al. (2002)
paper, evaluating Rsf models I note above. See the Rcode folder and the
vignette folder for a PDF of the function written in standard R format.

``` r
source("Rcode/kxv.R", verbose = FALSE)
```

    ## Module kxv.R loaded.
    ## 
    ## Type kxvtest() to run the following example of a fixed-effects
    ## model with sample Elk data:
    ## 
    ## elk <- read.csv(file="HR_Summer.csv", header=T)
    ## kxvglm( PTTYPE~DEM30M+DEM30M^2+VARCES1+FORGSUM+FORGSUM^2, elk, k=5,nbin=10)
    ## 
    ## The test will do kxvPrintFlag <- TRUE in order to
    ## print a summary of each fitted model, but this
    ## flag is FALSE by default.
    ## 
    ## To turn off plotting, do kxvPlotFlag <- FALSE.

Next, we will fit a 5-fold k=5 cross validation of the top.env model

``` r
# Kfolds with a 'fuzz' factor
kxvPrintFlag=FALSE
kxvPlotFlag=TRUE
kxvFuzzFactor = 0.01
kfolds = kxvglm(top.env$formula, data=wolfkde3, k=5, nbin=10) ## note we get a lot of ties here and some error messages, this is because of all the categories. Read more about this in the clumpy data.pdf in the Vignette folder.  
```

![](README_files/figure-gfm/unnamed-chunk-52-1.png)<!-- -->

``` r
kfolds
```

    ##                r.rho            p
    ## 1          0.9660918 5.551570e-06
    ## 2          0.9692234 3.782090e-06
    ## 3          0.8845386 6.749108e-04
    ## 4          0.9251873 1.251260e-04
    ## 5          0.9284518 1.050963e-04
    ## (meanfreq) 0.9695302 3.634851e-06

First, we got some error messages about tie values. Lets overlook that
for now, but refer to Boyce et al or the kvx.pdf vignette for more
detail.

These values tell you the Spearman rank correlation between every subset
of the data 1 - 5 and the predicted correlation between the \# of
observations in ranked categories of habitat from 1, 10.

But note that this K-folds cross validation does not address the
structure of the data within wolf packs. Obviously, the next step is to
subset by wolf pack and see how well the overall wolf model predicts
wolf use by both wolf packs

``` r
# Kfolds by each pack with a 'fuzz' factor
kxvPrintFlag=FALSE
kxvPlotFlag=TRUE
kxvFuzzFactor = 0.01
kfolds2 = kxvglm(top.env$formula, data=wolfkde3, k=5, nbin=10, partition="pack")
```

    ## Warning in all(diff(binbound)): coercing argument of type 'double' to logical

    ## Warning in cor.test.default(counts, 1:nbin, method = "spearman"): Cannot compute
    ## exact p-value with ties

    ## Warning in all(diff(binbound)): coercing argument of type 'double' to logical

    ## Warning in cor.test.default(counts, 1:nbin, method = "spearman"): Cannot compute
    ## exact p-value with ties

![](README_files/figure-gfm/unnamed-chunk-53-1.png)<!-- -->

``` r
kfolds2
```

    ##                 r.rho           p
    ## Bow Valley 0.73556571 0.015323456
    ## Red Deer   0.06707442 0.853933045
    ## (meanfreq) 0.80606061 0.008235571

So the answer is that the overall pooled model predicts the Bow Valley
pack really well, but fails to predict the Red Deer pack any better,
essentially, than random. This is obvious evidence that there is
unmodeled heterogeneity between wolf packs in this dataset that is being
ignored or masked when we fit a model that does not accomodate this.

Excercise 3: Evaluating the top Biotic Model -

On your own for homework - again, a great excercise.

## Manual k-folds Cross-Validation

To really get an idea of what k-folds cross validation is doing, truly,
we will evaluate the steps of at least 1 loop (k=1 of 5) MANUALLY so we
can see exactly what k-folds is doing. The steps are:

1.  First, we crete a vector of random ‘folds’, or subsets of the
    dataset, by default here 5.
2.  Run the model for each random subset i = 1..5
3.  Make predictions from model subset i=1, for the other subsets NOT in
    the model, i.e., 2…5.
4.  Make quantiles for the predictions - this calculates the ’bin’s of
    the categories of habitat availability
5.  Then count up the \# of observed locations (1’s) in each of the 10
    bins of predicted values in the available habitat.

Logically, if the model is working well, then there should be more
predicted 1’s in higher quality ranked ’bin’s/ deciles of prediction
habitat quality. We then do a spearman rank correlation (a
non-parametric correlation test) to test whether the frequency of
predicted 1’s is indeed higher in higher ranked habitat classes.

``` r
# 1. Create a vector of random "folds" in this case 5, 1:5
wolfkde3$rand.vec = sample(1:5,nrow(wolfkde3),replace=TRUE)

#2. Run the model for 1 single random subset of the data == 1
top.env.1= glm(used ~ Elevation2 + DistFromHighHumanAccess2 + openConif+modConif+closedConif+mixed+herb+shrub+water+burn, family=binomial(logit), data=wolfkde3, subset=rand.vec==1) ## note this last subset = rand.vec==1 is just 1 of the 5 subsets. 

# 3. Make predictions for points not used in this random subset (2:5) to fit the model.
pred.prob = predict(top.env.1,newdata=wolfkde3[wolfkde3$rand.vec!=1,],type="response") ## note that != means everything but subset 1. 

# 4. Make quantiles for the predictions - this calculates the 'bin's of the categories of habitat availability
q.pp = quantile(pred.prob,probs=seq(0,1,.1))

# 5. Then for each of 10 bins, put each row of data into a bin
bin = rep(NA,length(pred.prob))
for (i in 1:10){
    bin[pred.prob>=q.pp[i]&pred.prob<q.pp[i+1]] = i
}

## 5. This then makes a count of just the used locations for all other K folds 2:5 
used1 = wolfkde3$used[wolfkde3$rand.vec!=1]

## We then make a table of them
rand.vec.1.table <- table(used1,bin)
rand.vec.1.table
```

    ##      bin
    ## used1   1   2   3   4   5   6   7   8   9  10
    ##     0 157 169 166 165 159 154 137 119  79  61
    ##     1  12   0   3   4   9  15  32  50  90 107

This basically shows the data in each bin for used and available
locations. If the model is any ‘good’, then high ranking habitat bins
(7, 8, 9, 10) should have a lot more used locations in them.

``` r
cor.test(c(1, 2, 3, 4, 5, 6, 7, 8, 9, 10), c(0,2,0,8,6,15,24,50,99,110), method="spearman") 
```

    ## Warning in cor.test.default(c(1, 2, 3, 4, 5, 6, 7, 8, 9, 10), c(0, 2, 0, :
    ## Cannot compute exact p-value with ties

    ## 
    ##  Spearman's rank correlation rho
    ## 
    ## data:  c(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) and c(0, 2, 0, 8, 6, 15, 24, 50, 99, 110)
    ## S = 5.516, p-value = 5.248e-06
    ## alternative hypothesis: true rho is not equal to 0
    ## sample estimates:
    ##       rho 
    ## 0.9665698

This suggests that in this random fold of the data, the model predicted
habitat use well

Finally, we can look at a cheesy graph of this 1 fold:

``` r
rand.vec.1.table <- as.data.frame(rand.vec.1.table)
ggplot(rand.vec.1.table, aes(as.numeric(bin), Freq, col = used1)) + geom_point(size=5) + geom_line()
```

![](README_files/figure-gfm/unnamed-chunk-56-1.png)<!-- --> This shows
the observed frequency of 1’s in blue and observed frequency of 0’s
(red) plotted against the predicted ranked equal decile ranked ‘bins’ of
increasing habitat quality from 1 - 10, the similar K-folds graphs (for
1 fold, rand.vec ==1) for the top environmental model.

Excercise: Conduct k-folds cross validation ont the top-biotic models.

# Mapping Spatial Predictions of ‘Top’ Model

Spatial prediction is usually the last and final step in model
evaluation. During mapping, we learn a great deal about how well the
model fits in geographic space, and, deal with issues of interpolation
and extrapolation which we will discuss in class.

## Easy Spatial Prediction just in ggplot

It can be unwieldy to mess with big raster stacks for quick and dirty
spatial evaluation of how a model might ‘look’ when mapped back to
geographic space. Often, I use some simple ggplots first to explore
things.

``` r
par(mfrow = c(1,1)) # reset graphical parameters

ggplot(wolfkde3, aes(EASTING, NORTHING, col = fitted.top.biotic)) + geom_point(size=5) + coord_equal() +  scale_colour_gradient(low = 'yellow', high = 'red')
```

![](README_files/figure-gfm/unnamed-chunk-57-1.png)<!-- -->

``` r
ggplot(wolfkde3, aes(EASTING, NORTHING, col = fitted.top.env)) + geom_point(size=5) + coord_equal() +  scale_colour_gradient(low = 'yellow', high = 'red')
```

![](README_files/figure-gfm/unnamed-chunk-57-2.png)<!-- --> What did we
just do? What is this a map of? The predicted probabilities in
geographic space for our top biotic and top environmental models within
the home ranges (using the actual XY’s of the datset, so including 0’s
and 1’s) within the 2 territories mapped by kernels. Note we are not
technically extrapolating here.

## Raster Predictions

Next we will plot the spatial predictions using our raster stack for the
top Biotic Model, passed through the logistic regression equation y =
exp(X) / (1 + exp(X). Note that the final step takes a long time.

``` r
par(mfrow = c(1,1))
top.biotic$coefficients
```

    ##          (Intercept) DistFromHumanAccess2              deer_w2 
    ##         -3.553037526         -0.001421547          0.898069385 
    ##              goat_w2 
    ##         -0.333539833

``` r
biotic.coefs <- top.biotic$coefficients[c(1:4)]
```

Now we will calculate the probability of wolf use using the logistic
regression equation y = exp(x) / (1 + exp(x)). Note that this step takes
a long time.

``` r
rast.top.biotic <- exp(biotic.coefs[1] + biotic.coefs[2]*disthumanaccess2 + biotic.coefs[3]*deer_w + biotic.coefs[4]*goat_w) / (1 +exp(biotic.coefs[1] + biotic.coefs[2]*disthumanaccess2 + biotic.coefs[3]*deer_w + biotic.coefs[4]*goat_w ))
```

Note we need to use the names of the raster layers we brought in up
above. Note that they are not the same names as stored in the Raster
stack.

Lets bring in the wolfyht shapefile, and overlay the wolf home ranges
and telemetry points to overlap to also aid our VISUAL model evaluation.

``` r
wolfyht<-st_read("Data/wolfyht.shp")
```

    ## Reading layer `wolfyht' from data source 
    ##   `C:\Users\Administrator.KJLWS11\Documents\Lab6\Data\wolfyht.shp' 
    ##   using driver `ESRI Shapefile'
    ## Simple feature collection with 413 features and 21 fields
    ## Geometry type: POINT
    ## Dimension:     XY
    ## Bounding box:  xmin: 555853 ymin: 5656997 xmax: 605389 ymax: 5741316
    ## Projected CRS: NAD83 / UTM zone 11N

``` r
# plot predicted raster within extent of kernels raster
plot(rast.top.biotic, col=colorRampPalette(c("yellow", "orange", "red"))(255), ext=kernels)
plot(kernelHR, add=TRUE, col = NA)
```

    ## Warning in plot.sf(kernelHR, add = TRUE, col = NA): ignoring all but the first
    ## attribute

``` r
plot(wolfyht, col='blue', pch = 16, add=TRUE)
```

    ## Warning in plot.sf(wolfyht, col = "blue", pch = 16, add = TRUE): ignoring all
    ## but the first attribute

![](README_files/figure-gfm/unnamed-chunk-60-1.png)<!-- --> We note a
number of issues with this plot. First, there are no predictions outside
of Banff National Park, the extent of the underlying Deer, elk, moose
models. This is a common problem in spatial modeling, mismatched extents
limit spatial predictions.

Second, we note we are now firmly in the realm of extrapolation, that
is, making predictions beyond the range of data that went into the
model. In this case, this is predicting the probability of wolf use
OUTSIDE of the 95% KDE home ranges in black. This is OK, but we will see
this can lead to problems later.

Next, lets take a look at histogram of predicted values from this
surface.

``` r
#hist(rast.top.biotic@data@values) ## weird, this worked in R but not in R markdown. Try it on your own. 
```

This confirms that most of the study area is rock and ice, high
elevaiton, low probability of use. OK, not surprising here.

Lets zoom in to a specific area in the Bow Valley, and examine how the
spatial predictions are performing.

``` r
##
bv.raster<-rast()
ext(bv.raster) <- c(xmin=570000, xmax=600000, ymin=5665000, ymax=5685000) 
plot(rast.top.biotic, col=colorRampPalette(c("yellow", "orange", "red"))(255), ext=bv.raster)
plot(kernelHR, add=TRUE, col = NA)
```

    ## Warning in plot.sf(kernelHR, add = TRUE, col = NA): ignoring all but the first
    ## attribute

``` r
plot(wolfyht, col='blue', pch = 16, add=TRUE)
```

    ## Warning in plot.sf(wolfyht, col = "blue", pch = 16, add = TRUE): ignoring all
    ## but the first attribute

![](README_files/figure-gfm/unnamed-chunk-62-1.png)<!-- -->

Lets zoom in to a specific area in the Red Deer Pack, and examine how
the spatial predictions are performing.

``` r
##
rd.raster<-rast()
ext(rd.raster) <- c(xmin=540000, xmax=600000, ymin=5700000, ymax=5730000) 
plot(rast.top.biotic, col=colorRampPalette(c("yellow", "orange", "red"))(255), ext=rd.raster)
plot(kernelHR, add=TRUE, col = NA)
```

    ## Warning in plot.sf(kernelHR, add = TRUE, col = NA): ignoring all but the first
    ## attribute

``` r
plot(wolfyht, col='blue', pch = 16, add=TRUE)
```

    ## Warning in plot.sf(wolfyht, col = "blue", pch = 16, add = TRUE): ignoring all
    ## but the first attribute

![](README_files/figure-gfm/unnamed-chunk-63-1.png)<!-- --> OVerall, our
spatial model predictions seem to be matching the wolf used
distributions very well! Bravo - we have just fit our first RSF model!

# Lab 6 Excercises

1.  Building on the top model you identified for both the environmental
    suite of models and the biotic-species suite of models, compare the
    results of goodness of fit testing for both models. Write up your
    results of whichever of the following tests you used for assessing
    model fit – Pseudo-R2 values, classification tables, and ROC scores.
    Which is the ‘better’ model?

2.  Compare the classification success of your 2 models with the default
    cutpoint probability (0.5) with that revealed by the sensitivity –
    specificity trade off? Discuss some reasons why the optimal cutpoint
    probability may be \<0.5 in our case? How would your sampling
    design, species biology, etc., affect the cutpoint probability?

3.  Now, report on k-folds cross validation results for both models.
    Explain what k-folds cross validation is doing, what hypothesis is
    it testing, and report the results using a graph and spearman rank
    correlations. Now, comparing measures of model fit and k-folds,
    which is the best model?

THIS IS AN ADVANCED QUESTION - I ENCOURAGE YOU TO WORK IN GROUPS FOR
THIS QUESTION AND TO CONSIDER USING SOMETHING SIMILAR IN YOUR OWN
ANALYSES

4.  We have made ‘population’ level predictions for ALL WOLVES IN BNP
    using the averaged top RSF model for the two suites of models.
    Re-evaluate what the top habitat-dependent model is for each wolf
    pack and then conduct cross-validation of the predictions of one
    pack against the other pack.

Here are some hints for steps; a. Re-run the biotic and environmental
RSF’s for just the Bow Valley pack. Then repeat this step for the Red
Deer pack.

2.  With the results of the Bow Valley pack RSF, use this model to
    predict the probability of use for the Red Deer pack, and vice
    versa.

3.  Conduct model evaluation using the R-squared from a linear
    regression of the prediction for the Bow Valley pack using the Bow
    Valley model vs. the Red Deer Model, and vice versa. What does the
    slope parameter for this model represent?

4.  Can you think of how to do k-folds cross validation between packs?
    i.e., compare the area adjusted frequency of counts in each of 10
    ranked deciles to that predicted for the other pack??? What does the
    k-folds cross validation tell us about differences between
    predictions at the population level and predictions at the
    pack-level??? Follow the lab instructions for manually calculating
    k-folds above using the excel spreadsheets AT the very least. OR -
    do in R using the ‘manual’ code we used at the end of lab. How do
    these cross-validation between packs compare to the ‘naive’ k-folds
    we conducted partitioning by wolf pack in lab (e.g., by setting
    partition = “pack”).
