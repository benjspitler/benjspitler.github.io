## Using KNN classification and logistic regressions in R to predict executions in Egypt

As discussed [elsewhere](https://benjspitler.github.io/EDPI) in this portfolio, my prior human rights work with [Reprieve](https://reprieve.org/uk/) involved the creation of the [Egypt Death Penalty Index](https://egyptdeathpenaltyindex.com) (EDPI), a comprehensive database tracking all known capital trials in Egypt since 2013, and many from before then as well. The resulting data provides an important real-time view into the Egyptian government's application of capital punishment. The database allows human rights defenders to examine ongoing capital trials and identify individuals in need of legal assistance, which is largely how the EDPI has been used to date.

But given that the EDPI contains such rich historical information on hundreds of capital trials, I wondered if this information could also serve as the basis for machine learning modeling to predict the outcomes of future capital trials based on their characteristics. Specifically, I wondered if I could build models that would identify which factors related to a capital trial might increase the likelihood that a defendant would go on to be executed. In the EDPI dataset, each row represents one individual who received a prelmiinary death sentence in an Egyptian court. The dataset contains dozens of columns containing demographic and legal information about each individual, including age, gender, location of arrest, alleged offence, and details about what happened at each stage of the legal process in the individuals trial(s). Crucially, the dataset also indicates whether each individual indeed went on to be executed, or if some other outcome occured (acquittal, commutation, etc.). This is an ideal structure for classification modeling to predict how likely future individuals are to be executed, based on demographic and legal information. The project below represents my attempt to use logistic regression and K-nearest neighbors classification to answer this question.

### Paring down raw data

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
