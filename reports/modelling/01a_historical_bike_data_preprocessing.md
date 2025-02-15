---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.14.6
  kernelspec:
    display_name: pb-buddy
    language: python
    name: python3
---

# Preprocssing of Historical PinkBike Ads - Bike Ads


```python
from concurrent.futures import ThreadPoolExecutor

import matplotlib.pyplot as plt
import pandas as pd
import seaborn as sns
from IPython.display import Markdown

import pb_buddy.data_processors as dt
import pb_buddy.utils as ut
from pb_buddy.scraper import PlaywrightScraper, get_category_list

%load_ext autoreload
%autoreload 2
```

```python
# ------------------- SETTINGS ----------------------
# Where processed data will get written to
path_out = "s3://bike-buddy/data/historical/adjusted/"
file_stub = "_adjusted_bike_ads"
# ----------------------------------------------------
```

```python
# df_historical_sold = dt.stream_parquet_to_dataframe("historical_data.parquet.gzip", "pb-buddy-historical")
```

```python
df_historical_sold = pd.concat(
    [
        pd.read_parquet("s3://bike-buddy/data/historical/raw/historical_sold.parquet.gzip"),
        pd.read_parquet("s3://bike-buddy/data/historical/raw/historical_sold_v2.parquet.gzip"),
    ]
)
```

```python
df_historical_sold.shape
```

```python
df_current_sold = dt.get_dataset(-1, data_type="sold")
```

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.95 Safari/537.36"
}


def run_playwright():
    playwright = PlaywrightScraper()
    category_dict = get_category_list(playwright)
    return category_dict


with ThreadPoolExecutor() as executor:
    future = executor.submit(run_playwright)
    category_dict = future.result()
```

```python
existing_categories = df_historical_sold["category"].unique().tolist() + df_current_sold["category"].unique().tolist()
```

```python
exclude_keywords = ["parts", "frames", "bags", "racks", "stands", "rims", "mx"]

candidate_categories = []
for k in set(list(category_dict.keys()) + list(existing_categories)):
    if "bike" in k.lower() and not any(keyword in k.lower() for keyword in exclude_keywords):
        candidate_categories.append(k)

sorted(candidate_categories)
```

```python
bike_category_labels = candidate_categories
```

```python
# Add in category num for each category, and clean up datatypes.
df_historical_sold_world = (
    pd.concat([df_historical_sold, df_current_sold], sort=False)
    .query("still_for_sale.str.contains('Sold')")
    .assign(
        original_post_date=lambda x: pd.to_datetime(x["original_post_date"]),
        last_repost_date=lambda x: pd.to_datetime(x["last_repost_date"]).astype("datetime64[ns]"),
        category=lambda x: x.category.str.strip(),
        last_repost_year=lambda x: x.last_repost_date.dt.year,
        price=lambda x: x["price"].astype(float),
    )
)
```

```python
(
    df_historical_sold_world.assign(country=lambda _df: _df["location"].str.split(",").str[-1].str.strip())
    .groupby(["country", "currency"])
    .size()
    .sort_values(ascending=False)
    .head(30)
)
```

## Bikes

We'll first check how many ads we have that can be used for modelling bike prices in general. We'll first check counts by type of bike, and then the metadata available for modelling (i.e frame material etc.).

```python
# Check dataset size to model North American Pricing
(
    df_historical_sold_world.groupby(["category"], as_index=False)
    .count()
    .query("category.isin(@bike_category_labels)")[["category", "original_post_date"]]
    .rename(columns={"original_post_date": "count_ads"})
    .sort_values("count_ads", ascending=False)
    .style.hide(axis="index")
    .set_caption("Bike Ads by Category - World Wide Scraped Data")
)
```

```python
# for bikes ads - check how often each field is filled. This metadata couldn't have been entered all along
(
    df_historical_sold_world.query("category.isin(@bike_category_labels)")
    .dropna(axis=1, thresh=100)  # Drop cols never used for bikes
    .notnull()
    .sum()  # Sum TRUES for count
    .sort_values(ascending=False)
    .to_frame("Count Ads with Data For Field")
)
```

It appears we have good data for the standard ad title, description, frame size and wheel size. Otherwise the other columns aren't fully in use otherwise we could use front travel, rear travel more heavily to model mountain bikes more accurately potentially. Material also isn't fully utilized - otherwise this would be another valuable indicator. Finally, condition also isn't fully utilized - this could have potential as another indicator but ~ < 1/3 have this filled out.

```python
df_sold_bikes = df_historical_sold_world.query("category.isin(@bike_category_labels)")
```

Next, we check the quality of key columns for outliers. Here's the findings:

Price - Outliers Removed:
- Numerous ads put at fake high prices ( > 1 Mill.). From looking through ads, $20000 raw price was decided as a threshold. 
- Ads put < $100, asking "Make me an offer" or similar. We don't want to model cheap bikes!
- Ads mention "Stolen" or similar.
- People price ads with $1234 etc. These ads are all removed.

```python
df_sold_bikes_model = (
    df_sold_bikes.query("price < 20000 and price > 100")
    .query("description.str.lower().str.contains('stolen') == False")
    .query("description.str.lower().str.contains('looking for') == False")
    .query("price != 1234 and price != 12345 and price !=123")
    .assign(
        last_repost_month=lambda x: (
            x["last_repost_date"] + pd.offsets.Day(1) + pd.offsets.MonthBegin(-1)
        ).dt.date.astype("datetime64[ns]")
    )
)
```

```python
top_n = 10
top_bike_categories = df_sold_bikes_model.category.value_counts().index[0:top_n].tolist()
(
    df_sold_bikes_model.query("category.isin(@top_bike_categories)")
    .groupby(["last_repost_month", "category"])
    .mean()
    .unstack(1)
    .plot(y="price", figsize=(12, 8), title="Average Prices per Month by Category")
)

