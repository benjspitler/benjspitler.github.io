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

- Whether or not an individual was ultimately executed (column name **executed_0_1**). This would be the response variable.
- Whether the offence was "political" or "criminal" in nature, that is whether  the alleged facts of the case and the perceived motivation for the commission of the offence were in some way connected to the political and societal changes that have arisen in Egypt since the January 2011 revolution or not (column name **category_of_offence**).
- The gender of the defendant (column name **defendant_gender**).
- Whether the defendant was tried in a civilian court or a military tribunal (column name **court_type**).
- Whether the defendant was tried _in absentia_ or was present during proceedings (column name **defendant_status**).
- The amount of time elapsed between the defendant's alleged offence and the court reaching a verdict in that defendant's case in the first instance (column name **days_btwn_offence_and_crim_1_judgement**).
- The number of overall death sentences handed down in the governorate where the individual in question was being tried (column name **governorate_sentences**).

For the first five columns, which are binary in nature, I used R's fastDummies package to create dummy variables for each category (see the code at the end of this project for a full description of how this was accomplished). The **days_btwn_offence_and_crim_1_judgement** was achieved by subtracting the **date_of_offence** column from the **date_of_first_verdict** column in the original raw data. The **governorate_sentences** column was achieved by adding a new column to the data which populates with the number of death sentences from the governorate corresponding to where each row's offence occurred. After reformatting the data, creating dummy variables,a nd removing unnecessary columns, we are left with a csv that looks like the below (when in table form):

<img src="images/EDPI_logit_sample.png?raw=true"/>

In the course of working on individual cases, it became problematic that there did not exist anywhere in the world a single, codified database tracking all of those on Egypt's death row. It was understandable why that resource didn't exist--it would need to include detailed information about thousands of individuals, and building it would require considerable resources. That sounded like a challenge we could meet, so we set out to build it. With funding from the German Federal Foreign Office, I spent more than a year working with a team of researchers, lawyers, and human rights activists based in both London and Cairo. We fanned out across Egypt, collected and digitized paper court judgments, conducted interviews with victims and their family members, and built the world's first comprehensive database tracking Egypt's massive, unlawful application of the death penalty: the [Egypt Death Penalty Index].



In the governorates where more death sentences occurred, the data points are larger and darker red. The scale here is logarithmic, which allows for better comparison--without a logarithmic scale, some governorates, like Minya, which saw more than 1000 death sentences, would dwarf all others. This viz isn't perfect--the labels are overlapping and somewhat difficult to read in the Nile Delta area, where governorates are small and close together, and the logarithmic scale leaves the viewer unable to tell exactly how many death sentences occurred in each location. That said, it provides a good summary of where most death sentences were handed down.

The code for this viz looks like this:

```javascript
[//]: # Import statements
import pandas as pd
import matplotlib.pyplot as plt
plt.style.use('classic')
%matplotlib inline
import numpy as np
import sys
import cartopy.crs as ccrs
import cartopy.feature as cf
import cartopy.io.shapereader as shapereader
import geopandas
sys.path

[//]: # Reading in the data from .csv file. This .csv contains 3000+ rows, each of which contains information on a discrete individual sentenced to death by an Egyptian court between 2011 and 2018
EDPI = pd.read_csv("egypt_individuals_csv.csv", sep=",")

[//]: # Removing rows where date of death sentence is unknown
EDPI = EDPI[EDPI['Longitude of Offence'] != 'Unknown']
EDPI = EDPI[EDPI['Longitude of Offence'] != 'unknown']
EDPI = EDPI[EDPI['Latitude of Offence'] != 'Unknown']
EDPI = EDPI[EDPI['Latitude of Offence'] != 'unknown']

[//]: # Converting latitude and longitude to float types
EDPI['Latitude of Offence'] = EDPI['Latitude of Offence'].astype(float)
EDPI['Longitude of Offence'] = EDPI['Longitude of Offence'].astype(float)

[//]: # Creating new data frame grouped by governorate, latitude, and longitude, counting the number of death sentences in each location
EDPI_map_df = EDPI.groupby( [ 'Governorate of offence', 'Latitude of Offence', 'Longitude of Offence'] ).size().to_frame(name = 'count').reset_index()

[//]: # Defining parameters for the plot
lat, lon = EDPI_map_df['Latitude of Offence'], EDPI_map_df['Longitude of Offence']
count = EDPI_map_df['count']
gov = EDPI_map_df['Governorate of offence']


[//]: # Plotting the map using cartopy and matplotlib
fig = plt.figure(figsize=(12,12))
ax = fig.add_subplot(1,1,1, projection=ccrs.PlateCarree())
ax.stock_img()
ax.coastlines()
ax.add_feature(cfeature.STATES)
ax.set_extent([24.5, 37, 21.5, 32],
              crs=ccrs.PlateCarree()) ## Important

plt.scatter(x=lon, y=lat,
            cmap="YlOrRd",
            c=np.log10(count),
            s=count,
            alpha=0.8,
            edgecolors = "black",
            transform=ccrs.PlateCarree()) ## Important

[//]: # Creating and adjusting the location of labels for the data points. I did this initially with a for loop, but the labels were too crowded, so I ended up doing this part manually
plt.annotate("Alexandria", (lon[0]-.85, lat[0]+.2))
plt.annotate("Aswan", (lon[1]+.1, lat[1]-.1))
plt.annotate("Asyut", (lon[2]+.2, lat[2]-.1))
plt.annotate("Beheira", (lon[3]-.7, lat[3]-.25))
plt.annotate("Beni Sueif", (lon[4]+.1, lat[4]-.25))
plt.annotate("Cairo", (lon[5]+.25, lat[5]-.1))
plt.annotate("Dakahlia", (lon[6]+.1, lat[6]-.1))
plt.annotate("Damietta", (lon[7]-.2, lat[7]+.19))
plt.annotate("Faiyum", (lon[8]-.95, lat[8]-.05))
plt.annotate("Gharbia", (lon[9]+.07, lat[9]-.05))
plt.annotate("Giza", (lon[10]-.75, lat[10]-.05))
plt.annotate("Ismailia", (lon[11]+.1, lat[11]))
plt.annotate("Kafr El Sheikh", (lon[12]-.65, lat[12]+.1))
plt.annotate("Luxor", (lon[13]+.1, lat[13]-.1))
plt.annotate("Mersa Matruh", (lon[14]-.3, lat[14]+.1))
plt.annotate("Minya", (lon[15]+.45, lat[15]-.1))
plt.annotate("Monufia", (lon[16]-1, lat[16]))
plt.annotate("New Valley", (lon[17]-.55, lat[17]-.3))
plt.annotate("North Sinai", (lon[18]-.6, lat[18]-.3))
plt.annotate("Port Said", (lon[19], lat[19]+.05))
plt.annotate("Qalyubia", (lon[20]-1.05, lat[20]-.15))
plt.annotate("Qena", (lon[21]+.1, lat[21]-.1))
plt.annotate("Red Sea", (lon[22]-1.05, lat[22]-.1))
plt.annotate("Sharqia", (lon[23]-.15, lat[23]-.25))
plt.annotate("Sohag", (lon[24]-.85, lat[24]-.1))
plt.annotate("South Sinai", (lon[25], lat[25]))
plt.annotate("Suez", (lon[26], lat[26]))


[//]: # Creating colorbar and adjusting size
plt.colorbar(shrink=0.667)
plt.show()
```
