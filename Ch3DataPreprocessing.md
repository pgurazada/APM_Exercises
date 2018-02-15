Preprocessing Data
================
Pavan Gurazada
February 2018

``` r
library(AppliedPredictiveModeling)
library(caret)
library(tidyverse)

library(mlbench)

set.seed(20130810)
theme_set(theme_bw())
dev.new()
```

Here we are concerned about the preprocessing of features only. The outcome is kept out of the discussion.

Preprocessing is usually referred to as 'feature engineering' in practise.

All feature engineering decisions are made based on the training data and applied to test data.

The usual suspects for feature engineering are:

1.  Centering and scaling

2.  Box-Cox transformations with lambda derived using MLE

3.  Outlier treatment using spatial sign tranformation

4.  PCA for reduction to a small subset of orthogonal features. The number of features eventually selected is based on a scree plot or using cross-validation

5.  Removal of features with near-zero variance

6.  Removal of features exhibiting multicollinearity. Easiest way to deal with this is to assert that the pairwise correlation between any two features is below a certain threshold (say 0.75)

7.  Addition of dummy variables to split categorical data (one dummy variable per category, unless it is a regression setup)

Another pesky issue in feature engineering is the handling of missing values. Always check if missing values in a data set are concentrated in a small subset of predictors. This may be informative about the data generation mechanism. Based on an anlysis of missing values, it might be beneficial to either remove the offending features or the offending samples.

In case a decision has been made to not remove missing values, the first pass would be to see if the features with tons of missing values are correlated with any of those that do not have missing values. If this also does not work, an imputation model needs to be applied to infer the missing values from available data. This needs to be done with care and incorporated into the model parameter tuning too.

A widely applied method for imputation is K-Nearest Neighbors. Note that both the number of nearest neighbors and the distance metric are parameters in this case.