# Count of Ads
(
    df_sold_bikes_model.query("category.isin(@top_bike_categories)")
    .groupby(["last_repost_month", "category"])
    .count()
    .unstack(1)
    .plot(y="url", figsize=(12, 8), title="Counts of Sold Ads per Month by Category")
)
```

```python
# Count of Ads
g = (
    df_sold_bikes_model.assign(
        last_repost_month=lambda x: x.last_repost_date.dt.month,
        last_repost_year=lambda x: x.last_repost_date.dt.year,
    )
    # .query("category_num.isin(@top_bike_categories)")
    .groupby(["last_repost_month", "last_repost_year", "category"])
    .count()[["url"]]
    .rename(columns={"url": "count_ads"})
    .reset_index()
    .pipe(
        (sns.relplot, "data"),
        x="last_repost_month",
        y="count_ads",
        col="category",
        hue="last_repost_year",
        col_wrap=3,
        facet_kws={"sharey": False, "sharex": False},
        height=3,
        aspect=1.5,
        # title="Test",
        kind="line",
    )
)
plt.style.use("seaborn")
g.fig.subplots_adjust(top=0.9)
g.fig.suptitle("Count of Sold Ads Over Time")
```



<!-- #region -->
We can see a few interesting trends here:
- All Mountain bikes have a more major bump in volume in the spring, and in the fall.
- All Mountain bikes have increased significantly over the period of the dataset. In 2021, certain months had > 3000 ads that were marked as sold.
- Gravel bikes have increased significantly in the last 2 years going from < 100 ads per month, to over 300 ads per month in 2021.


## Preprocessing

To enable consistent pricing for modelling - we need to adjust for the USD -> CAD conversion, and the inflation over time.

<!-- #endregion -->

### Adjusting for Inflation

As a first attempt - we'll look at "All Items" level inflation to adjust our prices. There is most likely a more relevant index that is available - but this will be good enough for now. We need to find Canadian data and US Data separately.

```python
# Create a mapping of country to which CPI adjustment to use:
# US -> US CPI data
# Canada -> Canada CPI data
# United Kingdom -> UK CPI data
# Rest of Europe -> eu CPI data

country_to_cpi = {
    "United States": "united-states",
    "Canada": "canada",
    "United Kingdom": "uk",
    "Wales": "uk",
    "Ireland": "uk",
    "Northern Ireland": "uk",
    "Scotland": "uk",
    "Germany": "eu",
    "France": "eu",
    "Spain": "eu",
    "Italy": "eu",
    "Netherlands": "eu",
    "Belgium": "eu",
    "Austria": "eu",
    "Portugal": "eu",
    "Greece": "eu",
    "Sweden": "eu",
    "Denmark": "eu",
    "Finland": "eu",
    "Luxembourg": "eu",
    "Slovenia": "eu",
    "Slovakia": "eu",
    "Estonia": "eu",
    "Cyprus": "eu",
    "Malta": "eu",
    "Latvia": "eu",
    "Lithuania": "eu",
    "Czech Republic": "eu",
    "Poland": "eu",
    "Hungary": "eu",
    "Romania": "eu",
    "Bulgaria": "eu",
    "Croatia": "eu",
    "Norway": "eu",
    "Switzerland": "eu",
    "Iceland": "eu",
    "Liechtenstein": "eu",
    "Ukraine": "eu",
}
```

```python
# Build cpi datasets
from pb_buddy.modelling.normalization import CPISourceFactory

