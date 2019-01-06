## Introduction

This section gives an introduction to the Birthrate prediction Capstone project for IBM Applied Data Science Specialization.

This project aims to extract data from various data sources such as Wikipedia, [Greater London Authority London Datastore](https://data.london.gov.uk), Yandex (using geopy) and Foursquare, clean and combine them into a dataframe that contains population density, POIs availability and GFR (General fertility rate: live births per 1,000 women aged 15-44) on London boroughs.
Then, following the application of [PCA](https://en.wikipedia.org/wiki/Principal_Component_Analysis), a lower dimension data is generated and used as training to regression algorithms (linear and non-linear) in order to create a model that can predict GFR based on the remaining parameters.

This can be of interest to social science researchers who investigate the impact of POIs availability on the fertility rate, or to city planners, etc..

## Data 

This section gives an overview of the data used to create the regression models discussed above.

The data was extracted from the following sources:
* [Wikipedia - List of areas of London](https://en.wikipedia.org/wiki/List_of_areas_of_London) - Table of neighborhoods in London, the borough(s) they belong to, and postcodes.
* [Wikipedia - List of London boroughs](https://en.wikipedia.org/wiki/List_of_London_boroughs) - Table of boroughs, their area and population and co-ordinates.
* Yandex API (using geopy python package) - to identify the co-ordinates of each London neighborhood.
* Foursquare API - to identify POIs (Point Of Interest) availability for each neighborhood.
* [London Datastore - Births and Fertility Rates, Borough](https://data.london.gov.uk/dataset/births-and-fertility-rates-borough) - mLive births by local authority of usual residence of mother, General Fertility Rates and Total Fertility Rates.

An example of the finalised dataframe, combining all data sources can be seen below:

|   | Borough | arts_entertainment | building | education | event | food | nightlife | parks_outdoors | shops | travel | population_density (ppl/sq. mile) | GFR |
| - | - | - | - | - | - | - | - | - | - | - | - | - |
| 0 | Barking and Dagenham | 12.0 | 10.0 | 0.0 | 0.0 | 79.0 | 24.0 | 22.0 | 60.0 | 16.0 | 13952.05 | 82.6 |

## Methodology

This section gives an overview of the methodology used to approach the discussed problem.

### Data input & preprocessing

#### Wikipedia data 

The London areas table scraped from wikipedia ([Wikipedia - List of areas of London](https://en.wikipedia.org/wiki/List_of_areas_of_London)) contains a row mentioning the borough(s) where an area belongs to. Since some locations belong to more than one borough, we want to break down such locations to rows of "location (borough)". After examaning the dataset, we discovered that we have following potential separators: `,`, `&`, `and`. A closer look at the data reveals that all `and`s are part of borough names (`Hammersmith and Fulham`, `Barking and Dagenham`, `Kensington and Chelsea`), except these two cases:
* `Camden and Islington`
* `Haringey and Barnet`

Comma (with a possible space after it) and ampersand (`Islington & City`), on the other hand, are indeed borough names separators. Hence, we first replace `Camden and Islington` and `Haringey and Barnet` entries with `Camden,Islington` and `Haringey,Barnet`, respectively, and then do the split.

To resolve this, a list of London boroughs is also scraped from Wikipedia into a table (T2) and boroughs in T1 are checked against T2. If a match is not found, the borough string is split on commas and the check is repeated. If that also fails, the string is also split on "and" and he check is repeated. This iterative process yielded accurate results.

Also, some locations in the ares table from wikipedia have duplicate names (but different boroughs), so we rename them to "location (borough)" too.

Then we use the boroughs table from wikipedia ([Wikipedia - List of London boroughs](https://en.wikipedia.org/wiki/List_of_London_boroughs)) to extract the table of London boroughs and their co-ordinates. We use this table to check for borough entries validity in the areas table. We find out that we have one illegal borough -- Dartford -- that does not formally belong to London but Kent. Hence we removed the respective table row.

#### Locations co-ordinates

We used Yandex API, through python geopy package, to retrieve locations (areas) coordinates.

#### Location categories per area

We used Forsquare API to extract at most 100 nearest POIs (1000 metres radius) for each. For each POI, we extracted its "prefix" (not "category", since it's too specific and gives too many features afterwards), and saved rows "location - prefix" to a new dataset. This gave us 27,494 entries. After one-hot encoding and grouping over locations (and applying `sum()` function), we get 9 POIs categories and their counts for each location:

| | Neighborhood | arts_entertainment | building | education | event | food | nightlife | parks_outdoors | shops | travel |
| - | - | - | - | - | - | - | - | - | - | - |
| 0 | Abbey Wood | 1 | 0 | 0 | 0 | 2 | 0 | 2 | 4 | 1 |
| 1 | Acton (Ealing) | 1 | 3 | 0 | 0 | 23 | 8 | 3 | 12 | 7 |

#### London datastore details

Through the Greater London Authority Datastore, we gathered GFR rates per London borough. We descovered that all borough names in the respective GFR dataset are valid names, except `Hackney and City of London`, i.e., GFR for this borough combines data for Hackney borough and the City, which is not a borough, but is the 33rd principal division of Greater London. Hence, we renamed all `City` and `Hackney` boroughs in the ares dataset to `Hackney and City of London` to match this approach.

Finally, we group our areas dataset over boroughs (summing the POIs categories avalability) and merge it with GFR data, and population density from the boroughs dataset. This is out finalized dataset that we'll build and test out model on.

### Model building

The purpose of this project was to create a regression model that can accurately predict GFR in London boroughs. This is currently a proof-of-concept work, since publicly available data is relatively small, and hence models performance can be improved if more data is given (either historical, or per-location GFR).

#### Final preprocessing

Data were split into training (80%) and testing (20%). Two pipelines were created, each comprised of StandardScaler, PCA and the regression model self (multiple linear model with ridge normalization and Support Vector Machine regression model with RBF kernel).

#### Linear model

For multiple linear regression, a Ridge regression model was derived. The hyper parameter `alpha` was fine-tuned using grid search.

#### Non-linear model

For multiple non-linear regression, a Support Vector Machine (SVM) regression with a Radial Basis Function (RBF) kernel was derived. In that case, model parameters were optimised through k-folding. The hyper parameters `C`, `gamma` and `epsilon` were fine-tuned using randomized search.

## Results

Using k-fold validation, following hyper parameters were selected:
* Linear model: `alpha ~ 41.41`
* SVR model: `C ~ 160.3`, `gamma ~ 0.1` and `epsilon` ~ 0.87

The linear model provided an RMSE (Root Mean Square Error) of 7.73 live births per 1,000 women aged 15-44; and R2 score: 0.66 at validation.

The SVR model provided an RMSE (Root Mean Square Error) of 6.19 live births per 1,000 women aged 15-44; and R2 score: 0.8 at validation.

## Discussion
London has [GFR of 62.9](https://data.london.gov.uk/dataset/births-and-fertility-rates-borough), therefore using a linear model we get a median error of 12.29%, and of 9.84% using a non-linear module, only using publicly available information. This error is expected to decrease if larger dataset is used, by using historical data, or by acquireing GFR per London areas.

## Conclusion
This project illustrates an adequate approach for gathering, pre-processing and merging of publicly available data from different sources for the construction of a regression model that can predict General Fertility Rate (GFR). The error (of 12.29% and 9.84% for linear and SVR models, respectively) is expected to decrease if larger dataset is used, by using historical data, or by acquireing GFR per London areas.
