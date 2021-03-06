Practial Machine Learning Project - Data Science Track
========================================================

## Project Background
Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement.  One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. 

The goal of this project  will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here[1]. The section on weight lifting on the linked site[1] provides the background for this project.


## Project Approach
-The training data set was downloaded and inspected for data quality. 
-A usable tidy dataset was created from the raw data by which involved discarding some  variables.  
-Tidy data set was  partitioned into training,cross validation, and test sets.
-Random Forest was selected as the algorithm for training given input from course instructors on its real-world efficacy.
-Crossvalidation data set was used to determine if principle componets of tidy data sets have better predictive value compared to just the rawinputs.  
-The test data set (not the final one but subset from training data) was used to fine tune the random forest model
-The fine-tuned model was used to predict outcome class from  the final (& seperate) test data test 

## Reproducible Code for model building and tuning

Random Forest model is built and tuned as follows:


```r

# ------------------------------------------------------------------------------
# STEP 1 - Load training data from link provided and determine size of data
# ------------------------------------------------------------------------------

datalink <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"

download.file(datalink, dest = "./pml-training.csv", method = "curl", extra = "-k")

rawdf <- read.csv("pml-training.csv")

# From offline analysis, it was observed that the variable new_window has a
# lot of bearing on the kind of data that the other fields have.

numobs <- dim(rawdf)[1]  # number of rows in data set
numvars <- dim(rawdf)[2]  # number of columns  in data set

# ------------------------------------------------------------------------------
# STEP 2 - Exploration high level data quality & refine as needed
# ------------------------------------------------------------------------------

library(Hmisc)
library(pastecs)


# get % of NA values by col since data is not uniformly clean
percentnasbycol <- colSums(is.na(rawdf))/numobs

# get distribution of errors for visual inspection
hist(percentnasbycol)
```

![plot of chunk analysis](figure/analysis.png) 

```r

# It is clear that we two groups of fields - one group with mostly missing
# values and another group with values for all observations. We can then
# focus only on the fields that have good values for further analysis

naerrCol <- which(percentnasbycol > 0.2)  #  19k+ rows are NA for these fields.
tmpData <- rawdf[, -naerrCol]

# In inspecting this tmpData object, it is clear that we still have
# additional fields that are not of good quality. We first get rid of the
# rows that have new_window ='yes' because the corresponding records have
# different behaviour compared to vast majority of records

idx <- which(tmpData$new_window == "yes")
tmpData <- tmpData[-idx, ]


colclasses <- as.character(lapply(tmpData, class))  # list of col classes

factorcols <- which(colclasses == "factor")
names(tmpData[, factorcols])
```

```
##  [1] "user_name"               "cvtd_timestamp"         
##  [3] "new_window"              "kurtosis_roll_belt"     
##  [5] "kurtosis_picth_belt"     "kurtosis_yaw_belt"      
##  [7] "skewness_roll_belt"      "skewness_roll_belt.1"   
##  [9] "skewness_yaw_belt"       "max_yaw_belt"           
## [11] "min_yaw_belt"            "amplitude_yaw_belt"     
## [13] "kurtosis_roll_arm"       "kurtosis_picth_arm"     
## [15] "kurtosis_yaw_arm"        "skewness_roll_arm"      
## [17] "skewness_pitch_arm"      "skewness_yaw_arm"       
## [19] "kurtosis_roll_dumbbell"  "kurtosis_picth_dumbbell"
## [21] "kurtosis_yaw_dumbbell"   "skewness_roll_dumbbell" 
## [23] "skewness_pitch_dumbbell" "skewness_yaw_dumbbell"  
## [25] "max_yaw_dumbbell"        "min_yaw_dumbbell"       
## [27] "amplitude_yaw_dumbbell"  "kurtosis_roll_forearm"  
## [29] "kurtosis_picth_forearm"  "kurtosis_yaw_forearm"   
## [31] "skewness_roll_forearm"   "skewness_pitch_forearm" 
## [33] "skewness_yaw_forearm"    "max_yaw_forearm"        
## [35] "min_yaw_forearm"         "amplitude_yaw_forearm"  
## [37] "classe"
```

```r

# On inspection, through summary or stat.desc, the factor variables can all
# be discarded. This includes 'user_name','cvtd_timestamp','new_window' and
# 'classe' variables. The rest are sensor telemetry fields that dont have
# valid values for 19k+ records

integercols <- which(colclasses == "integer")
names(tmpData[, integercols])
```

