

## Week 2

### Preprossing data with caret
```{r}
library(caret)
library(kernlab)
data(spam)
inTrain<-createDataPartition(y=spam$type,p=0.75,list=FALSE)
training<-spam[inTrain,]
testing<-spam[-inTrain,]
hist(training$capitalAve,main="",xlab="avg. capital run length")
```

The histogram shows that the data are heavily skewed to the left. 

#### Standardizing the variables (so that they have `mean = 0` and `sd=1`)
```{r}
trainCapAve<-training$capitalAve
trainCapAveS<-(trainCapAve-mean(trainCapAve))/sd(trainCapAve)
mean(trainCapAveS)
sd(trainCapAveS)
```

#### Standardizing the test set, using mean and sd of the training set. This means that the standardized test cap will not be exactly the same as that of the training set, but they should be similar. 
```{r}
testCapAve<-testing$capitalAve
testCapAveS<-(testCapAve-mean(trainCapAve))/sd(trainCapAve)
mean(testCapAveS)
```

#### Use preprocess() function to do the standardization on the training set. The result is the same as using the above functions
```{r}
preObj<-preProcess(training[,-58],method=c("center","scale"))
trainCapAveS<-predict(preObj,training[,-58])$capitalAve
mean(trainCapAveS)
sd(trainCapAveS)
```
#### Use `preProcess()` to do the same on the testing dataset. Note that `preObj` (which was created based on the training set) is also used to predict on the testing set.

Note that `mean()` is not equal to 0 on the testing set, and `sd` is not equal to 1.

```{r}
testCapAveS<-predict(preObj,testing[,-58])$capitalAve
mean(testCapAveS)
sd(testCapAveS)
```

#### Use `preProcess()` directly when building a model

```{r}
set.seed(1)
model<-train(type ~.,data=training,preProcess=c("center","scale"),method="glm")
model
```

#### Standardising - Box-Cox Transforms

This transforms the data into normal shape - i.e. bell shape
```{r}
preObj<-preProcess(training[,-58],method=c("BoxCox"))
trainCapAveS<-predict(preObj,training[,-58])$capitalAve
par(mfrow=c(1,2))
hist(trainCapAveS)
qqnorm(trainCapAveS)
```

#### Standardization: Imputing data where it is NA using `knnImpute`

`knnImpute` uses the average of the k-nearest neighbours to impute the data where it's not available. 

```{r}
set.seed(1)

# Make some value NAs
training$capAve<-training$capitalAve
selectNA<-rbinom(dim(training)[1],size=1,prob=0.05)==1
training$capAve[selectNA]<-NA

# Impute data when it's NA, and standardize
preObj<-preProcess(training[,-58],method="knnImpute")
capAve<-predict(preObj,training[,-58])$capAve

# Standardize true values
capAveTruth<-training$capitalAve
capAveTruth<-(capAveTruth-mean(capAveTruth))/sd(capAveTruth)
```

Look at the difference at the imputed value (`capAve`) and the true value (`capAveTruth`), using `quantile()` function.

If the values are all relatively small, then it shows that imputing data works (i.e. doesn't change the dataset too much).
```{r}
quantile(capAve-capAveTruth)
```

#### Some notes on preprocessing data

* training and testing must be processed in the same way (i.e. use the same `preObj` in `predict()` function)


#### Covariate/Predictor/Feature Creation

1. Step 1: raw data -> features (e.g. free text -> data frame)
   Google "Feature extraction for [data type]"
   Examples:
   * Text files: frequency of words, frequency of phrases, frequency of capital letters
   * Images: Edges, corners, ridges
   * Webpages: # and type of images, position of elements, colors, videos (e.g. A/B testing)
   * People: Height, weight, hair color, gender etc.
   
2. Step 2: features -> new, useful features
   * more useful for some models (e.g. regression, SVM) than others( e.g. decision trees)
   * should be done **only on the training set**
   * new features should be added to data frames

3. An example of feature creation
        ```{r}
        library(ISLR)
        library(caret)
        data(Wage)
        inTrain<-createDataPartition(y=Wage$wage,p=0.7,list=FALSE)
        training<-Wage[inTrain,]
        testing<-Wage[-inTrain,]
        ```
  * Convert factor variables to dummy variables 
    
    The `jobclass` column is chacracters, so we can convert it to dummy variable with `dummyVars` function
    
    ```{r}
    dummies<-dummyVars(wage ~ jobclass,data=training)
    head(predict(dummies,newdata=training))
    ```
    
   * Remove features which is the same throughout the dataframe, using `nearZeroVar`
    
    If nsv (`nearZeroVar`) returns TRUE, then this feature is not important and thus can be removed. 
    
    ```{r}
    nsv<-nearZeroVar(training,saveMetrics = TRUE)
    nsv
    ```
    * Spline basis
    `df=3` says that we want a 3rd-degree polynomial on this variable `training$age`.
    First column means `age`
    Second column means `age^2`
    Third column means `age^3`
    ```{r}
    library(splines)
    bsBasis<-bs(training$age,df=3)
    bsBasis
    ```
    #### Fitting curves with splines
    ```{r}
    lm1<-lm(wage~bsBasis,data=training)
    plot(training$age,training$wage,pch=19,cex=0.5)
    points(training$age,predict(lm1,newdata=training),col="red",pch=19,cex=0.5)
    ```
    #### splines on the test set.
    Note that we are using the same `bsBasis` as is created in the training dataset
    ```{r}
    predict(bsBasis,age=testing$age)
    ```

### PCA (Principal Components Analysis)
1. Find features which are correlated

`which()` returns the list of features with correlation > 0.8
```{r}
library(caret)
library(kernlab)
data(spam)
set.seed(1)
inTrain<-createDataPartition(y=spam$type,p=0.75,list=FALSE)
training<-spam[inTrain,]
testing<-spam[-inTrain,]

M<-abs(cor(training[,-58]))
diag(M)<-0
which(M>0.8,arr.ind=T)
```
  
  Take a look at the correlated features:
  
```{r}
  names(spam)[c(34,32,40)]
  plot(spam[,34],spam[,32])
```

  Apply PCA in R: `prcomp()`
```{r}
smallSpam<-spam[,c(34,32)]
prComp<-prcomp(smallSpam)
plot(prComp$x[,1],prComp$x[,2])
prComp$rotation
```
 
 #### PCA on spam data
```{r}
typeColor<-((spam$type=="spam")*1+1)
prComp<-prcomp(log10(spam[,-58]+1))
plot(prComp$x[,1],prComp$x[,2],col=typeColor,xlab="PC1",ylab="PC2")
```

  #### PCA with caret, preProcess()
```{r}
preProc<-preProcess(log10(spam[,-58]+1),method="pca",pcaComp = 2)
spamPC<-predict(preProc,log10(spam[,-58]+1))
plot(spamPC[,1],spamPC[,2],col=typeColor)
```