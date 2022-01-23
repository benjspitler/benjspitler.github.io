## Using lasso regressions and multilayer neural networks to predict NBA player statistics and fantasy performance

<img src="images/jokic.png?raw=true/">

This portfolio is largely intended to showcase the power of data science and analytics to drive human rights investigations. That said, those who know me will know that my one of my biggest passions (obsessions?) is NBA basketball, and I've recently been enjoying using machine learning methods to play around with NBA statistics. This is a tidy example of how methods like lasso regression and neural networks can serve as powerful predictive models when fed large, robust datasets, so I wanted to post my results.

In this project, I took 30 years of historical NBA player data and built a series of machine learning models to predict how all non-rookie players would perform in the 2021-2022 NBA season, which was ongoing at the time of the writing of this post. I then fed the results of those predictive models into the alogrithms that top fantasy basketball leagues hosted by ESPN and Yahoo use to allocate player fantasy points. The result of this was a list of the predicted top performers in fantasy basketball for the 2021-2022 season.

### The raw data

The data for this project was scraped from [Basketball Reference](https://www.basketball-reference.com/), a leading online repository of NBA statistics. Compiling 30 years of data into one dataframe was a large task that involved quite a bit of code that I will not include here, but the end result is a data set that looks like the below, which I have already scaled: 

<img src="images/nba_data_screenshot.png?raw=true"/>

Each row of this dataframe corresponds to an an individual "player season." For example, there is one row for Lebron James's 2011-2012 season, a separate row for his 2010-2011 season, and so on, for every player that has played in the NBA since 1990. The columns in the data set represent a variety of different demographic and statistical categories. These statistical categories include both what are called basic "counting stats" (things like points, rebounds, assists, etc.), as well as "advanced stats", which are statistical categories invented by both amateur and professional NBA analysts which amalgamate counting stats into purportedly more descriptive new statistical parameters. Advanced stats include things like true shooting percentage, three-point attempt rate, and assist percentage, for example.

There are two important things to note about this data. The first relates to the response variables I am attempting to predict. For each player in the 2021-2022 season, these models aim to predict:

- Total points
- Total 3-pointers made
- Total field goals attempted
- Total field goals made
- Total free throws attempted
- Total free throws made
- Total rebounds
- Total assists
- Total steals
- Total blocks

Accordingly, these statistics are the final ten categories of the dataframe. However, it should be noted that in each row, the response columns correspond to **one season later** than the rest of the columns in that row. For example, for the row corresponding to Kevin Durant's 2015-2016 season, the last ten columns of that row (total points, total 3-pointers made, total field goals attempted, etc.) correspond to Kevin Durant's 2016-2017 season. The data must be set up this way in order to predict future performance.

The second note about this data is that I have included a column called 't'. This column is a continuous numerical variable that takes a larger value based on the recency of the value in the "season column". So a row corresponding to the most recent year represented in the data has 30 in the 't' column, a row corresponding to the second-most recent year represented in the data has 29 in the 't' column, and so on. Doing this is an easy (if relatively imprecise) way to weight data so that more recent data is given more predictive weight by machine learning algorithms. There are more sophisticated ways to do this (exponential smoothing or recurrent neural networks, for example), but this is the method I chose to implement within the context of the lasso.

### Lasso regression

The lasso is a handy tool to use in a case like this where we have many predictor variables that may be significant, and no clear way to choose which to include in our model. Broadly, the lasso does this for us by imposing a penalty term on the regression that shrinks to zero the coefficients of variables that the model deems to be less important. Below, I outline my approach to predicting total points for each player; I repeated this process ten times, building one model for each statistical category I wanted to predict.

The first step was to split the data set into training and testing data. I chose to use the data from the 2015/2016 - 2018/2019 seasons as the training data and the data from the 2019/2020 season as the testing data. This approach was chosen such that the 2019/2020 comprises 30% of the total data, and then I selected enough seasons for the training data such that they cumulatively comprised 70% of the total. Note that I have read in the scaled data from a .csv as a dataframe called "nba_scaled_names":

```javascript
library(tidyverse)
library(glmnet)

nba_train_w_names = subset(nba_scaled_names, t > 24 & t < 29)
nba_train <-  select(nba_train_w_names, -c(name, season))

nba_test_w_names = subset(nba_scaled_names, t == 29)
nba_test <-  select(nba_test_w_names, -c(name, season))
```
Next, I built x_train, y_train, x_test, and y_test objects for use with the lasso model. The glmnet() function, which I use below, requires matrices, not data frames. The model.matrix() function is helpful for this, as it creates a matrix corresponding to the predictors and also automatically transforms any qualitative variables into dummy variables. The latter property is important because glmnet() can also only take numerical, quantitative inputs:
```javascript
x_train <- model.matrix(tot_pts ~ ., nba_train)[, -1]
y_train <- nba_train$tot_pts
x_test <- model.matrix(tot_pts ~. , nba_test)[, -1]
y_test <- nba_test$tot_pts
```
The next step was to use a grid-search cross-validation process to find the optimal value for lambda, which is the parameter that the lasso will use to penalize the regression and shrink coefficients. By default the glmnet() function, which I use below, performs lasso regression for an automatically selected range of lambda values. However, here I chose to implement the function over a grid of values essentially covering the full range of scenarios from the null model containing only the intercept, to the least squares fit:
```javascript
grid <- 10^seq(10, -2, length = 100)

cv_out <- cv.glmnet(x_train, y_train, alpha = 1, lambda = grid)
best_lam <- cv_out$lambda.min
```
Next, I built the lasso model using glmnet():
```javascript
lasso_reg_nba <- glmnet(x_train, y_train, alpha = 0, family = "gaussian", lambda = best_lam)
```
From there, I used the model I fit on the training data to make predictions on the test data, and calculated the test mean squared error and test R-squared:
```javascript
lasso_pred_nba <- predict(lasso_reg_nba, s = best_lam, newx = x_test)
test_MSE <- mean((lasso_pred_nba - y_test)^2)
SSE <- sum((lasso_pred_nba - y_test)^2)
SST <- sum((y_test - mean(y_test))^2)
R_square <- 1 - SSE / SST
```
This results in a test R-squared of 0.856.
The final step is to use the lasso model to make predictions based on data from the most recent NBA season, which is the 2019-2020 season. These are the rows for which the value in the t column in the data set == 30. I scaled this data and stored it in a dataframe called t30_no_names. First, I created a matrix object for use with the lasso regression, and then I made predictions on that data:
```javascript
x_t30 <- model.matrix(tot_pts ~ ., t30_scaled_no_names)[, -1]
predict(lasso_reg_nba_full, s = best_lam, newx = x_t30)
```
When sorted in descending order, my lasso model predicted the following top 12 scorers in the NBA in the 2021-2022 season:

<img src="images/lasso_pts_leaders_all.png?raw=true" width = "300" height = "300">

These results actually make a good deal of sense, though there are some glaring omissions and questionable inclusions, which I discuss later in this post.


The first step was to pare down the raw data that forms the back end of the EDPI ([downloadable here](https://egyptdeathpenaltyindex.com/download-data)) into a format that could serve as the basis for regression analysis. The EDPI data contains a wealth of information about individual defendants in capital trials in Egypt, including:

- Demographic information about defendants, such as age, gender, birthplace, occupation, etc.
- Situational information about alleged offences/crimes, such as time/place of alleged occurrence, type of offence, etc.
- Procedural information about trials, such as the verdicts reached at different appeal phases, time periods in which judgements were handed down, presence/absence of defendants, whether a defendant was ultimately executed, etc.

That data is rich and useful, but messy for the purposes of regression. It looks like this:

<img src="images/EDPI_raw_screenshot.png?raw=true"/>

Some of these pieces of metadata are better suited than others to serving as predictors. For example, information regarding when a final verdict was reached in a case allows us to determine which years in the past produced the most capital trials that led to executions, but that information is unlikely to help us predict the outcome of future cases. To prepare the data for use in a logistic regression process, I identified the variables most relevant for these predictive purposes, and settled on the following:

- Whether or not an individual was ultimately executed (column name **executed_0_1** in the new, pared down dataset). This would be the response variable. Any individual who was ultimately executed received a 1 in this column, anyone who was not executed received a 0.
- Whether the offence was "political" or "criminal" in nature--that is, whether  the alleged facts of the case and the perceived motivation for the commission of the offence were in some way connected to the political and societal changes that have arisen in Egypt since the January 2011 revolution or not (column name **category_of_offence**).
- The gender of the defendant (column name **defendant_gender**).
- Whether the defendant was tried in a civilian court or a military tribunal (column name **court_type**).
- Whether the defendant was tried _in absentia_ or was present during proceedings (column name **defendant_status**).
- The amount of time elapsed between the defendant's alleged offence and the court reaching a verdict in that defendant's case in the first instance (column name **days_btwn_offence_and_crim_1_judgement**).
- The number of overall death sentences handed down in the governorate where the individual in question was being tried (column name **governorate_sentences**).
- The specific offence that a defendant was accused of (column name **offence**).

### Dummies or factors?

One of the mistakes I made during my first attempt at this analysis was to use R's fastDummies package to create dummy variables for each category. This meant that every factor value within the text columns received its own new column, wherein each row received a 1 or a 0. For example, the "offence" column initially contained a number of different death-eligible offenses that defendants were charged with--espionage, terrorist acts, murder, etc. When I used the fastDummies package, this created a new column for every possible offence (titled offence_espionage, offence_terrorist_acts, offence_murder, etc.), and each row received a 1 in the column pertaining to that defendant's alleged offence, and a 0 everywhere else. The problem there is that this is not a statistically sound way to approach logistic regression. Specifically, wherever there are factor variables, R's glm function will remove one "level" from each factor as a reference. Then, in the resulting logistic regression output, the coefficients for the other factor variables within that column express the extent to which those variables influenced the chosen response variable *compared to the reference level*." As such, including every level of a factor in a logistic regression model, like I initially tried to do with the fastDummies package, is incorrect, as it leaves no reference point for interpreting the other variables.

THe correct way to do this is actually to feed the factor variables to R's glm function using the as.factor kwarg. Before doing that, though, you can actually choose which level of each factor will be treated as the reference level by R--i.e. which level will be absorbed into the intercept. To do that, I coded each factor variable in the data as an unordered factor and then specified the reference level:

```
EDPI$category_of_offence <- factor(EDPI$category_of_offence, ordered = FALSE)
EDPI$category_of_offence = relevel(EDPI$category_of_offence, ref = "Political")

EDPI$offence <- factor(EDPI$offence, ordered = FALSE)
EDPI$offence = relevel(EDPI$offence, ref = "Espionage")

EDPI$offence_governorate <- factor(EDPI$offence_governorate, ordered = FALSE)
EDPI$offence_governorate = relevel(EDPI$offence_governorate, ref = "Other")

EDPI$defendant_gender <- factor(EDPI$defendant_gender, ordered = FALSE)
EDPI$defendant_gender = relevel(EDPI$defendant_gender, ref = "Male")

EDPI$court_type <- factor(EDPI$court_type, ordered = FALSE)
EDPI$court_type = relevel(EDPI$court_type, ref = "Civilian court")
```

 After reformatting the data, creating dummy variables, and removing unnecessary columns, we are left with a csv that looks like the below (when in table form). (see the bottom of this page for the full block of code I used for this project):

<img src="images/EDPI_logit_raw_screenshot.png?raw=true"/>


### Testing for significance

I then began testing these variables for their significance in influencing the response variable to shift from 0 to 1, i.e. the extent to which they influenced whether or not an individual was ultimately executed.

```javascript
[//]: # Building the first logistic regression with glm function
glm.fit_1 <- glm(executed_0_1 ~ as.factor(category_of_offence) + as.factor(offence) +
                   governorate_sentences + as.factor(defendant_gender) + as.factor(court_type) + 
                   days_btwn_offence_and_crim_1_judgement, data = EDPI, family = binomial)
```
This first logistic regression produced the following result:

<img src="images/edpilogit1.png?raw=true"/>

As above, this first regression found that whether a defendant was tried in a military court (as opposed to a civilian court) had major explanatory power for indicating whether an individual was likely to go on to be executed. However, the model did not find that any of the other variables fed to it (alleged offence, defendant gender, etc.) had any explanatory power. Based on my experience as a human rights defender working on the death penalty in Egypt, I believed that some of these variables were likely significant, but were being clouded by excessive inclusion of less relevant variables. With that in mind, I chose to run a second regression, pared down to the columns that I thought most likely to have real explanatory power. These included the category of offence allegedly committed (political vs criminal), the type of court the defendant was tried in (military vs civilian), and the number of death sentences handed down in the governorate where the individual in question was sentences to death. The code for this second regression looked like this:

```javascript
[//]: # Building the second logistic regression with glm function
glm.fit_2 <- glm(executed_0_1 ~ as.factor(category_of_offence) + as.factor(court_type)
                 + governorate_sentences, data = EDPI, family = binomial)
```

This second logistic regression produces this result:

<img src="images/edpilogit2.png?raw=true"/>

We see now that being charged with a criminal offence rather than a political one, being tried in a military tribunal rather than a civilian court, and the number of death sentences handed down in the relevant geographical area all appear to be strongly correlated with whether an individual defendant will go on to be executed.

We can also calculate the Variance Inflaction Factor (VIF) values of each variable in the model to see if multicollinearity is a problem. We do this by installing the car package and running the VIF function:

```javascript
install.packages('car')
library(car)
vif(glm.fit_2)
```

When we do so, we get this result:

<img src="images/EDPI_collinearity.png?raw=true"/>

Generally, VIF values below 5 indicate no multicollinearity, so it does not appear to be a problem here.


Finally, we can also can also exponentiate the coefficients and interpret them as odds-ratios: 

```javascript
exp(coef(glm.fit_2))
```

This code produces this output:

<img src="images/EDPI_coeff.png?raw=true"/>


### Analysis and explanation

This model tells us that:

- Controlling for other variables, the odds of eventual execution were 2.87 times (287%) higher for individuals charged with criminal offences, as opposed to political ones.
- Controlling for other variables, the odds of eventual execution were 8.23 times (823%) higher for individuals convicted in a military tribunal, as opposed to a civilian court.
- Controlling for other variables, the odds that an individual would go on to be executed were reduced by .003 times (.3%) for every additional death sentence handed down in the governorate where the defendant in question was sentenced to death.


These results make sense. It is not surprising to me that as more death sentences are issued in a given governorate, that corresponds with a slightly reduced chance of any one individual sentenced to death in that governorate going on to be executed. I believe the explanation here is that the highest numbers of death sentences, in governorates like Minya, Cairo, and Giza, resulted from mass trials held in those locations, where hundreds of people were sentenced to death simultaneously in some cases. These trials garnered considerable international scrutiny, and thus were probably less likely to lead to actual executions. Conversely, it was likely easier and less controversial for the government to carry out executions related to death sentences resulting from smaller, lesser-known trials.

It also makes sense that individuals convicted of criminal offences would be more likely to be executed. Criminal offences (as opposed to political ones) are generally viewed by Egyptian authorities as more cut-and-dried and less controversial. These tend to be cases where individuals are convicted of murder related to personal vendettas (for example), as compared to a political offence where a peaceful protestor may be accused of being a terrorist. The government is more likely to feel comfortable carrying out executions in the former cases than the latter, as they garner less international attention.

Likewise, if an individual is tried in a military tribunal instead of a civilian court, that is often indicative of the government's commitment to punishing that individual as harshly as possible. That is not to say that individuals convicted in military tribunals are more likely to be guilty, or are accused of more serious offences than those tried in civilian courts--this is very often not the case. Rather, the government trying an individual in a military tribunal makes clear the government's intent to deal with that person as harshly as possible, so it is not surprising that those cases were more likely to end in execution.

Overall, the human rights investigations field is ripe for applications of this kind of advanced statistical analysis, and indeed needs more of it; this is a big reason why I am pursuing a statistics and data science education. For example, one could use this type of logistic regression to make decisions about where to allocate casework resources within a human rights organization--if individuals sentenced to death for criminal offences, in military tribunals, and in governorates with overall fewer death sentences are more likely to be executed, organizations can focus their resources on those cases.


### Full code block

The full code for this project is as follows

```javascript
[//]: # INSTALLING PACKAGES
install.packages('caret')
install.packages('mctest')
install.packages('car')
install.packages('tidyverse')
install.packages('tidyr')
install.packages('dplyr')


[//]: # UNPACKING LIBRARIES
library(tidyr)
library(tidyverse)
library(dplyr)
library(data.table)
library(car)


[//]: # Set working directory
setwd("C:/Users/benpi/OneDrive/Documents/EDPI")

[//]: # Read in the original EDPI raw data in csv form
EDPI <- read.csv("EDPI logistic regression csv.csv", header = TRUE, sep=",")

[//]: # Select relevant columns
EDPI <- EDPI[, c(1, 3, 6:8, 10:11, 21, 31, 34:37, 45)]

[//]: # Change date formats
EDPI$offence_date <- as.Date(EDPI$offence_date, "%d/%m/%Y")
EDPI$crim_1_judgement_date <- as.Date(EDPI$crim_1_judgement_date, "%d/%m/%Y")

[//]: # Create days_btwn_offence_and_crim_1_judgement column
EDPI$days_btwn_offence_and_crim_1_judgement = EDPI$crim_1_judgement_date-EDPI$offence_date

[//]: # Remove irrelevant values in defendant_status column
EDPI <- subset(EDPI, EDPI$defendant_status!="Died before Referal to the Court" & EDPI$defendant_status!="Died during Procedures")

[//]: # Adding column tracking how many death sentences occurred in each governorate
EDPI = EDPI %>% 
  mutate(governorate_sentences = case_when(offence_governorate == "Minya" ~ 1302,
                                           offence_governorate == "Cairo" ~ 642,
                                           offence_governorate == "Giza" ~ 533,
                                           offence_governorate == "Sharqia" ~ 304,
                                           offence_governorate == "Alexandria" ~ 189,
                                           offence_governorate == "Qena" ~ 156,
                                           offence_governorate == "Sohag" ~ 150,
                                           offence_governorate == "Beheira" ~ 131,
                                           offence_governorate == "Qalyubia" ~ 100,
                                           offence_governorate == "Ismailia" ~ 98,
                                           offence_governorate == "North Sinai" ~ 95,
                                           offence_governorate == "Gharbia" ~ 89,
                                           offence_governorate == "Kafr El-Sheikh" ~ 85,
                                           offence_governorate == "Dakahlia" ~ 83,
                                           offence_governorate == "Faiyum" ~ 68,
                                           offence_governorate == "Monufia" ~ 66,
                                           offence_governorate == "Asyut" ~ 54,
                                           offence_governorate == "Damietta" ~ 47,
                                           offence_governorate == "Port Said" ~ 42,
                                           offence_governorate == "Red Sea" ~ 36,
                                           offence_governorate == "unknown" ~ 34,
                                           offence_governorate == "Aswan" ~ 31,
                                           offence_governorate == "Beni Suef" ~ 25,
                                           offence_governorate == "South Sinai" ~ 20,
                                           offence_governorate == "Luxor" ~ 13,
                                           offence_governorate == "El Wadi El-Gedid" ~ 11,
                                           offence_governorate == "Mersa Matruh" ~ 10,
                                           offence_governorate == "Suez" ~ 10,
                                           TRUE ~ 0))

[//]: # Grouping all goverorates that handed down less than 150 death sentences into a single "other" category:
EDPI$offence_governorate[EDPI$governorate_sentences < 150] <- "Other"


[//]: # Resetting the factor levels. We do this because for each factor, the glm function will eliminate one level
[//]: # of the factor as a reference and absorb it into the intercept. This is necessary, but we can reorder the
[//]: # factors so that the levels of each factor used as the reference (and thus omitted) are not the important
[//]: # ones we want included in our model. Note that the columns are currently ordered factors, and relevel only
[//]: # works on unordered factors, so we change each column from an ordered factor to an unordered one:

EDPI$category_of_offence <- factor(EDPI$category_of_offence, ordered = FALSE)
EDPI$category_of_offence = relevel(EDPI$category_of_offence, ref = "Political")

EDPI$offence <- factor(EDPI$offence, ordered = FALSE)
EDPI$offence = relevel(EDPI$offence, ref = "Espionage")

EDPI$offence_governorate <- factor(EDPI$offence_governorate, ordered = FALSE)
EDPI$offence_governorate = relevel(EDPI$offence_governorate, ref = "Other")

EDPI$defendant_gender <- factor(EDPI$defendant_gender, ordered = FALSE)
EDPI$defendant_gender = relevel(EDPI$defendant_gender, ref = "Male")

EDPI$court_type <- factor(EDPI$court_type, ordered = FALSE)
EDPI$court_type = relevel(EDPI$court_type, ref = "Civilian court")

[//]: # Building the first logistic regression with glm function
glm.fit_1 <- glm(executed_0_1 ~ as.factor(category_of_offence) + as.factor(offence) +
                   governorate_sentences + as.factor(defendant_gender) + as.factor(court_type) + 
                   days_btwn_offence_and_crim_1_judgement, data = EDPI, family = binomial)

summary(glm.fit_1)


[//]: # Refining the logistic regression with glm function
glm.fit_2 <- glm(executed_0_1 ~ as.factor(category_of_offence) + as.factor(court_type)
                 + governorate_sentences, data = EDPI, family = binomial)

summary(glm.fit_2)

[//]: # Use vif to determine multicollinearity. VIF values below 5 reveal no multicollinearity
vif(glm.fit_2)

[//]: # We can also exponentiate the coefficients and interpret them as odds-ratios: 
exp(coef(glm.fit_2))
```