```
##  [1] "X"                    "raw_timestamp_part_1" "raw_timestamp_part_2"
##  [4] "num_window"           "total_accel_belt"     "accel_belt_x"        
##  [7] "accel_belt_y"         "accel_belt_z"         "magnet_belt_x"       
## [10] "magnet_belt_y"        "magnet_belt_z"        "total_accel_arm"     
## [13] "accel_arm_x"          "accel_arm_y"          "accel_arm_z"         
## [16] "magnet_arm_x"         "magnet_arm_y"         "magnet_arm_z"        
## [19] "total_accel_dumbbell" "accel_dumbbell_x"     "accel_dumbbell_y"    
## [22] "accel_dumbbell_z"     "magnet_dumbbell_x"    "magnet_dumbbell_y"   
## [25] "total_accel_forearm"  "accel_forearm_x"      "accel_forearm_y"     
## [28] "accel_forearm_z"      "magnet_forearm_x"
```

```r

# Amongst the integer cols, we can omit X, rawtimestamp1, rawtimestap2 and
# numwindow but retail the rest since they look like telemetry from the
# sensors
discardintegercols <- c(1, 3, 4, 7)

unneededcols <- union(discardintegercols, factorcols)

alltrndata <- tmpData[, -unneededcols]
alltrndata$classe <- tmpData$classe

# ------------------------------------------------------------------------------
# STEP 3 - Perform feature selection for training using cross validation
# ------------------------------------------------------------------------------

# The course instructors mentioned that Random Forest is amongst the best
# algorithms that exist for prediction. So we are going to test how random
# forest behaves with just raw predictors,and with principal components that
# cover 80% & 90% of raw predictor variance

# we use the data as follows: 60% Training 20% cross Validation 20% Testing

library(caret)

### 3 A - Partition Data Set
inTrainCV <- createDataPartition(y = alltrndata$classe, p = 0.8, list = FALSE)
traincv <- alltrndata[inTrainCV, ]

testing <- alltrndata[-inTrainCV, ]  # Testing data set is seperated out.(mocktest)

inTrain <- createDataPartition(y = traincv$classe, p = 0.75, list = FALSE)

training <- traincv[inTrain, ]  # Training data set seperated out 
crossvalidation <- traincv[-inTrain, ]  # Cross Validation  data set seperated out 

### 3 B - Identify number of relevant principle components covering 80 & 90%
### of predictor variance and get the transformed data sets

svdobj <- svd(scale(training[, -53]))  # singular value decomposition
dsquared <- svdobj$d^2  # square of d matrix by element
cumdsquared <- cumsum(dsquared)  # cum sum of dsquared
varindex <- cumdsquared/sum(dsquared)  # proportion of variance covered


# number of principal components that capture 80% of variance
prComp_0.8 <- min(which(varindex >= 0.8))
# preprocess dataset
preobj_0.8 <- preProcess(scale(training[, -53]), method = "pca", pcaComp = prComp_0.8)
# get transformed predictors back
training_0.8 <- predict(preobj_0.8, scale(training[, -53]))
# convert to dataframe and add outcome variable
training_0.8 <- as.data.frame(training_0.8)
training_0.8$classe <- training$classe

# number of principal components that capture 90% of variance
prComp_0.9 <- min(which(varindex >= 0.9))
# preprocess dataset
preobj_0.9 <- preProcess(scale(training[, -53]), method = "pca", pcaComp = prComp_0.9)

# get transformed predictors back
training_0.9 <- predict(preobj_0.9, scale(training[, -53]))
# convert to dataframe and add outcome variable
training_0.9 <- as.data.frame(training_0.9)
training_0.9$classe <- training$classe



### 3 C - Train 3 models with each of the predictor sets

modelfitrfraw <- train(classe ~ ., data = training, method = "rf", ntree = 20, 
    preProcess = c("center", "scale"))

modelfitrfpr_0.8 <- train(classe ~ ., data = training_0.8, method = "rf", ntree = 20)
modelfitrfpr_0.9 <- train(classe ~ ., data = training_0.9, method = "rf", ntree = 20)

### 3 D - evaluate performance on cross validation data set

# Predict with raw inputs
predictionsrawinputs <- predict(modelfitrfraw, newdata = crossvalidation)

# Predict with 80% principle components
crossval_0.8 <- predict(preobj_0.8, scale(crossvalidation[, -53]))
crossval_0.8 <- as.data.frame(crossval_0.8)
crossval_0.8$classe <- crossvalidation$classe
predictionspr_0.8 <- predict(modelfitrfpr_0.8, newdata = crossval_0.8)

# Predict with 90% principle components
crossval_0.9 <- predict(preobj_0.9, scale(crossvalidation[, -53]))
crossval_0.9 <- as.data.frame(crossval_0.9)
crossval_0.9$classe <- crossvalidation$classe
predictionspr_0.9 <- predict(modelfitrfpr_0.9, newdata = crossval_0.9)

rawinputresults <- confusionMatrix(data = predictionsrawinputs, crossvalidation$classe)
prresults_0.8 <- confusionMatrix(data = predictionspr_0.8, crossval_0.8$classe)
prresults_0.9 <- confusionMatrix(data = predictionspr_0.9, crossval_0.9$classe)

# Get Accuracy for each model in crossvalidation data set
rawinputresults[[3]][1]
```