``` r
data("segmentationOriginal")
glimpse(segmentationOriginal)
```

    ## Observations: 2,019
    ## Variables: 119
    ## $ Cell                          <int> 207827637, 207932307, 207932463,...
    ## $ Case                          <fct> Test, Train, Train, Train, Test,...
    ## $ Class                         <fct> PS, PS, WS, PS, PS, WS, WS, PS, ...
    ## $ AngleCh1                      <dbl> 143.247705, 133.752037, 106.6463...
    ## $ AngleStatusCh1                <int> 1, 0, 0, 0, 2, 2, 1, 1, 2, 1, 2,...
    ## $ AreaCh1                       <int> 185, 819, 431, 298, 285, 172, 17...
    ## $ AreaStatusCh1                 <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ AvgIntenCh1                   <dbl> 15.71186, 31.92327, 28.03883, 19...
    ## $ AvgIntenCh2                   <dbl> 3.954802, 205.878517, 115.315534...
    ## $ AvgIntenCh3                   <dbl> 9.548023, 69.916880, 63.941748, ...
    ## $ AvgIntenCh4                   <dbl> 2.214689, 164.153453, 106.696602...
    ## $ AvgIntenStatusCh1             <int> 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 1,...
    ## $ AvgIntenStatusCh2             <int> 2, 0, 0, 0, 0, 1, 1, 2, 0, 0, 1,...
    ## $ AvgIntenStatusCh3             <int> 2, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0,...
    ## $ AvgIntenStatusCh4             <int> 2, 0, 0, 2, 0, 1, 0, 2, 0, 0, 0,...
    ## $ ConvexHullAreaRatioCh1        <dbl> 1.124509, 1.263158, 1.053310, 1....
    ## $ ConvexHullAreaRatioStatusCh1  <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ ConvexHullPerimRatioCh1       <dbl> 0.9196827, 0.7970801, 0.9354750,...
    ## $ ConvexHullPerimRatioStatusCh1 <int> 0, 2, 0, 2, 0, 1, 1, 2, 2, 2, 0,...
    ## $ DiffIntenDensityCh1           <dbl> 29.51923, 31.87500, 32.48771, 26...
    ## $ DiffIntenDensityCh3           <dbl> 13.77564, 43.12228, 35.98577, 22...
    ## $ DiffIntenDensityCh4           <dbl> 6.826923, 79.308424, 51.357050, ...
    ## $ DiffIntenDensityStatusCh1     <int> 2, 0, 0, 2, 0, 1, 1, 2, 2, 2, 0,...
    ## $ DiffIntenDensityStatusCh3     <int> 2, 0, 0, 0, 0, 1, 0, 0, 2, 0, 0,...
    ## $ DiffIntenDensityStatusCh4     <int> 2, 0, 0, 2, 2, 1, 1, 2, 0, 0, 0,...
    ## $ EntropyIntenCh1               <dbl> 4.969781, 6.087592, 5.883557, 5....
    ## $ EntropyIntenCh3               <dbl> 4.371017, 6.642761, 6.683000, 5....
    ## $ EntropyIntenCh4               <dbl> 2.718884, 7.880155, 7.144601, 5....
    ## $ EntropyIntenStatusCh1         <int> 2, 0, 0, 2, 2, 0, 0, 2, 2, 2, 1,...
    ## $ EntropyIntenStatusCh3         <int> 0, 1, 1, 0, 0, 1, 1, 0, 0, 0, 0,...
    ## $ EntropyIntenStatusCh4         <int> 2, 1, 0, 0, 0, 0, 0, 2, 0, 0, 0,...
    ## $ EqCircDiamCh1                 <dbl> 15.36954, 32.30558, 23.44892, 19...
    ## $ EqCircDiamStatusCh1           <int> 0, 1, 0, 0, 0, 2, 2, 0, 0, 0, 0,...
    ## $ EqEllipseLWRCh1               <dbl> 3.060676, 1.558394, 1.375386, 3....
    ## $ EqEllipseLWRStatusCh1         <int> 1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0,...
    ## $ EqEllipseOblateVolCh1         <dbl> 336.9691, 2232.9055, 802.1945, 7...
    ## $ EqEllipseOblateVolStatusCh1   <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ EqEllipseProlateVolCh1        <dbl> 110.0963, 1432.8246, 583.2504, 2...
    ## $ EqEllipseProlateVolStatusCh1  <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ EqSphereAreaCh1               <dbl> 742.1156, 3278.7256, 1727.4104, ...
    ## $ EqSphereAreaStatusCh1         <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ EqSphereVolCh1                <dbl> 1900.996, 17653.525, 6750.985, 3...
    ## $ EqSphereVolStatusCh1          <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ FiberAlign2Ch3                <dbl> 0.00000000, 0.48793541, 0.300521...
    ## $ FiberAlign2Ch4                <dbl> 0.000000000, 0.352374247, 0.5223...
    ## $ FiberAlign2StatusCh3          <int> 2, 0, 0, 0, 0, 0, 0, 2, 0, 1, 0,...
    ## $ FiberAlign2StatusCh4          <int> 2, 0, 0, 1, 0, 0, 0, 2, 0, 1, 0,...
    ## $ FiberLengthCh1                <dbl> 26.98132, 64.28230, 21.14115, 43...
    ## $ FiberLengthStatusCh1          <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ FiberWidthCh1                 <dbl> 7.410365, 13.167079, 21.141150, ...
    ## $ FiberWidthStatusCh1           <int> 2, 0, 1, 2, 2, 0, 0, 2, 0, 0, 1,...
    ## $ IntenCoocASMCh3               <dbl> 0.011183899, 0.028051061, 0.0068...
    ## $ IntenCoocASMCh4               <dbl> 0.050448005, 0.012594975, 0.0061...
    ## $ IntenCoocASMStatusCh3         <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ IntenCoocASMStatusCh4         <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ IntenCoocContrastCh3          <dbl> 40.751777, 8.227953, 14.446074, ...
    ## $ IntenCoocContrastCh4          <dbl> 13.895439, 6.984046, 16.700843, ...
    ## $ IntenCoocContrastStatusCh3    <int> 1, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0,...
    ## $ IntenCoocContrastStatusCh4    <int> 1, 0, 1, 1, 2, 0, 1, 0, 0, 1, 2,...
    ## $ IntenCoocEntropyCh3           <dbl> 7.199458, 6.822138, 7.580100, 6....
    ## $ IntenCoocEntropyCh4           <dbl> 5.249744, 7.098988, 7.671478, 7....
    ## $ IntenCoocEntropyStatusCh3     <int> 0, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0,...
    ## $ IntenCoocEntropyStatusCh4     <int> 0, 0, 1, 0, 0, 0, 0, 2, 0, 1, 0,...
    ## $ IntenCoocMaxCh3               <dbl> 0.07741935, 0.15321477, 0.028350...
    ## $ IntenCoocMaxCh4               <dbl> 0.17197452, 0.07387141, 0.023195...
    ## $ IntenCoocMaxStatusCh3         <int> 0, 0, 2, 0, 0, 2, 2, 2, 0, 0, 0,...
    ## $ IntenCoocMaxStatusCh4         <int> 0, 0, 2, 0, 0, 2, 2, 1, 0, 0, 0,...
    ## $ KurtIntenCh1                  <dbl> -0.656744087, -0.248769067, -0.2...
    ## $ KurtIntenCh3                  <dbl> -0.608058268, -0.330783900, 1.05...
    ## $ KurtIntenCh4                  <dbl> 0.7258145, -0.2652638, 0.1506140...
    ## $ KurtIntenStatusCh1            <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ KurtIntenStatusCh3            <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ KurtIntenStatusCh4            <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ LengthCh1                     <dbl> 26.20779, 47.21855, 28.14303, 37...
    ## $ LengthStatusCh1               <int> 0, 1, 0, 0, 0, 2, 2, 0, 0, 0, 0,...
    ## $ MemberAvgAvgIntenStatusCh2    <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ MemberAvgTotalIntenStatusCh2  <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ NeighborAvgDistCh1            <dbl> 370.4543, 174.4442, 158.4774, 20...
    ## $ NeighborAvgDistStatusCh1      <int> 1, 2, 2, 0, 0, 0, 0, 0, 0, 0, 1,...
    ## $ NeighborMinDistCh1            <dbl> 99.10349, 30.11114, 34.94477, 33...
    ## $ NeighborMinDistStatusCh1      <int> 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ NeighborVarDistCh1            <dbl> 127.96080, 81.38063, 90.43768, 1...
    ## $ NeighborVarDistStatusCh1      <int> 0, 2, 2, 0, 0, 0, 0, 0, 2, 2, 0,...
    ## $ PerimCh1                      <dbl> 68.78338, 154.89876, 84.56460, 1...
    ## $ PerimStatusCh1                <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ ShapeBFRCh1                   <dbl> 0.6651480, 0.5397584, 0.7243116,...
    ## $ ShapeBFRStatusCh1             <int> 0, 2, 1, 0, 0, 0, 1, 0, 0, 0, 1,...
    ## $ ShapeLWRCh1                   <dbl> 2.462450, 1.468181, 1.328408, 2....
    ## $ ShapeLWRStatusCh1             <int> 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0,...
    ## $ ShapeP2ACh1                   <dbl> 1.883006, 2.255810, 1.272193, 2....
    ## $ ShapeP2AStatusCh1             <int> 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0,...
    ## $ SkewIntenCh1                  <dbl> 0.45450484, 0.39870467, 0.472487...
    ## $ SkewIntenCh3                  <dbl> 0.46039340, 0.61973079, 0.971378...
    ## $ SkewIntenCh4                  <dbl> 1.2327736, 0.5272631, 0.3247065,...
    ## $ SkewIntenStatusCh1            <int> 0, 0, 0, 1, 0, 2, 2, 0, 0, 0, 2,...
    ## $ SkewIntenStatusCh3            <int> 0, 0, 0, 0, 0, 2, 2, 2, 0, 0, 0,...
    ## $ SkewIntenStatusCh4            <int> 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0,...
    ## $ SpotFiberCountCh3             <int> 1, 4, 2, 4, 1, 1, 0, 2, 1, 1, 1,...
    ## $ SpotFiberCountCh4             <int> 4, 11, 6, 7, 7, 4, 4, 7, 11, 7, ...
    ## $ SpotFiberCountStatusCh3       <int> 0, 1, 0, 1, 0, 0, 2, 0, 0, 0, 0,...
    ## $ SpotFiberCountStatusCh4       <int> 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0,...
    ## $ TotalIntenCh1                 <int> 2781, 24964, 11552, 5545, 6603, ...
    ## $ TotalIntenCh2                 <int> 700, 160997, 47510, 28869, 30305...
    ## $ TotalIntenCh3                 <int> 1690, 54675, 26344, 8042, 5569, ...
    ## $ TotalIntenCh4                 <int> 392, 128368, 43959, 8843, 11037,...
    ## $ TotalIntenStatusCh1           <int> 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1,...
    ## $ TotalIntenStatusCh2           <int> 2, 1, 0, 0, 0, 1, 1, 2, 0, 0, 1,...
    ## $ TotalIntenStatusCh3           <int> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ TotalIntenStatusCh4           <int> 2, 1, 0, 2, 0, 0, 0, 2, 0, 0, 0,...
    ## $ VarIntenCh1                   <dbl> 12.47468, 18.80923, 17.29564, 13...
    ## $ VarIntenCh3                   <dbl> 7.609035, 56.715352, 37.671053, ...
    ## $ VarIntenCh4                   <dbl> 2.714100, 118.388139, 49.470524,...
    ## $ VarIntenStatusCh1             <int> 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0,...
    ## $ VarIntenStatusCh3             <int> 2, 0, 0, 0, 0, 0, 0, 2, 0, 2, 0,...
    ## $ VarIntenStatusCh4             <int> 2, 0, 0, 2, 0, 0, 0, 2, 0, 2, 0,...
    ## $ WidthCh1                      <dbl> 10.64297, 32.16126, 21.18553, 13...
    ## $ WidthStatusCh1                <int> 2, 1, 0, 0, 0, 0, 0, 2, 1, 0, 0,...
    ## $ XCentroid                     <int> 42, 215, 371, 487, 283, 191, 180...
    ## $ YCentroid                     <int> 14, 347, 252, 295, 159, 127, 138...