cpi_data = []
for region in ["united-states", "canada", "uk", "eu"]:
    cpi = CPISourceFactory().get_source(region).get_cpi_data()
    cpi["region"] = region
    cpi_data.append(cpi)

df_cpi_data = pd.concat(cpi_data)


# Ensure 'year' column is of integer type
df_cpi_data["year"] = df_cpi_data["year"].astype(int)

# Get the current year
current_year = pd.Timestamp.now().year

# Create an index with all combinations of regions and years up to the current year
regions = df_cpi_data["region"].unique()
years = range(df_cpi_data["year"].min(), current_year + 1)
idx = pd.MultiIndex.from_product([regions, years], names=["region", "year"])

# Reindex the DataFrame and forward-fill missing values
df_cpi_data = df_cpi_data.set_index(["region", "year"]).reindex(idx).groupby(level=0).ffill().reset_index()
```

```python
# Join on all CPI data per currency and adjust historical dollars to most recent year

df_sold_bikes_model_adjusted = (
    df_sold_bikes_model.assign(
        country=lambda x: x["location"].str.split(",").str[-1].str.strip(),
        cpi_region=lambda x: x["country"].str.strip().map(country_to_cpi),
    )
    .merge(
        df_cpi_data.drop(columns=["currency"]),
        how="left",
        left_on=["last_repost_year", "cpi_region"],
        right_on=["year", "region"],
    )
    .assign(price_cpi_adjusted=lambda x: (x.price * (x.most_recent_cpi / x.cpi)))
)
```

```python
for region, group in df_sold_bikes_model_adjusted.groupby("cpi_region"):
    (
        group.groupby("last_repost_month")
        .mean(numeric_only=True)[["price", "price_cpi_adjusted"]]
        .plot(figsize=(12, 8), title=f"Inflation Adjusted Average Prices in {region}")
    )
```


### Exchange Rates - USD/EUR/GBP -> CAD

```python
# Use Yahoo Finance API through pandas_datareader
import numpy as np
from pandas_datareader.yahoo.fx import YahooFXReader

fx_reader = YahooFXReader(
    symbols=["USDCAD", "EURCAD", "GBPCAD"],
    start="01-JAN-2010",
    end=df_sold_bikes_model.last_repost_date.max(),
    interval="mo",
)

