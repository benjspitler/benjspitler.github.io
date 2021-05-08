## Using logistic regressions in R to predict executions in Egypt

As discussed [elsewhere](https://benjspitler.github.io/EDPI) in this portfolio, my prior human rights work with [Reprieve](https://reprieve.org/uk/) involved the creation of the [Egypt Death Penalty Index](https://egyptdeathpenaltyindex.com) (EDPI), a comprehensive database tracking all known capital trials in Egypt since 2013, and many from before then as well. The resulting data provides an important real-time view into the Egyptian government's application of capital punishment. The database allows human rights defenders to examine ongoing capital trials and identify individuals in need of legal assistance, which is largely how the EDPI has been used to date.

But given that the EDPI contains such rich historical information on hundreds of capital trials, it occurred to me that this information could also serve as valuable raw material for statistical analysis that might allow us to predict the outcomes of future capital trials based on their characteristics. Specifically, I wondered if I could build a regression model that would identify which factors related to a capital trial might increase the likelihood that a defendant would go on to be executed. The project below represents an early attempt to use logistic regression to answer that question.

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

<img src="images/glm_screenshot_1.png?raw=true"/>

As above, this first regression found that whether a defendant was tried in a military court (as opposed to a civilian court) had major explanatory power for indicating whether an individual was likely to go on to be executed. However, the model did not find that any of the other variables fed to it (alleged offence, defendant gender, etc.) had any explanatory power. Based on my experience as a human rights defender working on the death penalty in Egypt, I believed that some of these variables were likely significant, but were being clouded by excessive inclusion of less relevant variables. With that in mind, I chose to run a second regression, pared down to the columns that I thought most likely to have real explanatory power. These included the category of offence allegedly committed (political vs criminal), the type of court the defendant was tried in (military vs civilian), and the number of death sentences handed down in the governorate where the individual in question was sentences to death. The code for this second regression looked like this:

```javascript
[//]: # Building the second logistic regression with glm function
glm.fit_2 <- glm(executed_0_1 ~ as.factor(category_of_offence) + as.factor(court_type)
                 + governorate_sentences, data = EDPI, family = binomial)
```

This second logistic regression produces this result:

<img src="images/edpilogit1.png?raw=true"/>

We see now that being charged with a criminal offence rather than a political one, being tried in a military tribunal rather than a civilian court, and the number of death sentences handed down in the relevant geographical area all appear to be strongly correlated with whether an individual defendant will go on to be executed.

that **governorate_sentences**, **category_of_offence_Criminal**, and **court_type_Military_court** are all significant, and **category_of_offence_Criminal** has indeed become more significant based on removing **defendant_gender_Female** and **defendant_status_in_Absentia**.

We can also calculate the Variance Inflaction Factor (VIF) values of each variable in the model to see if multicollinearity is a problem. We do this by installing the car package and running the VIF function:

```javascript
install.packages('car')
library(car)
vif(glm.fit_2)
```

When we do so, we get this result:

<img src="images/edpilogit2.png?raw=true"/>

Generally, VIF values below 5 indicate no multicollinearity, so it does not appear to be a problem here.

Note that I have chosen to retain and eliminate variables here manually, but this can also be automated through a stepwise regression process. Stepwise regressions have some [issues in the automation process](https://stats.stackexchange.com/questions/20836/algorithms-for-automatic-model-selection/20856#20856) that mean they are not always the soundest choice, especially if the analyst has a deep understanding of the likely relevance of different explanatory variables, but I do plan to redo this process in the future using an automated stepwise procedure to see if I get different results. I'm also aware that separating this dataset into training and testing groups could improve the accuracy of this model; that's something I'm going to get more into as I spend more time on machine learning principles in the coming months.


### Analysis and explanation

This model tells us that:

- Controlling for other variables, each additional death sentence handed down in the governorate where an individual was sentenced to death changes the log odds of that individual being executed by -0.0023 
- Controlling for other variables, an individual being convicted of a criminal offence (as opposed to a political one) changes the log odds of that individual being executed by 1.05
- Controlling for other variables, an individual being convicted in a military court (as opposed to a civilian court) changes the log odds of that individual being executed by 2.1

We can also exponentiate the coefficients and interpret them as odds-ratios: 

```javascript
exp(coef(glm.fit_2))
```

This produces this result:

<img src="images/glm_screenshot_3.png?raw=true"/>

Which tells us that: 

- Controlling for other variables, each additional death sentence handed down in the governorate where an individual was sentenced to death decreases the  odds of that individual being executed by a factor of just less than 1 
- Controlling for other variables, an individual being convicted of a criminal offence (as opposed to a political one) increases the odds of that individual being executed by a factor of 2.87
- Controlling for other variables, an individual being convicted in a military court (as opposed to a civilian court) increases the  odds of that individual being executed by a factor of 8.23

These results make sense. It is not surprising to me that as more death sentences are issued in a given governorate, that corresponds with a slightly reduced chance of any one individual sentenced to death in that governorate going on to be executed. I believe the explanation here is that the highest numbers of death sentences, in governorates like Minya, Cairo, and Giza, resulted from mass trials held in those locations, where hundreds of people were sentenced to death simultaneously in some cases. These trials garnered considerable international scrutiny, and thus were probably less likely to lead to actual executions. Conversely, it was likely easier and less controversial for the government to carry out executions related to death sentences resulting from smaller, lesser-known trials.

It also makes sense that individuals convicted of criminal offences would be more likely to be executed. Criminal offences (as opposed to political ones) are generally viewed by Egyptian authorities as more cut-and-dried and less controversial. These tend to be cases where individuals are convicted of murder related to personal vendettas (for example), as compared to a political offence where a peaceful protestor may be accused of being a terrorist. The government is more likely to feel comfortable carrying out executions in the former cases than the latter, as they garner less international attention.

Likewise, if an individual is tried in a military tribunal instead of a civilian court, that is often indiciative of the government's commitment to punishing that individual as harshly as possible. That is not to say that individuals convicted in military tribunals are more likely to be guilty, or are accused of more serious offences than those tried in civilian courts--this is very often not the case. Rather, the government trying an individual in a military tribunal makes clear the government's intent to deal with that person as harshly as possible, so it is not surprising that those cases were more likely to end in execution.

Overall, the human rights investigations field is ripe for applications of this kind of advanced statistical analysis, and indeed needs more of it; this is a big reason why I am pursuing a statistics and data science education. For example, one could use this type of logistic regression to make decisions about where to allocate casework resources within a human rights organization--if individuals sentenced to death for criminal offences, in military tribunals, and in governorates with overall fewer death sentences are more likely to be executed, organizations can focus their resources on those cases.


### Full code block

The full code for this project is as follows

```javascript
[//]: # INSTALLING PACKAGES
[//]: # Install fastDummies package:
install.packages('fastDummies')
[//]: # https://www.marsja.se/create-dummy-variables-in-r/#Dummy_Coding
[//]: # https://www.statology.org/logistic-regression-in-r/#:~:text=%20How%20to%20Perform%20Logistic%20Regression%20in%20R,Model.%20The%20coefficients%20in%20the%20output...%20More%20
install.packages('caret')
install.packages('mctest')
[//]: # car package needed to run VIF function later to determine multicollinearity
install.packages('car')


[//]: # UNPACKING LIBRARIES
library(tidyr)
library(tidyverse)
library(dplyr)
library(data.table)
[//]: # car is for testing for multicollinearity below
library(car)
[//]: # fastdummies is for auto creation of dummy variables below
library('fastDummies')


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

[//]: # create dummy variables:
EDPI_logit <- dummy_cols(EDPI, select_columns = c('category_of_offence','offence', 'offence_year', 
                                                  'offence_period', 'offence_governorate',
                                                  'defendant_gender',
                                                  'court_type', 'crim_1_verdict',
                                                  "defendant_status", 'crim_1_judgement_year'))

[//]: # Retaining just the numerical and categorical variables you want. Note, the list below is all of the useful
[//]: # ones, but you may need to pare that down further for more targeted regressions
EDPI_logit_selected <- EDPI_logit[, c(2, 12:100)]

[//]: # Fixing the column names
setnames(EDPI_logit_selected, old = c('offence_Civilian clashes',
                                   'offence_Membership in a terrorist organisation',
                                   'offence_Sit-in clashes',
                                   'offence_Storming government installations',
                                   'offence_Terrorism toward religious minorities',
                                   'offence_Terrorist Acts',
                                   'offence_period_Adly Mansour',
                                   'offence_governorate_Beni Suef',
                                   'offence_governorate_El-Wadi El-Gedid',
                                   'offence_governorate_Kafr El-Sheikh',
                                   'offence_governorate_Mersa Matruh',
                                   'offence_governorate_North Sinai',
                                   'offence_governorate_Port Said',
                                   'offence_governorate_Red Sea',
                                   'offence_governorate_South Sinai',
                                   'defendant_status_in Absentia',
                                   'court_type_Civilian court',
                                   'court_type_Military court',
                                   'crim_1_verdict_Acquittal after Preliminary death sentence, not yet confirmed',
                                   'crim_1_verdict_Death sentence',
                                   'crim_1_verdict_Preliminary death sentence, not yet confirmed',
                                   'crim_1_verdict_Prison term after preliminary death sentence, not yet confirmed'),
         new = c('offence_Civilian_clashes','offence_Membership_in_a_terrorist_organisation',
                 'offence_Sit_in_clashes',
                 'offence_Storming_government_installations',
                 'offence_Terrorism_toward_religious_minorities',
                 'offence_Terrorist_Acts',
                 'offence_period_Adly_Mansour',
                 'offence_governorate_Beni_Suef',
                 'offence_governorate_El_Wadi_El_Gedid',
                 'offence_governorate_Kafr_El_Sheikh',
                 'offence_governorate_Mersa_Matruh',
                 'offence_governorate_North_Sinai',
                 'offence_governorate_Port_Said',
                 'offence_governorate_Red_Sea',
                 'offence_governorate_South_Sinai',
                 'defendant_status_in_Absentia',
                 'court_type_Civilian_court',
                 'court_type_Military_court',
                 'crim_1_verdict_Acquittal_after_Preliminary_death_sentence_not_yet_confirmed',
                 'crim_1_verdict_Death_sentence',
                 'crim_1_verdict_Preliminary_death_sentence_not_yet_confirmed',
                 'crim_1_verdict_Prison_term_after_preliminary_death_sentence_not_yet_confirmed'))

[//]: # Paring down the list for the first logistic regression. This selection was made by trial and error to see which variables would be
[//]: # significant, as opposed to a stepwise process, which has issues as laid out here:
[//]: # https://stats.stackexchange.com/questions/20836/algorithms-for-automatic-model-selection/20856#20856
EDPI_logit_1 <- EDPI_logit_selected[, c(1, 6, 71, 74, 79, 5)]

[//]: # Building the first logistic regression with glm function
[//]: # https://www.datacamp.com/community/tutorials/logistic-regression-R
glm.fit_1 <- glm(executed_0_1 ~ category_of_offence_Criminal + defendant_gender_Female +
                 court_type_Military_court + defendant_status_in_Absentia + governorate_sentences,
               data = EDPI_logit_1, family = binomial)

[//]: # We see from the summary below that gender and status are not significant, and category of offence is, but just barely,
[//]: # so let's remove gender and status and see if that improves the model
summary(glm.fit_1)

[//]: # Paring down the list further for the second logistic regression.
[//]: # https://stats.stackexchange.com/questions/20836/algorithms-for-automatic-model-selection/20856#20856
EDPI_logit_2 <- EDPI_logit_selected[, c(1, 5, 6, 74)]

[//]: # Building the second logistic regression with glm function
[//]: # https://www.datacamp.com/community/tutorials/logistic-regression-R
glm.fit_2 <- glm(executed_0_1 ~ governorate_sentences + category_of_offence_Criminal +
                  court_type_Military_court,
                data = EDPI_logit_2, family = binomial)

[//]: # Now we see that governorate_sentences, category_of_offence_Criminal, and _court_type_Military_court are all significant,
[//]: # and category_of_offence_Criminal has gotten more significant based on removing status and gender
summary(glm.fit_2)

[//]: # Use vif to determine multicollinearity. VIF values below 5 reveal no multicollinearity
[//]: # https://www.statology.org/logistic-regression-in-r/#:~:text=%20How%20to%20Perform%20Logistic%20Regression%20in%20R,Model.%20The%20coefficients%20in%20the%20output...%20More%20
vif(glm.fit_2)

[//]: # We can also exponentiate the coefficients and interpret them as odds-ratios: 
[//]: # https://stats.idre.ucla.edu/r/dae/logit-regression/
exp(coef(glm.fit_2))
```