The code below performs the followng steps:

1.  Filter the training data

2.  Remove the predictor and sample id features

3.  Remove redundant columns that contain "Status" in their name

``` r
segTrain <- segmentationOriginal %>% filter(Case == "Train") %>% 
                                     select(-Cell, -Class, -Case) %>% 
                                     select(-contains("Status"))
```

Next we apply a pre-processing pipeline to the training data, do a box-cox transform, center and scale the data and apply a PCA. The output is a model object.

``` r
trans <- segTrain %>% preProcess(method = c("BoxCox", "center", "scale", "pca"))
```

These transformations need to be applied to the data

``` r
transSegTrain <- predict(trans, segTrain)
glimpse(transSegTrain) # you should onl see the 19 principal components here
```

    ## Observations: 1,009
    ## Variables: 19
    ## $ PC1  <dbl> 1.5684742, -0.6664055, 3.7500055, 0.3768509, 1.0644951, -...
    ## $ PC2  <dbl> 6.2907855, 2.0455375, -0.3915610, -2.1897554, -1.4646516,...
    ## $ PC3  <dbl> -0.33332995, -1.44168410, -0.66902601, 1.43801667, -0.990...
    ## $ PC4  <dbl> -3.06332674, -4.70118302, -4.02075287, -5.32711644, -5.62...
    ## $ PC5  <dbl> -1.3415782, -1.7422020, 1.7927777, -0.4066757, -0.8650174...
    ## $ PC6  <dbl> 0.3933609, 0.4313114, -0.8542507, 1.1092318, 0.1070749, -...
    ## $ PC7  <dbl> -1.317794806, 1.284502479, -0.070927238, 0.702318738, 0.4...
    ## $ PC8  <dbl> -1.8965728, -3.0829008, -0.5997223, -0.9667673, -0.656911...
    ## $ PC9  <dbl> 0.7111800857, 1.9973302847, 0.9873784383, 0.4970412086, 0...
    ## $ PC10 <dbl> 0.16193272, 0.58665039, -0.47230884, -0.10925035, -0.0165...
    ## $ PC11 <dbl> 1.44061816, 0.80080447, 1.22229470, 1.59963522, 0.0115479...
    ## $ PC12 <dbl> -0.664707822, 1.448093530, 1.127734837, -0.666573808, -0....
    ## $ PC13 <dbl> -0.50341167, 0.44875803, -1.37477652, -1.26751477, -0.068...
    ## $ PC14 <dbl> -0.5251037, -0.4299460, -1.4884756, -0.2010528, -0.292584...
    ## $ PC15 <dbl> 0.209541528, -0.610433146, -0.712689953, 0.135890457, 0.0...
    ## $ PC16 <dbl> 0.001408739, -1.058349329, -0.359746977, -1.125603114, -0...
    ## $ PC17 <dbl> 0.783699478, -0.791062674, -0.002913506, -0.025376331, -0...
    ## $ PC18 <dbl> -0.55515083, 0.06569274, 1.35736326, -0.63185874, -0.5846...
    ## $ PC19 <dbl> 0.68129112, 0.14157724, 0.10187098, -0.96813663, 0.050848...

