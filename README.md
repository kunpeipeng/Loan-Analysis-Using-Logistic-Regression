# Loan-Analysis-Using-Logistic-Regression

In this short case analysis, I used logistic regression to predict the likelihood of a loan will default by using independent variables such as borrowers' FICO scores and their delinquency records. 

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

### Split data set and find accuracy of baseline model on test set

```{r}
Loan = read.csv("Loans.csv")
str(Loan)
View(Loan)
# Creating Training and Testing Sets
#install.packages("caTools")
library(caTools)
#Split Loan into train and test sets
set.seed(1234)
split = sample.split(Loan$NotFullyPaid, SplitRatio = 0.70)
Train = subset(Loan, split==TRUE)
Test = subset(Loan, split==FALSE)
#Baseline model accuracy on test set
Baseline_Accuracy <- (nrow(Test)-sum(Test$NotFullyPaid))/nrow(Test)
```
### Build a logit model to predict NotFullyPaid using all other variables. 
```{r}
#Build model with training dataset
Loan.logit = glm(NotFullyPaid ~ ., data = Train, family=binomial)
summary(Loan.logit)
```
From the model outputs above, we can see that the following independent variables are significant: CreditPolicy, Purposecredit_card, Purposedebt_consolidation, Purposesmall_business, Installment, LogAnnualInc, Fico, RevolBal, RevolUtil, InqLast6mths, and PubRec. 

### Application A has a FICO credit socre of 700 while Application B has a FICO score of 710. What's Logit(A)-Logit(B)?
Since all else are the same besides the Fico scores, we can just calculate the difference between Logit(A) and Logit(B) by subtracting their Fico scores and times the result with the Fico coefficient from the logit model. 
```{r}
#Find coefficient for Fico variable from the Logit model 
FICO_Coefficient <- Loan.logit$coefficients[["Fico"]]
#Multiply the difference in Fico score by the Fico coefficient from the model
LogitA.LogitB.diff <- (700-710)*FICO_Coefficient
LogitA.LogitB.diff
```
### Predict the probability of the test set loans not being paid back in full. 
```{r}
PredictedRisk <- predict(Loan.logit, type="response", newdata = Test)
Test$Predicted.Risk = PredictedRisk
library(dplyr)
Test$PredictedRiskInt = ifelse(PredictedRisk < 0.5, 0, 1)
Actual.vs.Pred <- table(Test$NotFullyPaid, Test$PredictedRiskInt)
Actual.vs.Pred
Loan.Pred.Test.Accuracy <- sum(diag(Actual.vs.Pred))/sum(Actual.vs.Pred)
Loan.Pred.Test.Accuracy
```
This model accuracy of 83.78% is very close to the benchmark model accuracy of 83.99%. 

### What's the AUC? 
```{r}
#ROC curve
#install.packages("ROCR")
library(ROCR)
prediction = prediction(Test$Predicted.Risk, Test$NotFullyPaid)
performance(prediction, "auc")@y.values
```
The AUC of the model on the test set is about 0.6674, while the model accuracy on test set is about 83.78%. I think investors can use this as a temporary model for reference while developing more accurate models. 


### Use a logistic regression model to predict NotFullyPaid using only IntRate
```{r}
# uild new logit model using interest rate as the only independent variable
Loan.logit.IR = glm(NotFullyPaid ~ IntRate, data = Train, family=binomial)
summary(Loan.logit.IR)
```
From the model output above, we can see that IntRate is statistically significant when it was the only independent variable in the new logit model. However, it was not significant in the previous model built when other independent variables are included. My hypothesis is that the IntRate is highly correlated with one or more other variables in the previous model built, causing multi-collinearity. 

### Use the interest rate model to predict probability of NotFullyPaid on the test set
```{r}
Prediction.IR <- predict(Loan.logit.IR, type="response", newdata = Test)
Test$Prediction.IR = Prediction.IR
#get the highest predicted probability of a loan not being paid back in full on the test set 
max(Prediction.IR)
```
If we use 0.5 as a threshold to determine if a customer will pay back in full or not, none of the customers on the testing set will have a probability higher than 0.5.

```{r}
#calculate AUC
library(ROCR)
prediction.IR.vs.Actual = prediction(Prediction.IR, Test$NotFullyPaid)
performance(prediction.IR.vs.Actual, "auc")@y.values
```
The the current model is more robust than the previous model because the current model only has 1 independent variable, while the previous model has all other independent variables as input. The contrast between 0.6186 of AUC is so much worse than the 0.6674 of AUC from the more complicated model. 

### Using our logistic regression model to identify loans that are expected to be profitable: 
### Scenario 1: assume that we have a $10 investment with an annual interest rate of 6% pay back after 3 years, using continuous compounding of interest: 
$$10*e^{0.06*3} = 11.97217$$
Based on the calculation above, the expected returns (principle + interest) after 3 years at 6% payback would be $11.97-$10 = $1.97. 

Note: If the investment is not paid back in full, the investor will have a negative balance on their returns because the investor will have no profit. 


### Computing the profit of a $1 investment in each loan
```{r}
#Create new column for profit, assuming C = $1
Test$profit = 1*exp(Test$IntRate*3) - 1
Test$profit[Test$NotFullyPaid == 1] = -1
# max(test$profit)
MaxProfitC1 <- max(Test$profit)
MaxProfitC1
```
The max profit amount is $0.889. 

### Alternative Investment Strategy
```{r}
#create new data set for loans with high interest rates
HighInterest  <- subset(Test, Test$IntRate  >= 0.15)
#Mean profit for high interest rate loan 
mean(HighInterest$profit)
#Proportion of interest not paid back in full
sum(HighInterest$NotFullyPaid == 1)/nrow(HighInterest)
```
###  What is the profit of an investor who invested $1 in each of these 100 lonas? How many of the 100 selected loans were not paid in full? How does this compare to the simple strategy?
```{r}
# sort by predicted risk
HighInterest <- HighInterest[order(HighInterest$PredictedRisk),]
#Creating the SelectedLoan dataset
Threshold = sort(HighInterest$Predicted.Risk, decreasing=FALSE)[100]
SelectedLoans = subset(HighInterest, Predicted.Risk <= Threshold) 
dim(SelectedLoans)
#Total profit for an investor who invested $1 in each of the 100 loans
Total.Profit.for.investors <- sum(SelectedLoans$profit)
Total.Profit.for.investors
#Number of loans not paid back in full
table(SelectedLoans$NotFullyPaid)
```
As the above analyses shown, the profit of an investor who invested $1 in each of these loans would earn an expected profit of $31.24. Out of the 100 loans, 19 of them will not be paid in full. This simple strategy, yielding $31.24 profit seems perform better than investing in all loans, yielding $20.94 for $100 investment. 

### Note: Be wary of the assumptions behind this analysis: 
The important assumptions of predictive modeling is that past is indicative of the future. In the context of financial situations, credit risk might fluctuate based on a person's living quality and job stability. If a person's life is heavily impacted by sudden change of events in factors such as job security, they might now be able to maintain a good credit score like they did in the past. To hedge this risk, an analyst can do further research in gathering more data or even survey a few of the customers who took out the loans to get a better understanding of their personal financial visions and plans. 

