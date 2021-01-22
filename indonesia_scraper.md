## Using RSelenium to scrape the web for court cases in Indonesia

This is a project I'm working on for a journalist friend who investigates environmental justice issues in Southeast Asia. One such issue relates to prosecutions of smallholder farmers in Indonesia. Indonesia has a major problem with air pollution, and in particular smoky haze, which blankets parts of Southeast Asia each year. Much of this haze originates in Indonesia, on the islands of Sumatra and Borneo, and then wafts across the water to Malaysia and Singapore; the haze is a source of diplomatic tension in the region, as well as a major health hazard.

The source of this haze, in part, is deliberate burning to clear land area in Sumatra and Borneo. It is true that small farmers in these places have practiced swidden agriculture for generations, but the large palm oil and pulpwood conglomerates based in Indonesia also use these tactics on a much larger scale to make space for expanding their plantations. Nevertheless, the Indonesian judiciary continues to prosecute small farmers for burning land, while relatively few plantation concerns are punished.

Information regarding the prosecution of farmers for burning is posted on a series of websites hosted by Indonesia's judiciary; each website pertains to a different federal district of Indonesia, and lists prosecutions and court cases for that district:

### Determining search terms

There is no "executions" tag on the SPA website, so we need to use a search term that will return all execution announcements. The search term I chose is the Arabic word "حكم", which means "judgement" or "verdict". All SPA execution announcement headlines include the phrase "تنفيذ حكم", which means "implementation of verdict", so a search for "حكم" should return all execution announcements:

<img src="images/spa-search-screenshot.png?raw=true"/>


### Determining filtering terms

From there, we need to find a way to filter the larger set of search results whose headlines contain the word "حكم" into a smaller subset that contains exclusively execution announcements. I chose the word "تنفيذ" itself, as that seems to be present only in execution announcement headlines. 


### Examining the HTML

Now we need to inspect the SPA website's HTML to determine exactly where the headline text sits within the search results page. The SPA search results [(e.g.)](https://www.spa.gov.sa/search.php?lang=ar&search=%D8%AD%D9%83%D9%85) are dynamically retrieved by Ajax [(e.g.)](https://www.spa.gov.sa/ajax/search.php?searchbody=1&search=%D8%AD%D9%83%D9%85&cat=0&cabinet=0&royal=0&lang=ar&pg=1&pg=1), so that's where we need to look.

When we inspect the HTML on that Ajax retrieval page, we find that the actual hyperlink/headline titles are contained in the "h2NewsTitle" node, and the corresponding URLs are contained in the "aNewsTitle" node:

<img src="images/ksa-html-screenshot.png?raw=true"/>


### Creating the scraping script

With that information, we can use rvest to create a script that will return a list of all URLs for execution announcements on a given search results page:

```javascript
library(tidyverse)
library(rvest)

//]: # Crawls through HTML on the first page of results for the search term "حكم" and returns all matching titles:
list1 <- read_html("https://www.spa.gov.sa/ajax/search.php?searchbody=1&search=%D8%AD%D9%83%D9%85&cat=0&cabinet=0&royal=0&lang=ar&pg=1&pg=1") %>% html_nodes(".h2NewsTitle") %>% html_text()

[//]: # Crawls through same HTML and returns corresponding URLs:
list2 <- read_html("https://www.spa.gov.sa/ajax/search.php?searchbody=1&search=%D8%AD%D9%83%D9%85&cat=0&cabinet=0&royal=0&lang=ar&pg=1&pg=1") %>% html_nodes(".aNewsTitle") %>% html_attr("href")

[//]: # Returns indices of all list1 elements containing the word "تنفيذ", which appears only in execution announcements:
grep_list <- grep("تنفيذ", list1)

[//]: # Creates blank list:
link_list <- vector("list", length(grep_list))

[//]: # Populates blank list with URLs corresponding to titles of execution announcements in grep_list:
for (i in grep_list){
  link_list[i] <- paste("https://www.spa.gov.sa", list2[i], sep = "")}

[//]: # Removes null values:
link_list = link_list[-which(sapply(link_list, is.null))]

[//]: # Creates data frame and exports to .csv file:
link_df <- as.data.frame(link_list)
link_df <- t(link_df)
write.csv(link_df,"C:/Users/benpi/OneDrive/Documents/R/ksa_executions.csv", row.names = FALSE)
```

This creates a .csv file that contains a list of the relevant URLs:

<img src="images/ksa-links-screenshot.png?raw=true"/>

The script only crawls through one page at a time. This could be sufficient, if one runs the script once a week, but I may also edit it in the future so that it runs through all available search results pages and appends only new URLs that it hasn't found previously.