This highlights a disadvantage of PCA, i.e., the loss of interpretability of features.

PCA essentially projects the data onto a vector pointed in the direction of maximum variance. The first direction is along the eigen vector corresponding to the largest eigen value and so on.

An alternative pathway is to remove zero variance features and features that are highly correlated Check for degenerate features

``` r
length(nearZeroVar(segTrain))> 0
```

    ## [1] FALSE

Check for highly correlated features Compute the correlation among all features, then filter beyond a cutoff

``` r
highCorrVars <- cor(segTrain) %>% findCorrelation(cutoff = 0.75)
segTrain <- segTrain %>% select(-highCorrVars)
glimpse(segTrain)
```

    ## Observations: 1,009
    ## Variables: 26
    ## $ AngleCh1                <dbl> 133.75204, 106.64639, 69.15032, 109.41...
    ## $ ConvexHullPerimRatioCh1 <dbl> 0.7970801, 0.9354750, 0.8658291, 0.920...
    ## $ EntropyIntenCh1         <dbl> 6.087592, 5.883557, 5.420065, 5.383272...
    ## $ FiberAlign2Ch3          <dbl> 0.48793541, 0.30052198, 0.22042390, 0....
    ## $ FiberAlign2Ch4          <dbl> 0.35237425, 0.52231582, 0.73325044, 0....
    ## $ FiberWidthCh1           <dbl> 13.167079, 21.141150, 7.404412, 12.057...
    ## $ IntenCoocASMCh3         <dbl> 0.028051061, 0.006862315, 0.030962071,...
    ## $ IntenCoocASMCh4         <dbl> 0.012594975, 0.006141691, 0.011033195,...
    ## $ IntenCoocContrastCh3    <dbl> 8.2279529, 14.4460738, 7.2994574, 6.16...
    ## $ IntenCoocContrastCh4    <dbl> 6.984046, 16.700843, 13.390884, 10.590...
    ## $ KurtIntenCh1            <dbl> -0.24876907, -0.29348463, 0.62585612, ...
    ## $ KurtIntenCh3            <dbl> -0.3307839, 1.0512813, 0.1277406, 1.08...
    ## $ KurtIntenCh4            <dbl> -0.2652638, 0.1506140, -0.3472936, -0....
    ## $ NeighborAvgDistCh1      <dbl> 174.4442, 158.4774, 206.3344, 263.6345...
    ## $ NeighborMinDistCh1      <dbl> 30.11114, 34.94477, 33.08030, 38.43038...
    ## $ ShapeBFRCh1             <dbl> 0.5397584, 0.7243116, 0.5891625, 0.634...
    ## $ ShapeLWRCh1             <dbl> 1.468181, 1.328408, 2.826854, 1.313937...
    ## $ SpotFiberCountCh3       <int> 4, 2, 4, 0, 1, 1, 4, 2, 2, 2, 6, 4, 0,...
    ## $ SpotFiberCountCh4       <int> 11, 6, 7, 5, 4, 5, 4, 2, 5, 1, 10, 3, ...
    ## $ TotalIntenCh2           <int> 160997, 47510, 28869, 30855, 30719, 74...
    ## $ VarIntenCh1             <dbl> 18.80923, 17.29564, 13.81897, 13.92294...
    ## $ VarIntenCh3             <dbl> 56.71535, 37.67105, 30.00564, 18.64303...
    ## $ VarIntenCh4             <dbl> 118.38814, 49.47052, 24.74954, 40.3317...
    ## $ WidthCh1                <dbl> 32.16126, 21.18553, 13.39283, 17.54686...
    ## $ XCentroid               <int> 215, 371, 487, 211, 172, 276, 239, 95,...
    ## $ YCentroid               <int> 347, 252, 295, 495, 207, 385, 404, 95,...

