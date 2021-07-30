---
title: 'Investigating Global COVID-19 Vaccinations'
date: 2021-02-23
draft: false
tags:
  - data science
  - data analytics
  - covid-19
---

# Introduction

I'm curious to see how vaccination rates compare between countries and within countries (specifically the United States, India, and Israel).

Israel is the country with the fastest vaccination rate, already surpassing 30% of the total population vaccinated. I'm curious to see how this compares to the other local Arab states (I know the UAE is in second in terms of vaccination rate), and how vaccination rates among Palestinians compare, if that data is available.

The United States is of course a personal interest to investigate, as I live here, but it also underwent a presidential election and transfer of power during the pandemic, so it'll be interesting to see how vaccination rates compare before and after inauguration and how they compare across the country. Throughout the pandemic I've been following [Dan Goodspeed's COVID-19 charts](https://dangoodspeed.com/covid/total-cases-since-june), I especially like the chart which compares COVID-19 rates per capita between states and colors them by partisanship (red or blue). Thinking a similar idea here, but instead comparing vaccination rates.

India is another personal interest to investigate, but similar to how the USA underwent a political upheaval (change in leadership, attempted insurrection, etc.) India is going through a period of major social/economic upheaval as well through the farmers protests. Here I am more interested in investigating how the protests have impacted the incidence of COVID-19 and vaccination rates. So this will just be a graph of cumulative vaccinations (or rates per week) with points labeled with major events in the protests.

## Datasets

For this investigation I'm using the Our World In Data COVID-19 dataset and their COVID-19 vaccination-specific dataset, which is large (17MB) and probably a bit in excess of what I need. I'm going to start by loading in the entire thing, but take a focused look at the `vaccinations` dataset.


```python
!curl -o ./data/covid19_data.csv https://covid.ourworldindata.org/data/owid-covid-data.csv
!curl -o ./data/covid19_vaccinationdata.csv https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/vaccinations/vaccinations.csv

# Let's fetch the WorldBank population dataset
!curl -o country_pop 'http://api.worldbank.org/V2/country/all/indicators/SP.POP.TOTL?downloadformat=csv'
!unzip country_pop
!ls
# trim first four metadata lines (seriously why are these lines even here?)
!tail -n +5 "API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv" > "./data/country_pop.csv"
!rm country_pop API_SP* Metadata*
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 17.4M    0 17.4M    0     0  44.0M      0 --:--:-- --:--:-- --:--:-- 43.9M
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  259k  100  259k    0     0  2545k      0 --:--:-- --:--:-- --:--:-- 2545k
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 80069  100 80069    0     0   111k      0 --:--:-- --:--:-- --:--:--  111k
    Archive:  country_pop
      inflating: Metadata_Indicator_API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv  
      inflating: API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv  
      inflating: Metadata_Country_API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv  
     API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv
     COVID19Vaccinations.ipynb
    'Global Debt Database.dta'
     Metadata_Country_API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv
     Metadata_Indicator_API_SP.POP.TOTL_DS2_EN_csv_v2_2064197.csv
     country_pop
     country_pop.csv
     covid19_data.csv
     covid19_vaccinationdata.csv
     daily_full_vaccinations_per_hundred.png
     daily_vaccinations.png
     daily_vaccinations_per_million.png
     data
     global_debt.csv
     graphs
     israel_neighbors_daily_doses.png
     israel_neighbors_daily_doses_per_capita.png


That cURL request just fetched two CSV files, one for the general dataset and one for the vaccination specific dataset. This cell should be rerun every day as that's how frequently Our World in Data updates their dataset. Now we can look into the set and see what we want. As a preliminary chart I'm going to make a line chart of weekly vaccination rates by country (top ten will be graphed with distinct colors) starting in December. So the y-axis will be number of people fully (two doses) vaccinated per 100 people, and the x-axis will be dates (week index, with 0 being first week of December).

## Exploring the Data


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from datetime import datetime, date
import math

import urllib
import os
import csv
```


```python
df = pd.read_csv("covid19_vaccinationdata.csv")
pop_df = pd.read_csv("country_pop.csv")
# trim
vaccine_df = df[["location", "iso_code", "date",  "total_vaccinations", "daily_vaccinations",
                 "daily_vaccinations_raw", "people_fully_vaccinated",  "daily_vaccinations_per_million",
                 "people_fully_vaccinated_per_hundred"]]

# I'm calculating the per capita rate for daily doses administered myself
# (daily_vaccinations_per_million in the Our World in Data dataset). This is because I'll be using the
# World Bank dataset later which draws it's population counts from a more complete set of sources
# and I want to be consistent about what population values I'm using. I will fill in the NaN calculated values with
# per capita values from OWiD's dataset.
year = 2020
while year >= 1960:
  pop_df["2020"].fillna(inplace=True, value=pop_df[str(year)])
  year -= 1
pop_df = pop_df[["Country Name", "2020"]]

pop_df.rename(inplace=True, columns={"2020": "latest_available_population"})
pop_df.dropna(inplace=True, how="any", axis="index")
vaccine_df = pd.merge(vaccine_df, pop_df, left_on=["location"], right_on=["Country Name"], how="outer")
vaccine_df = vaccine_df.assign(daily_vaccinations_per_million_calculated = lambda x:
                               x["daily_vaccinations"] / (x["latest_available_population"] / 1000000))
vaccine_df["daily_vaccinations_per_million_calculated"].fillna(inplace=True, value=vaccine_df["daily_vaccinations_per_million"])
vaccine_df.drop(inplace=True, columns=["daily_vaccinations_per_million"])
vaccine_df.rename(inplace=True, columns={
  "daily_vaccinations_per_million_calculated":"daily_vaccinations_per_million"
})
```

So we have metrics for daily vaccinations, and this is provided in a number of ways:
- `daily_vaccinations_raw`: only calculated when there is data on total vaccinations for consecutive days.
- `daily_vaccinations`: calculated for every day
  1. Fill gaps in data for `total_vaccinations` assuming a perfectly linear progression
  2. Calculate the daily vaccinations as the difference in the interpolated totals day by day
  3. The smoothed daily vaccinations are calculated by taking an average of the previous 7 days as calculated in Step 2, or fewer if 7 data points are not available.
  - Further description provided [here](https://github.com/owid/covid-19-data/issues/333#issuecomment-763015298).
- `daily_vaccinations_per_million`: calculated for every day as `daily_vaccinations` / (country population in millions)


As a starting point let's get the average daily vaccination rate by taking the `total_vaccinations` for each country and dividing by the number of days since the first day they reported vaccination numbers.



```python
def get_daily_average(df, doses=True):
  # doses is a boolean flag to indicate whether to measure the average as a fraction of individual vaccine doses administered
  #   or of people fully vaccinated
  cumulative_val = "total_vaccinations" if doses else "people_fully_vaccinated"

  # Get total vaccinations for all countries and drop null vals by row
  tv_raw = df[["location", cumulative_val]].copy()
  tv_raw.dropna(inplace=True, axis="index", how="any")

  countries = tv_raw.location.unique()

  # Create a dictionary where keys are countries and values are average daily vaccinations
  dv_dict = {}
  for country in countries:
    tv_country = tv_raw[tv_raw["location"] == country]
    total_vac = tv_country[cumulative_val].iloc[-1]
    # Assume first date with any value (even 0 or NaN) provided is the day vaccinations began in that country
    first_vac_date = df[df["location"] == country]["date"].iloc[0]
    fv_date_obj = datetime.strptime(first_vac_date, '%Y-%m-%d')
    today = datetime.today().date()
    delta = today - fv_date_obj.date()

    dv_dict[country] = total_vac / delta.days

  return dv_dict

# Print the dictionary sorted by average daily doses administered
dv_doses = get_daily_average(vaccine_df)
print(sorted(dv_doses.items(), key=lambda item: item[1], reverse=True)[:15])

```

    [('World', 2856593.6097560977), ('United States', 1006486.6285714286), ('China', 540266.6666666666), ('European Union', 386668.6219512195), ('India', 312657.7272727273), ('United Kingdom', 258618.0779220779), ('England', 217997.93506493507), ('Brazil', 188413.6511627907), ('Turkey', 180408.80434782608), ('Israel', 112081.50704225352), ('Morocco', 110461.12903225806), ('United Arab Emirates', 109875.9074074074), ('Germany', 93818.04761904762), ('Bangladesh', 86392.12121212122), ('France', 68231.31746031746)]


So that's some heartening news given that I live in the US. I should've dropped all rows that pertain to the "World" at large, but it's useful to have that as a sort of baseline metric (divide by number of countries, 101, to get the average countries average daily vaccination rate for a country).

The above is a function which creates a dictionary associating each country with the average daily number of vaccinations administered, measured as either the number of individual doses administered or as the number of people fully vaccinated.

## Graphing to Compare Across a Subset of Countries

Now let's write a function that allows us to graph the performance of multiple countries based on different vaccination rate metrics (i.e. raw doses adminsitered, doses administered per million, or people fully vaccinated--2 doses--per hundred). We'll start by comparing the per capita performance of the leading ten countries when we measured average number of doses administered per day. We assume the larger countries by population, like China, India, and the US, should have higher raw total doses administered, since they just have more people and a greater demand. It will be interesting to see how these massive countries compare when normalized for population.


```python
top_ten_items = sorted(dv_doses.items(), key=lambda item: item[1], reverse=True)[1:11]
top_ten_countries = []
for item in top_ten_items:
  top_ten_countries.append(item[0])

def plot_combined_linegraph(countries_arr, metric, ylabel, title, filename):
  # helper function to plot a combined linegraph of multiple countries by some metric in the vaccine_df
  #   input is an array of country names
  dv_df = vaccine_df[["location", "date", metric]].copy()
  dv_df.dropna(inplace=True, axis="index", how="any")

  fig, ax = plt.subplots(figsize=(20,10))

  for country in countries_arr:
    country_df = dv_df[dv_df["location"] == country]
    dates = country_df["date"].values
    x = [datetime.strptime(d, '%Y-%m-%d').date() for d in dates]
    y = country_df[metric].values
    plt.plot(x, y, label=country)

  # Let's configure pyplot properly

  formatter = mdates.DateFormatter("%Y-%m-%d")
  ax.xaxis.set_major_formatter(formatter)
  locator = mdates.DayLocator()
  ax.xaxis.set_major_locator(locator)
  plt.xticks(rotation=45, ha="right")

  plt.xlabel("Dates")
  plt.ylabel(ylabel)
  plt.title(title)

  # Now display and download the graph

  plt.legend()
  plt.show()

  full_path = "./graphs/" + filename

  fig.savefig(full_path, bbox_inches='tight')

plot_combined_linegraph(top_ten_countries, metric="daily_vaccinations",
                       ylabel="Number of Vaccine Doses Administered Daily",
                       title="Daily Vaccination Rate for Leading Countries",
                       filename="daily_vaccinations.png")
plot_combined_linegraph(top_ten_countries, metric="daily_vaccinations_per_million",
                       ylabel="Number of Vaccine Doses Administered Daily per Million People",
                       title="Daily Per Capita Vaccination Rate for Leading Countries",
                       filename="daily_vaccinations_per_million.png")
plot_combined_linegraph(top_ten_countries, metric="people_fully_vaccinated_per_hundred",
                       ylabel="Number of People Fully Vaccinated Daily per Hundred People",
                       title="Full Vaccination Rate per Hundred People for Leading Countries",
                       filename="daily_full_vaccinations_per_hundred.png")

```



![png](../COVID19Vaccinations_files/COVID19Vaccinations_11_0.png)





![png](../COVID19Vaccinations_files/COVID19Vaccinations_11_1.png)





![png](../COVID19Vaccinations_files/COVID19Vaccinations_11_2.png)



Just to clarify, the countries being graphed above are the top ten when measured by the average number of doses administered in a day. When we compare the performance of countries in terms of how many vaccines they're able to administer we should always normalize for population. Larger countries by population, like the US or India, should ideally have more infrastructure and manpower available to deliver the vaccine at scale. What we end up seeing, however, is that the countries which were originally leaders in average number of doses delivered per day aren't actually delivering vaccines as efficiently as countries with smaller populations, and therefore lower raw doses delivered counts.

Israel is a small country by population when compared against the other countries in the top ten, but it is a leader in both raw counts of doses delivered per day and adjusted per capita rates. There's a couple explanations for this: the difference between how robust and efficient Israel's infrastructure is and it's size by population is huge and/or the infrastructure of large pop countries like India and the US is inefficient relative to their size. The latter is illustrated by the graph above, larger population countries don't compare well against smaller countries when measured using per capita rates. The former we can measure by graphing Israel against countries of a similar size by population (the k-nearest neighbors).


```python
# take Israel and it's top ten nearest neighbors by population for the latest year it was reported for Israel (2019)
israel_pop = pop_df[pop_df["Country Name"] == "Israel"]["latest_available_population"].item()
israel_neighbors = pop_df.iloc[(pop_df["latest_available_population"]-israel_pop).abs().argsort()[:11]]["Country Name"].to_numpy()
# def plot_combined_linegraph(countries_arr, metric, ylabel, title, filename):
plot_combined_linegraph(israel_neighbors, metric="daily_vaccinations",
                        ylabel="Number of Doses Administered",
                        title="Daily Doses Administered per Day for Israel and it's 10 Nearest Neighbors by Population",
                        filename="israel_neighbors_daily_doses.png")

plot_combined_linegraph(israel_neighbors, metric="daily_vaccinations_per_million",
                        ylabel="Number of Doses Administered per Million Citizens",
                        title="Daily Doses Administered per Capita for Israel and it's 10 Nearest Neighbors by Population",
                        filename="israel_neighbors_daily_doses_per_capita.png")
```



![png](../COVID19Vaccinations_files/COVID19Vaccinations_13_0.png)





![png](../COVID19Vaccinations_files/COVID19Vaccinations_13_1.png)



I'll admit I did miss the fact that the UAE was also on both lists and is certainly a similar size by population as Israel. It does seem, however, that Israel and the UAE are far and away from their population neighbors in terms of vaccinations administered per day. It would seem that for their populations they certainly boast either more access to vaccine manufacturers/distributors, or the infrastructure to manufacture vaccines themselves.

# Expanding the Dataset

To get a better idea of what factors impact the ability for a country to administer more vaccines per capita I need to compare vaccine distribution rates by other metrics pertaining to economic factors like GDP, government spend and national debt, and healthcare factors like healthcare spending, size in number of doctors and hospitals and ventilators and ICU capacity.

I'm going to load in a couple more datasets which I'll need to join by some common key. Country code seems like a good bet, and an inner join so we only take countries tracked in both datasets.

I'll make a couple graphs to make sure the dataset is of good quality, then I can run some principal component analysis to see which features have the greatest correlation with daily_vaccinations and people_fully_vaccinated. I can then use this to generate a smaller dataset of principal features, split into a testing and training set and start projecting daily_vaccinations and people_fully_vaccinated and calculate which country is most likely to reach herd immunity within their borders first.

It's obvious that I need to include much more historical data to train this model on, so I'll have to find data pertaining to previous government-led vaccine administration programs.


```python
# Load in the global national debt dataset and pd.read_excel
!rm *.dta
!curl -o global_debt_stata.zip https://www.imf.org/external/datamapper/GDD/GlobalDebtDatabase.zip
!unzip global_debt_stata.zip
!rm *.zip
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  517k  100  517k    0     0   239k      0  0:00:02  0:00:02 --:--:--  239k
    Archive:  global_debt_stata.zip
      inflating: Global Debt Database.dta  



```python
debt_df = pd.read_stata("Global Debt Database.dta")
print(debt_df.columns)
print(debt_df.head())
debt_df.to_csv("./data/global_debt.csv")


```

    Index(['ifscode', 'country', 'year', 'pvd_all', 'pvd_ls', 'hh_all', 'hh_ls',
           'nfc_all', 'nfc_ls', 'ps', 'nfps', 'gg', 'cg', 'ngdp'],
          dtype='object')
       ifscode        country    year    pvd_all     pvd_ls     hh_all      hh_ls  \
    0    111.0  United States  1950.0  75.826082  55.314634  25.482242  24.802614   
    1    111.0  United States  1951.0  72.692564  53.657228  24.518253  23.911650   
    2    111.0  United States  1952.0  74.282425  56.555505  26.598532  25.976296   
    3    111.0  United States  1953.0  75.037319  58.016532  28.427253  27.794074   
    4    111.0  United States  1954.0  79.868981  62.573558  31.539677  30.862721   

         nfc_all     nfc_ls  ps  nfps         gg         cg        ngdp  
    0  50.343839  30.512020 NaN   NaN  83.126037  78.199645  301.782705  
    1  48.174311  29.745577 NaN   NaN  72.955032  68.201929  348.993057  
    2  47.683893  30.579209 NaN   NaN  72.525492  66.341449  368.027836  
    3  46.610066  30.222458 NaN   NaN  71.753219  64.602206  389.147698  
    4  48.329304  31.710837 NaN   NaN  73.568066  64.955715  390.276672  


# Notes

Ideally the place I'd like to get to with this is to incorporate some other datasets into this, calculate my own smoothed daily vaccination counts, then project future daily vaccination counts and try to predict which country will hit herd immunity first. I can then check my prediction against other peoples results and shop my projection model. I need to consider several regression models to use here, and do some dimensionality reduction.

I also want to set up my own runtime environment in GCP with a Jupyter notebook frontend.

# Appendix

Include code snippets and function definitions here. Have some basic documentation for helper functions. Ideally helper functions should all be moved into a utils.py modules file which is then imported in along with all the other packages.
