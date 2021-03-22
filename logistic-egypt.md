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

summary(glm.fit_1)
```

```javascript
install.packages("RSelenium")
install.packages("tidyverse")

library(RSelenium)
library(rvest)
library(stringr)
library(tidyverse)
library(dplyr)

[//]: # Initiate Selenium browser.
driver <- rsDriver(browser=c("chrome"), chromever = "87.0.4280.88")
remote_driver <- driver[["client"]]
remote_driver$open()

[//]: # Navigate to search page
remote_driver$navigate("http://sipp.pn-pangkalanbun.go.id/list_perkara/search")

[//]: # Locate search box
search_element <- remote_driver$findElement(using = 'id', value = 'search-box')

[//]: # Enter search term
search_element$sendKeysToElement(list("Kebakaran Hutan"))

[//]: # Locate search button
button_element <- remote_driver$findElement(using = 'id', value = 'search-btn1')

[//]: # Click search button
button_element$clickElement()

[//]: # Extract results table as html_table
scraped_table <- read_html(remote_driver$getPageSource()[[1]]) %>%
  html_nodes(xpath = '//*[@id="tablePerkaraAll"]') %>% html_table()

[//]: # Convert table to data frame
scraped_table_df <- as.data.frame(scraped_table)

[//]: # Rename table columns, adding underscores
colnames(scraped_table_df) <- c("No", "Nomor_Perkara", "Tanggal_Register", "Klasifikasi_Perkara", "Para_Pihak", "Status_Perkara", "Lama_Proses", "Link")

[//]: Remove first row (which has column names in it)
scraped_table_df = scraped_table_df[-1,]

[//]: # Remove rownames
rownames(scraped_table_df) = NULL

[//]: # Delete "Link" column, which currently has just the hyperlink title in it ("detil")
scraped_table_df <- scraped_table_df[, -8]

[//]: # Convert "No" column to integer
scraped_table_df$No <- as.integer(scraped_table_df$No)

[//]: # Retrieve all URLs on the page residing in the "a" node. This returns all of the URLs on the whole page, which is too many. The last "n" of these links are the ones we want, where "n" is the number of rows in scraped_table_df
links <- read_html(remote_driver$getPageSource()[[1]]) %>%
  html_nodes("a") %>% html_attr("href")

[//]: # Turn "links" into a list
link_list <- as.list(links)

[//]: # Retain only the last "n" links, where "n" is the number of rows in scraped_table_df
link_list_2 <- tail(link_list, (nrow(scraped_table_df)))

[//]: # Append link_list_2 to scraped_table_df as "Link" column:
scraped_table_df$Link <- sapply(link_list_2, paste0)

[//]: # Read in previous version of csv file. This is the sheet where we store information previously extracted via this script, so that we can examine this data and only add **new** information to it
comb_df <- read.csv("farmers_r_sheet.csv")

[//]: # Change Link column to character type
comb_df$Link <- as.character(comb_df$Link)


[//]: # Use anti_join to get rows in scraped_table_df that are not in comb_df and bind them with comb_df. This is how we ensure we are only adding new information to our data base, and not duplicating information we previously added. The unique id we use for matching purposes is the "Nomor_Perkara" (or "case number") column.
comb_df <- bind_rows(scraped_table_df, anti_join(comb_df, scraped_table_df, by = 'Nomor_Perkara'))

[//]: # Converting "Tanggal_Register" column to date format
comb_df$Tanggal_Register <- as.Date(comb_df$Tanggal_Register, "%d %b %Y")

[//]: # Writing the newly updated data back to our working directory
write.csv(comb_df, "farmers_r_sheet.csv", row.names = FALSE)
```