Exercise 3.1

``` r
data("Glass")
glimpse(Glass)
```

    ## Observations: 214
    ## Variables: 10
    ## $ RI   <dbl> 1.52101, 1.51761, 1.51618, 1.51766, 1.51742, 1.51596, 1.5...
    ## $ Na   <dbl> 13.64, 13.89, 13.53, 13.21, 13.27, 12.79, 13.30, 13.15, 1...
    ## $ Mg   <dbl> 4.49, 3.60, 3.55, 3.69, 3.62, 3.61, 3.60, 3.61, 3.58, 3.6...
    ## $ Al   <dbl> 1.10, 1.36, 1.54, 1.29, 1.24, 1.62, 1.14, 1.05, 1.37, 1.3...
    ## $ Si   <dbl> 71.78, 72.73, 72.99, 72.61, 73.08, 72.97, 73.09, 73.24, 7...
    ## $ K    <dbl> 0.06, 0.48, 0.39, 0.57, 0.55, 0.64, 0.58, 0.57, 0.56, 0.5...
    ## $ Ca   <dbl> 8.75, 7.83, 7.78, 8.22, 8.07, 8.07, 8.17, 8.24, 8.30, 8.4...
    ## $ Ba   <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...
    ## $ Fe   <dbl> 0.00, 0.00, 0.00, 0.00, 0.00, 0.26, 0.00, 0.00, 0.00, 0.1...
    ## $ Type <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, ...