```
## Accuracy 
##   0.9909
```

```r
prresults_0.8[[3]][1]
```

```
## Accuracy 
##   0.9347
```

```r
prresults_0.9[[3]][1]
```

```
## Accuracy 
##   0.9487
```

```r

## It is clear that just the raw inputs serve as best version of predictors
## when we use random forest algorithm

# ------------------------------------------------------------------------------
# STEP 4 - Fine Tune model by evaluating performance on the (mock)test set
# ------------------------------------------------------------------------------

## A note on tuning parameter range selection: The larger the number of trees
## -i.e. ntree, the better fit we will get but it also is computationally
## more expensive.  There is also a potential to overfit here as well.  The
## order of magnitude ( i.e ~ 10) was arrived at through trial and error and
## some online research. Larger values were tried but the algorithm took
## longer than the author was willing to wait


rf_10 <- train(classe ~ ., data = training, method = "rf", ntree = 10, preProcess = c("center", 
    "scale"))
rf_20 <- train(classe ~ ., data = training, method = "rf", ntree = 20, preProcess = c("center", 
    "scale"))
rf_30 <- train(classe ~ ., data = training, method = "rf", ntree = 30, preProcess = c("center", 
    "scale"))


predictions_10 <- predict(rf_10, newdata = testing[, -53])
predictions_20 <- predict(rf_20, newdata = testing[, -53])
predictions_30 <- predict(rf_30, newdata = testing[, -53])


result_10 <- confusionMatrix(data = predictions_10, testing$classe)
result_20 <- confusionMatrix(data = predictions_20, testing$classe)
result_30 <- confusionMatrix(data = predictions_30, testing$classe)

# Get Accuracy for each model in testing data set
result_10[[3]][1]
```

```
## Accuracy 
##   0.9862
```

```r
result_20[[3]][1]
```

```
## Accuracy 
##   0.9909
```

```r
result_30[[3]][1]
```

```
## Accuracy 
##   0.9906
```

```r

## The models have performed similarly above.  Since ntree=20 has worked well
## in both cross validation and testing data sets, this is what we use in the
## final & seperate testing set.
```


## Estimating Out of Sample Error 

Out of sample error is estimated to be in the order of  1 %, which is to say that the model seems to generalize  well at least within the above  dataset.   
 
Error estimate is based on crossvalidation & tuning results

## Reproducible code for Final (Test) value Prediction

Test  file was downloaded and transformed in the same manner employed for the  training set. The selected random forest model (rf_20 with 20 trees) was applied to the transformed test set and the outcome variable - classe, was predicted. The predictions were written into seperate files (one each for the 20 test cases).



```r
# ------------------------------------------------------------------------------
# STEP 1 - Download final test data and predict final values
# ------------------------------------------------------------------------------

testfilelink <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"

download.file(testfilelink, dest = "./pml-testing.csv", method = "curl", extra = "-k")

df <- read.csv("pml-testing.csv")

df <- df[order(df$problem_id), ]

firstclean <- df[, -naerrCol]
ft <- firstclean[, -unneededcols]

# ft<-as.data.frame(scale(ft)) all the new_window values are no

answers <- as.character(predict(modelfitrfraw, newdata = ft))

# ------------------------------------------------------------------------------
# STEP 2 - write out final files
# ------------------------------------------------------------------------------

pml_write_files = function(x) {
    n = length(x)
    for (i in 1:n) {
        filename = paste0("problem_id_", i, ".txt")
        write.table(x[i], file = filename, quote = FALSE, row.names = FALSE, 
            col.names = FALSE)
    }
}

pml_write_files(answers)
```




[1]:http://groupware.les.inf.puc-rio.br/har
