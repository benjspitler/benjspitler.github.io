## Using logistic regressions in R to predict executions in Egypt

As discussed elsewhere in this portfolio, my prior human rights work with [Reprieve](https://reprieve.org/uk/) involved the creation of the [Egypt Death Penalty Index](https://egyptdeathpenaltyindex.com) (EDPI), a comprehensive database tracking all known capital trials in Egypt since 2013, and many from before then as well. The resulting data provides an important real-time view into the Egyptian government's application of capital punishment. The database allows human rights defenders to examine ongoing capital trials and identify individuals in need of legal assistance, which is largely how the EDPI has been used to date.

But given that the EDPI contains such rich historical information on hundreds of capital trials, it occurred to me that this information could also serve as valuable raw material for statistical analysis that might allow us to predict the outcomes of capital trials based on their characteristics. Specifically, I wondered if I could build a regression model that would identify which factors related to a capital trial might increase the likelihood that a defendant would indeed go on to be executed. The project below represents an early attempt to use logistic regression to answer that question.

### Paring down raw data

The first step was to pare down the raw data that forms the back end of the EDPI ([downloadable here](https://egyptdeathpenaltyindex.com/download-data)) into a format that could serve as the basis for regression analysis. The EDPI data contains a wealth of information about individual defendants in capital trials in Egypt, including:

- Demographic information about defendants, such as age, gender, birthplace, occupation, etc.
- Situational information about alleged offences/crimes, such as time/place of alleged occurrence, type of offence, etc.
- Procedural information about trials, such as verdicts reached at different appeal phases, time periods in which judgements were handed down, presence/absence of defendants, whether a defendant was ultimately executed, etc.

That data is rich and useful, but messy for the purposes of regression. It looks like this:

<img src="images/EDPI_raw_screenshot.png?raw=true"/>

Some of these pieces of metadata are better suited than others to serving as predictors. For example, information regarding when a final verdict was reached in a case may help us determine which years in the past produced the most capital trials that led to executions, but that information is unlikely to help us predict the outcome of future cases. To prepare the data for use in a logistic regression process, I identified the variables most relevant for these predictive purposes, and settled on the following:

- Whether or not an individual was ultimately executed (column name **executed_0_1**). This would be the response variable. Any individual who was ultimately executed received a 1 in this column, anyone who was not executed received a 0.
- Whether the offence was "political" or "criminal" in nature, that is whether  the alleged facts of the case and the perceived motivation for the commission of the offence were in some way connected to the political and societal changes that have arisen in Egypt since the January 2011 revolution or not (column name **category_of_offence**).
- The gender of the defendant (column name **defendant_gender**).
- Whether the defendant was tried in a civilian court or a military tribunal (column name **court_type**).
- Whether the defendant was tried _in absentia_ or was present during proceedings (column name **defendant_status**).
- The amount of time elapsed between the defendant's alleged offence and the court reaching a verdict in that defendant's case in the first instance (column name **days_btwn_offence_and_crim_1_judgement**).
- The number of overall death sentences handed down in the governorate where the individual in question was being tried (column name **governorate_sentences**).

For the first five columns, which are binary in nature, I used R's fastDummies package to create dummy variables for each category (see the code at the end of this project for a full description of how this was accomplished). The **days_btwn_offence_and_crim_1_judgement** was achieved by subtracting the **date_of_offence** column from the **date_of_first_verdict** column in the original raw data. The **governorate_sentences** column was achieved by adding a new column to the data which populates with the number of death sentences from the governorate corresponding to where each row's offence occurred. After reformatting the data, creating dummy variables,a nd removing unnecessary columns, we are left with a csv that looks like the below (when in table form):

<img src="images/EDPI_logit_raw_screenshot.png?raw=true"/>


### Testing for significance

I then began testing these variables for their significance in influencing the response variable to shift from 0 to 1, i.e. the extent to which they influenced whether or not an individual was ultimately executed. The first thing I noticed was that the **days_btwn_offence_and_crim_1_judgement** had too many null values to be truly useful; these resulted from individuals whose original offence date or initial verdict date were unknown. I decided to eliminate this column from contention. From there, a first ran a logistic regression using the **executed_0_1** as the response variable and the remaining columns as explanatory variables: **category_of_offence**, **defendant_gender**, **court_type**, **defendant_status**, and **governorate_sentences**. The code for that looked like this:

```javascript
[//]: # Building the first logistic regression with glm function
glm.fit_1 <- glm(executed_0_1 ~ category_of_offence_Criminal + defendant_gender_Female +
                 court_type_Military_court + defendant_status_in_Absentia + governorate_sentences,
               data = EDPI_logit_1, family = binomial)
```