Gather all the features into a column "Variable" and its corresponding value into a column "Value". We leave the Type column alone. We can then plot the histogram of the variables at once using a facet wrap and compare values

``` r
meltedGlass <- Glass %>% select(-Type) %>%  
                         gather("Variable", "Value", 1:9)
glimpse(meltedGlass)
```

    ## Observations: 1,926
    ## Variables: 2
    ## $ Variable <chr> "RI", "RI", "RI", "RI", "RI", "RI", "RI", "RI", "RI",...
    ## $ Value    <dbl> 1.52101, 1.51761, 1.51618, 1.51766, 1.51742, 1.51596,...

``` r
ggplot(meltedGlass) +
  geom_histogram(aes(x = Value)) + 
  facet_wrap(~Variable)
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](C:\Users\kimmcodxb\Documents\GitHub\APMExercises\notes\Ch3DataPreprocessing_files/figure-markdown_github/unnamed-chunk-9-1.png)

The plot shows signs of both bimodality and skewness. Check for degenerate features

``` r
length(nearZeroVar(Glass)) > 0 # No problem here
```

    ## [1] FALSE

We look for highly correlated predictors

``` r
highCorrVars <- Glass %>% select(-Type) %>% 
                          cor() %>% 
                          findCorrelation(cutoff = 0.75)