# Get data and fix symbols - we want to adjust USD prices -> CAD, leaving
# CAD prices untouched. We'll get the data at a monthly level for now.
# There are data issues with multiple returns for some months, and none for other.
# Average across multiple returns for same month, forward fill for missing.
df_fx = (
    fx_reader.read()
    .reset_index()
    .assign(
        fx_month=lambda x: (pd.to_datetime(x.Date) + pd.offsets.Day(1) + pd.offsets.MonthBegin(-1)),
        currency=lambda x: x["PairCode"].str.replace("CAD", ""),
    )
    .rename(columns={"Close": "fx_rate_CAD"})
    .filter(["fx_month", "currency", "fx_rate_CAD", "PairCode"])
    .groupby(["currency", "fx_month"], as_index=False)
    .mean(numeric_only=True)
    .set_index("fx_month")
    .groupby("currency")
    .resample("MS")
    .mean(numeric_only=True)
    .ffill()
    .reset_index()
    .merge(
        df_sold_bikes_model[["last_repost_month", "currency"]]
        .assign(last_repost_month=lambda x: x.last_repost_month.astype("datetime64[ns]"))
        .drop_duplicates(),
        how="right",
        left_on=["fx_month", "currency"],
        right_on=["last_repost_month", "currency"],
    )
    .assign(
        fx_month=lambda x: np.where(x.currency == "CAD", x.last_repost_month, x.fx_month),
        fx_rate_CAD=lambda x: np.where(x.currency == "CAD", 1, x.fx_rate_CAD),
    )
    .sort_values("last_repost_month")
    .groupby("currency", group_keys=False)
    .apply(lambda group: group.ffill())
    .reset_index(drop=True)
    .drop(columns=["last_repost_month"])
)
# df_fx  # .query("currency=='CAD'")
```

```python
## Add on currency converted column and a few helper columns
df_sold_bikes_model_adjusted_CAD = df_sold_bikes_model_adjusted.merge(
    df_fx,
    how="left",
    left_on=["last_repost_month", "currency"],
    right_on=["fx_month", "currency"],
).assign(
    fx_month=lambda x: x.fx_month.where(~x.fx_month.isna(), other=x.last_repost_month),
    price_cpi_adjusted_CAD=lambda x: x.price_cpi_adjusted * x.fx_rate_CAD,
)
```

```python
df_sold_bikes_model_adjusted_CAD.shape
```

```python
(
    df_sold_bikes_model_adjusted_CAD.groupby(["last_repost_month"])
    .mean(numeric_only=True)[["price", "price_cpi_adjusted", "price_cpi_adjusted_CAD"]]
    # .unstack(1)
    .plot(figsize=(12, 8), title="Inflation Adjusted Average Prices")
)
```

```python
# Count of Ads
g = (
    df_sold_bikes_model_adjusted_CAD.assign(
        last_repost_month=lambda x: x.last_repost_date.dt.month,
        last_repost_year=lambda x: x.last_repost_date.dt.year,
        count_in_month=lambda x: x.groupby(["last_repost_month", "last_repost_year"])["url"].transform(len),
    )
    .query("count_in_month > 10")
    .reset_index()
    .pipe(
        (sns.relplot, "data"),
        x="last_repost_month",
        y="price_cpi_adjusted_CAD",
        hue="last_repost_year",
        height=5,
        aspect=2,
        kind="line",
    )
)
g.fig.subplots_adjust(top=0.95)
g.fig.suptitle("Average Adjusted Price by Year - 95% CI Denoted")
```

```python
sns.lineplot(
    data=df_sold_bikes_model_adjusted_CAD.assign(
        original_post_month=lambda _df: _df.original_post_date.dt.to_period("M").dt.to_timestamp()
    ),
    x="original_post_month",
    y="price_cpi_adjusted_CAD",
    estimator="mean",
)
```

```python
(
    df_sold_bikes_model_adjusted_CAD.assign(last_repost_month=lambda x: pd.to_datetime(x.last_repost_month))
    .groupby(["last_repost_month", "currency"])
    .count()[["category"]]
    .reset_index()
    .assign(
        count_in_month=lambda x: x.groupby("last_repost_month")["category"].transform(sum),
        fraction_by_currency=lambda x: (x.category / x.count_in_month),
    )
    .pivot(index="last_repost_month", columns="currency", values="fraction_by_currency")
    .reset_index()
    .plot(
        kind="area",
        x="last_repost_month",
        title="Porportion of Ads in United States vs. Canada",
        ylabel="Porportion",
    )
)
```

Due to people appearing to mark an ad as "Sold" and then reopening it later at a lower price and reselling - we have duplicates on URL to ads in our dataset. To deal with this - we'll take the latest by last repost date.

```python
df_sold_bikes_model_adjusted_CAD = df_sold_bikes_model_adjusted_CAD.assign(
    last_repost_date=lambda x: pd.to_datetime(x.last_repost_date),
    datetime_scraped=lambda x: pd.to_datetime(x.datetime_scraped, utc=True),
    url_rank=lambda x: x.groupby("url")["last_repost_date"].rank("dense", ascending=False),
    scrape_rank=lambda x: x.groupby("url")["datetime_scraped"].rank(
        "dense", ascending=False
    ),  # Sometimes I have multiple scrapes of same sold ad!
).query("url_rank == 1 and scrape_rank == 1")
```

```python
df_sold_bikes_model_adjusted_CAD = df_sold_bikes_model_adjusted_CAD.pipe(
    ut.convert_to_float, colnames=["view_count", "watch_count"]
).pipe(ut.cast_obj_to_string)
```

```python
# Checks!
assert len(df_sold_bikes_model_adjusted_CAD) == len(
    df_sold_bikes_model_adjusted_CAD.drop_duplicates(subset=["url"])
), "Duplicates found on URL!"
```

```python
Markdown(
    f"Shape of modelling dataset for complete bikes, in North America: {len(df_sold_bikes_model_adjusted_CAD)} rows, \n"
    f"{df_sold_bikes_model_adjusted_CAD.shape[1]} columns."
)
```

```python
# # Save out versioned file for modelling
timestamp = pd.Timestamp.now().strftime("%Y-%m-%d_%H_%M_%S")
filename = f"{timestamp}_{file_stub}.parquet.gzip"
df_sold_bikes_model_adjusted_CAD.to_parquet(f"{path_out}{filename}", compression="gzip")
```

```python
Markdown(f"The processed and adjusted data has been written to container: { path_out } with filename: {filename}")
```

```python

```
