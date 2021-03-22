## Using logistic regressions in R to predict executions in Egypt

As discussed [elsewhere](https://benjspitler.github.io/EDPI) in this portfolio, my prior human rights work with [Reprieve](https://reprieve.org/uk/) involved the creation of the [Egypt Death Penalty Index](https://egyptdeathpenaltyindex.com) (EDPI), a comprehensive database tracking all known capital trials in Egypt since 2013, and many from before then as well. The resulting data provides an important real-time view into the Egyptian government's application of capital punishment. The database allows human rights defenders to examine ongoing capital trials and identify individuals in need of legal assistance, which is largely how the EDPI has been used to date.

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

I then began testing these variables for their significance in influencing the response variable to shift from 0 to 1, i.e. the extent to which they influenced whether or not an individual was ultimately executed. The first thing I noticed was that the **days_btwn_offence_and_crim_1_judgement** had too many null values to be truly useful; these resulted from individuals whose original offence date or initial verdict date were unknown. I decided to eliminate this column from contention. From there, a first ran a logistic regression using the **executed_0_1** as the response variable and the remaining columns as explanatory variables: **category_of_offence_Criminal**, **defendant_gender_Female**, **court_type_Military_court*, **defendant_status_in_Absentia**, and **governorate_sentences**. The code for that looked like this:

```javascript
[//]: # Building the first logistic regression with glm function
glm.fit_1 <- glm(executed_0_1 ~ category_of_offence_Criminal + defendant_gender_Female +
                 court_type_Military_court + defendant_status_in_Absentia + governorate_sentences,
               data = EDPI_logit_1, family = binomial)

summary(glm.fit_1)
```
This first logistic regression produced the following result:

<img src="images/glm_screenshot_1.png?raw=true"/>

We notice based on this regression result that the **defendant_gender_Female** and **defendant_status_in_Absentia** variables are not significant (based on their high p-values), and the **category_of_offence_Criminal** variable is only significant at a p-value greater than 0.05. Based on those observations, I chose to remove **defendant_gender_Female** and **defendant_status_in_Absentia** and see if that improves the model:

```javascript
# Building the second logistic regression with glm function
glm.fit_2 <- glm(executed_0_1 ~ governorate_sentences + category_of_offence_Criminal +
                  court_type_Military_court,
                data = EDPI_logit_2, family = binomial)

summary(glm.fit_2)
```

This second logistic regression produces this result:

<img src="images/glm_screenshot_2.png?raw=true"/>

We see now that **governorate_sentences**, **category_of_offence_Criminal**, and **court_type_Military_court** are all significant, and **category_of_offence_Criminal** has indeed become more significant based on removing **defendant_gender_Female** and **defendant_status_in_Absentia**.

Note that I have chosen to retain and eliminate variables here manually, but this can also be automated through a stepwise regression process. Stepwise regressions have some [issues in the automation process](https://stats.stackexchange.com/questions/20836/algorithms-for-automatic-model-selection/20856#20856) that mean they are not always the soundest choice, especially if the analyst has a deep understanding of the likely relevance of different explanatory variables, but I do plan to redo this process in the future using an automated stepwise procedure to see if I get different results.


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

Overall, these results make sense. It is not surprising to me that as more death sentences are issued in a given governorate, that corresponds with a slightly reduced chance of any one individual sentenced to death in that governorate going on to be executed. I believe the explanation here is that the governorates with the highest numbers of death sentences, like Minya, Cairo, and Giza, are reflective of mass trials held in those locations, where hundreds of people were sentenced to death simultaneously in some cases. These trials garnered considerable international scrutiny, and thus were probably less likely to lead to actual executions.

It also makes sense that individuals committed of criminal offences and individuals convicted in military courts would be more likely to be executed, though for different reasons. Criminal offences (as opposed to political ones) are generally viewed by authorities as more cut-and-dried and less controversial. These tend to be cases where individuals are convicted of murder related to personal vendettas (for example), as compared to a political offence where a peaceful protestor is accused of being a terrorist. The government is more likely to feel comfortable carrying out executions in the former cases than the latter. Likewise, if an individual is tried in a military tribunal instead of a civilian court, that is often indiciative of the government's commitment to punishing that individual as harshly as possible. That is not to say that individuals convicted in military tribunals were more likely to be guilty, or were accused of more serious offences than those tried in civilian courts--this was very often not the case. Rather, the government trying an individual in a military tribunal makes clear the government's intent to deal with that person as harshly as possible, so it is not surprising that those cases were more likely to end in execution.