```

This is suggesting that removing Ca might be a good option since it is highly correlated with others Many zeros exist in the data, hence Yeo-Johnson transformation might be better

``` r
yjTrans <- Glass %>% select(-Type) %>% 
                     preProcess(method = "YeoJohnson")

yjTransData <- predict(yjTrans, Glass[, -10]) 
glimpse(yjTransData)
```

    ## Observations: 214
    ## Variables: 9
    ## $ RI <dbl> 1.52101, 1.51761, 1.51618, 1.51766, 1.51742, 1.51596, 1.517...
    ## $ Na <dbl> 2.110448, 2.120686, 2.105878, 2.092344, 2.094909, 2.074022,...
    ## $ Mg <dbl> 19.71831, 13.12961, 12.80174, 13.73095, 13.26199, 13.19571,...
    ## $ Al <dbl> 0.7428059, 0.8598251, 0.9335354, 0.8296351, 0.8075021, 0.96...
    ## $ Si <dbl> 71.78, 72.73, 72.99, 72.61, 73.08, 72.97, 73.09, 73.24, 72....
    ## $ K  <dbl> 0.05663025, 0.32528951, 0.28128447, 0.36428786, 0.35600981,...
    ## $ Ca <dbl> 0.7032119, 0.6983904, 0.6980940, 0.7005729, 0.6997596, 0.69...
    ## $ Ba <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,...
    ## $ Fe <dbl> 0.00, 0.00, 0.00, 0.00, 0.00, 0.26, 0.00, 0.00, 0.00, 0.11,...

Now we relook at the distribution of the variables

``` r
meltedYJTransData <- yjTransData %>% gather("Variable", "Value", 1:9)
ggplot(meltedYJTransData) +
  geom_histogram(aes(x = Value)) +
  facet_wrap(~Variable)
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](C:\Users\kimmcodxb\Documents\GitHub\APMExercises\notes\Ch3DataPreprocessing_files/figure-markdown_github/unnamed-chunk-13-1.png)

Does not seem to have a big difference Moving on to spatial sign transformation for outliers

``` r
spatSignTrans <- Glass %>% select(-Type) %>% 
                           preProcess(method = c("center", "scale", "spatialSign"))
ssData <- predict(spatSignTrans, Glass[, -10])
glimpse(ssData)
```

    ## Observations: 214
    ## Variables: 9
    ## $ RI <dbl> 0.3862138, -0.1782866, -0.4739625, -0.1932662, -0.2268925, ...
    ## $ Na <dbl> 0.12608198, 0.42318047, 0.09851782, -0.20158533, -0.1230318...
    ## $ Mg <dbl> 0.5551342, 0.4548937, 0.3951817, 0.5799799, 0.4726731, 0.24...
    ## $ Al <dbl> -0.30638163, -0.12188805, 0.12544395, -0.25814737, -0.29911...
    ## $ Si <dbl> -0.49869486, 0.07316353, 0.28831726, -0.04397201, 0.4037353...
    ## $ K  <dbl> -0.297206467, -0.018743853, -0.108111077, 0.093056534, 0.05...
    ## $ Ca <dbl> -0.064496571, -0.567561472, -0.544684692, -0.430850723, -0....
    ## $ Ba <dbl> -0.1561358, -0.2523255, -0.2318677, -0.2929133, -0.2565822,...
    ## $ Fe <dbl> -0.2594843, -0.4193433, -0.3853442, -0.4867968, -0.4264175,...

Exercise 3.2

