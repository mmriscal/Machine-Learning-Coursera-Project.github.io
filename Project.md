---
title: "Machine Learning Project"
author: "Manuel Mariscal"
date: "25/2/2020"
output:
  pdf_document: default
  html_document: default
        keep_md: yes
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, cache = TRUE, message=FALSE, warning=FALSE)
```

## SUMMARY

I use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways (A,B,C,D,E). More information is available from the website here: http://web.archive.org/web/20161224072740/http:/groupware.les.inf.puc-rio.br/har (see the section on the Weight Lifting Exercise Dataset).

The goal is to predict the manner in which different people did the exercise. This is the "classe" variable in the training set. 
In this report, I describe how I choose the best variables to train the model, I built a model using cross validation, and I calculate the accuracy of the prediction. See conclusions to check the results. 

The training data for this project are available here:

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv

The test data are available here:

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv


## LOAD AND PRE-PROCESS DATA

I will take only the columns with some information in the test set. Therefore, I will delete the variables in which all the 
observations are NAs. I am going to train the model using only those predictors. 

```{r load_data, message=FALSE, warning=FALSE}
set.seed(222)
library(caret); library(kernlab); library(randomForest); library(dplyr)

setwd("C:/Users/mmariscal/Documents/3. Personal/JHU/8. Practical Machine Learning/Project/")
train <- read.csv("pml-training.csv") 
test <- read.csv("pml-testing.csv") 

index <- colSums(is.na(test)) == nrow(test)
train <- train[!index]
test <- test[!index]
# I am going to delete also the first columns with time and names information because 
# in my opinion that could mislead the model
train <- train[-(1:7)]
test <- test[-(1:7)]
test <- select(test, -problem_id)
```

## FEATURE SELECTION

I am going to use different methods and mixing them. I will use *t_feat* as a small data sample only to 
study the best features and in order not to require a lot of computing time.

To train these models and also for the final one, I am using a k=5 fold cross validation method.

```{r features, message=FALSE, warning=FALSE}
inTrain <- createDataPartition(y=train$classe, p=0.8, list=FALSE)
t_feat <- train[-inTrain,]

# set parallel processing will speed up the computing 
library(parallel)
library(doParallel)
cluster <- makeCluster(detectCores() - 1) # convention to leave 1 core for OS
registerDoParallel(cluster)

# Define train control for k fold cross validation and parallel processing
fitControl <- trainControl(method = "cv", number = 5, allowParallel = TRUE)

# RFE Method
subsets <- c(15, 20, 30) #the number of most important features the rfe should iterate
ctrl <- rfeControl(functions = rfFuncs, # for ramdon forest
                   method = "repeatedcv", # k-fold cross validation
                   repeats = 5, # I set K = 5
                   verbose = FALSE)

mod_rfe <- rfe(classe ~., data = t_feat, sizes = subsets, 
               rfeControl = ctrl, trControl = fitControl)

names_rfe <- mod_rfe$optVariables

# varImp function
mod_rf <- train(classe ~., data = t_feat, method = "rf", trControl = fitControl)
imp <- varImp(mod_rf)
impDF<-data.frame(imp[1])
names_rf <- rownames(impDF)[order(impDF$Overall, decreasing=TRUE)[1:40]]

# Boruta
library(Boruta)
mod_bor <- Boruta(classe ~., data = t_feat, doTrace = 2, maxRuns = 300)
plot(mod_bor, las = 2, cex.axis = 0.7)

names_bor <- getSelectedAttributes(mod_bor)

# And finally, I choose the features that are common to the three of them.
features <- intersect(names_rf, intersect(names_bor, names_rfe))
features
```

These features are the most important ones for training the model according to RFE, varIMP and Boruta methods. We can see also a picture representing the importance of the variables according to Boruta.


## MODEL SELECTION

I will use random forest with the list of features selected in the previous code. I am going to divide the data (because it is
big enough) into a new training and testing data sets. That way, I will be able to test my model before predicting on the final test.

```{r model_sel, message=FALSE, warning=FALSE}
feat <- features[1]
for (i in 2:length(features))
{
        feat <- paste(feat, "+", features[i])
}
form <- as.formula(paste('classe ~', feat))

inTrain <- createDataPartition(y=train$classe, p=0.7, list=FALSE)
testing <- train[-inTrain,]
training <- train[inTrain,]

model <- train(form, data =training, method="rf", ntree=200, trControl = fitControl)

```

## TEST VALIDATION

```{r validation, message=FALSE, warning=FALSE}
pred <- predict(model, newdata = testing)

confusionMatrix(pred, testing$classe)
```

Because accuracy is good, I can train the final model with the complete data and predict the test set.

## FINAL MODEL

```{r final, message=FALSE, warning=FALSE}
final_model <- train(form, data =train, method="rf", ntree=200, trControl = fitControl)

final_pred <- predict(model, newdata = test)
```

# Conclusion

The model has been trained using k-fold cross validation. The features were selected using Boruta, VarIMP and RFE methods: 
```{r error, message=FALSE, warning=FALSE}
features

1 - confusionMatrix(pred, testing$classe)$overall[1]

final_pred
```

The accuracy of the model is bigger than 99.4% and has an out of sample error of only 0.0057 %. Probably, it would be neccessary to adjust better the prediction for class D. 

The final prediction for the test set is also shown.