``` r
data("Soybean")
glimpse(Soybean)
```

    ## Observations: 683
    ## Variables: 36
    ## $ Class           <fct> diaporthe-stem-canker, diaporthe-stem-canker, ...
    ## $ date            <fct> 6, 4, 3, 3, 6, 5, 5, 4, 6, 4, 6, 4, 3, 6, 6, 5...
    ## $ plant.stand     <ord> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ precip          <ord> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 0, 0, 0, 0, 0, 0...
    ## $ temp            <ord> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 2, 2...
    ## $ hail            <fct> 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1...
    ## $ crop.hist       <fct> 1, 2, 1, 1, 2, 3, 2, 1, 3, 2, 1, 1, 1, 3, 1, 3...
    ## $ area.dam        <fct> 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 3, 3, 2, 3, 3, 3...
    ## $ sever           <fct> 1, 2, 2, 2, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1, 1, 1...
    ## $ seed.tmt        <fct> 0, 1, 1, 0, 0, 0, 1, 0, 1, 0, 1, 1, 0, 1, 1, 1...
    ## $ germ            <ord> 0, 1, 2, 1, 2, 1, 0, 2, 1, 2, 0, 1, 0, 0, 1, 2...
    ## $ plant.growth    <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1...
    ## $ leaves          <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1...
    ## $ leaf.halo       <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ leaf.marg       <fct> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2...
    ## $ leaf.size       <ord> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2...
    ## $ leaf.shread     <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ leaf.malf       <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ leaf.mild       <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ stem            <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1...
    ## $ lodging         <fct> 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0...
    ## $ stem.cankers    <fct> 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 0, 0, 0, 0, 0, 0...
    ## $ canker.lesion   <fct> 1, 1, 0, 0, 1, 0, 1, 1, 1, 1, 3, 3, 3, 3, 3, 3...
    ## $ fruiting.bodies <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0...
    ## $ ext.decay       <fct> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0...
    ## $ mycelium        <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ int.discolor    <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 2, 2, 2, 2, 2...
    ## $ sclerotia       <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1...
    ## $ fruit.pods      <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ fruit.spots     <fct> 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4...
    ## $ seed            <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ mold.growth     <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ seed.discolor   <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ seed.size       <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ shriveling      <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    ## $ roots           <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...

Are the missing values concentrated among few predictors? Here is a helper function that computes the missing values percentage by variable

``` r
naPercentage <- function(df) {
  percentage <- floor(colSums(is.na(df))*100/dim(df)[1])
  return(percentage)
}
```

Another alternative is to use dplyr; one of the rare occassions where it is a bit ugly

``` r
Soybean %>% select_if(function(x) any(is.na(x))) %>% 
            summarize_all(funs(sum(is.na(.))*100/length(.)))
```

    ##        date plant.stand  precip     temp     hail crop.hist  area.dam
    ## 1 0.1464129    5.270864 5.56369 4.392387 17.71596  2.342606 0.1464129
    ##      sever seed.tmt     germ plant.growth leaf.halo leaf.marg leaf.size
    ## 1 17.71596 17.71596 16.39824     2.342606  12.29868  12.29868  12.29868
    ##   leaf.shread leaf.malf leaf.mild     stem  lodging stem.cankers
    ## 1    14.64129  12.29868  15.81259 2.342606 17.71596      5.56369
    ##   canker.lesion fruiting.bodies ext.decay mycelium int.discolor sclerotia
    ## 1       5.56369        15.51977   5.56369  5.56369      5.56369   5.56369
    ##   fruit.pods fruit.spots     seed mold.growth seed.discolor seed.size
    ## 1   12.29868    15.51977 13.46999    13.46999      15.51977  13.46999
    ##   shriveling    roots
    ## 1   15.51977 4.538799

We can look at the distribution of NAs by Class. This will help decide if we need to exclude certain variables What is the percentage of missing values for each predictor by class?

``` r
naByPredByClass <- Soybean %>% gather("Variable", "Value", -Class) %>% 
                               group_by(Class, Variable) %>% 
                               filter(is.na(Value)) %>% 
                               summarize_at("Value", funs(sum(is.na(.)))) 
```

    ## Warning: attributes are not identical across measure variables;
    ## they will be dropped

``` r
glimpse(naByPredByClass)
```

    ## Observations: 106
    ## Variables: 3
    ## $ Class    <fct> 2-4-d-injury, 2-4-d-injury, 2-4-d-injury, 2-4-d-injur...
    ## $ Variable <chr> "area.dam", "canker.lesion", "crop.hist", "date", "ex...
    ## $ Value    <int> 1, 16, 16, 1, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,...

There are loads of missing values, scattered among mainly three classes How would one do imputation here?