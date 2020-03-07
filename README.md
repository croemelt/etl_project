# Technical Report for Performing ETL

* This README is a technical report for performing ETL. The ETL was conducted by Connor Roemelt and Phil Stubbs.
* The technical report is also available here : <https://etl-technical-report.appspot.com/>

## Background

ETL stands for **Extract**, **Transform**, and **Load**. For this project, we went through the ETL process using a couple of different open source datasets. This report includes an overview of the specific steps we took to perform the ETL, information about the technologies we used, and information about what we learned from performing ETL and how we could make the process more efficient and reliable for doing ETLs in the future.

As we quickly found out from performing an ETL for this project, ETL is a long and tedious process. But, we also realized that ETL is an essential process and key starting point for any data science work so that the data is easy to work with and analyze.

## <a name="about_the_datasets"></a> About the datasets

For this ETL, we used a couple of datasets on global trade and financial aid that we thought would be interesting to look at. The focus of this report is not to analyze the actual data (that might be a future project). The focus of this report is on the ETL process; however, the following table gives a brief overview of the datasets we pulled for this project and where you can go to learn more about these datasets.

* [Global Commodity Trade Statistics](https://www.kaggle.com/unitednations/global-commodity-trade-statistics)
  * This is an open source dataset available on [kaggle](https://www.kaggle.com/). It was originally published by the United Nations Statistic Division on the [UNData](http://data.un.org/Explorer.aspx) website. The dataset includes country trade statistics for exports and imports over the past 30 years, such as year in which the trade took place, a description of the commodity being traded, the flow of trade (i.e., export or import), the currency value of the trade, etc. The format of this dataset is csv.
* [Nepal - Financial Aid data for earthquake as of June 9,2015](https://data.world/opennepal/6ed8dcfe-7877-4811-9353-6d8a8d4ecf5f)
  * This is an open source dataset available on [data.world](https://data.world/).  This dataset includes information on foreign and national aid provided by different organizations to help with the relief efforts of the [April 2015 Nepal earthquake](https://en.wikipedia.org/wiki/April_2015_Nepal_earthquake). The dataset includes information such as the country where the aid originated from, the donor organization, the type of aid (for example, cash), the country receiving the aid, the currency amount of the aid, etc. The format of this dataset is csv.

## Modeling the data with an entity relationship diagram (ERD)

The first step we took in performing the ETL was inspect the csv files and model the data. To model the data, we created an entity relationship diagram (ERD). The ERD allows us to quickly visualize how the different entities in the dataset are related as well as define the entity types (string, integer, etc.). Creating an ERD especially helped with understanding how large datasets like the ones we used can be further broken down into smaller datasets/tables that are easier to analyze, read, and work with. Specifically, one of the challenges we faced going through this process was working with large datasets that had millions of rows and had more columns of data than we needed. The first step in solving that challenge of working with large datasets was creating an ERD. To create the ERD, we used [Quick DBD](https://www.quickdatabasediagrams.com/).

![Image of ERD](./erd/erd.png)

## Extract

The first step in our ETL process included extracting data, which involved retrieving raw, uncleaned data from multiple sources.

### Storing the csv files into Pandas dataframes using ```read_hdf```

After creating a simple conceptual model of the data we were working with, we started the first step in the ETL process, extracting the data. As mentioned in the <a href="#about_the_datasets">About the datasets</a> section, we started this project with two csv files. Because of the large size of these files, working with these files in Microsoft Excel or another similar tool was not an option. So, we decided to use Pandas. As a result, we converted the csv files into Pandas dataframes. To store a csv file into a Pandas dataframe, the normal way of doing that is using a Pandas method called ```read_csv```. However, one of our datasets had over a million rows and was too slow to load/write with using ```read_csv```. Having to wait for the csv to load into a a Pandas dataframe every time we ran our jupyter notebook file using ```read_csv``` was slow and painful. After doing some research, [this](https://twitter.com/python_tip/status/1087139487816302592?lang=en) appears to be a common problem when working with large datasets. So, to improve performance of loading and working with the large csv files, we used a Pandas method called ```read_hdf```, which (although similar) seems to have a huge performance advantage over <code>read_csv</code>. For more information on the performance of ```read_hdf```, check out [this article](https://medium.com/@bobhaffner/gist-to-medium-test-db3d51b8ba7b).

## Transform

The next step in our ETL process is transforming data, which involved ensuring that the data is clean, the format of the data is consistent, and the data is structured in such a way that it is ready to put in a database as well as retrieve it from the database later on to perform queries.

To transform the data, we used two primary tools, Pandas and SQLAlchemy. Transforming the data was the most difficult and most time consuming part of this process. But, this was the most important step because it ensured that all the data was in a consistent format, was broken down into small, manageable chunks, and was in a structure that could be stored in a database for making efficient queries and analyzing the data.

### Using consistent naming format for column names

After loading the csv files into Pandas dataframes, we noticed that the column names were not in any particular consistent format and were not consistent among the two datasets. To make the datasets easier to work with and load into the database, we defined a specific format for all column names because these are the column names that will be used to create the database tables. For consistency, we renamed all columns for both datasets to be lower case. And, if a column name was more than one word in length, than those words would be separated with an underscore character.

### Dropping null values from dataframes

There were quite a few values in the datasets that were null. To make the loading of the data into the final production database easier, we decided that the data shouldn't contain null values. As a part of the cleaning process, we removed all null values from the dataframes before being inserted into the database.

### Adding id columns to the dataframes

One critical column that the original datasets did not have was an id column or some type of column to uniquely identify a row in the dataframe. As a result, for each dataframe, we added an auto-incrementing id column. By adding this column, we are now able to easily reference different values within a table, but more important, we are able to define foreign key relationships across different tables.

### Breaking down the data into smaller, manageable chunks for better performance

The biggest hurdle we faced in the transformation process was working with large datasets. Specifically,the [Global Commodity Trade Statistics](https://www.kaggle.com/unitednations/global-commodity-trade-statistics) dataset includes millions of rows, which is unacceptable to go into the database as this form. Working with this dataframe as is resulted in performance problems when trying to read it into pandas as well as when trying to load it into the database. When we tried to load the original dataframe into the database as is, the loading and database queries were very slow, and the data was almost impossible to work with. So, as a result, we decided to break this specific dataframe into smaller, manageable chunks. As a first step, we removed all information that could be potentially duplicated into separate, small dataframes. For example, each commodity in the ```commodity_export``` table is associated with a specific category. Because a category can have many commodities, the category column contains duplicate values. As a result, we stripped out the duplicates in the category column and moved this column to its own dataframe. As shown in the ERD diagram above, the ```commodity_category``` dataframe, the ```commodity_exports``` dataframe, and the ```commodity_imports``` dataframe are related by the ```category_id``` field in each dataframe. Similar to what we did for category, we went through the same process for the ```commodity_code``` dataframe. By moving category and commodity code values to new, separate dataframes, we not only removed duplicate information from the original dataset and now have a single source of truth, but we also gained significant performance improvements for working with the data inside the jupyter notebook file as well as inside the database.

### Standardizing and cleaning country names

One thing that makes data really nice and easy to work with is consistency. So, the next focus point within the transformation phase was standardizing the list of country names. Within each dataset, there is a country name, which allows us to eventually join the two datasets into one to be able to do analysis by country. However, because of the inconsistent naming of countries, it is impossible to join the two without standardizing and cleaning the list of countries first. For example, one dataset had a spelling of "Vietnam" and the other had a spelling of "Viet Nam." To standardize the countries, we extracted just the country names from the Global Commodity Trade Statistics dataset into a new, separate dataframe. We then extracted the country names from the Financial Aid dataset into a new, separate dataframe. We then stripped the duplicate coutries from each of the new country dataframes, compared the country names, and when the country names were inconsistent, we renamed the country names so that they were the same. This was a bit of manual process of having to go through the list of country names one by one as we could not think of a better way of doing this. But, by doing this, we have a single source of truth of country names between the two datasets, and there are no duplicate countries. In the end, this work allowed us to insert a standardized, consistent list of countries into the database.

## Load

The final step in our ETL process is loading data, which involved taking the final data from the transform step and putting it into a production database where that data is easily accessible for querying and analysis. For our final database, we decided to use a SQL database. Specifically, we are using a PostgresSQL database. We chose to use a SQL database for this project because the data is very structured, and there are many relationships between the different tables that would not be possible with a non relational database.

### Using SQLAlchemy

While transforming the data involved working with Pandas to manipulate the datasets, loading the data involved working with SQLAlchemy. We chose to use SQLAlchemy for loading the data because with SQLAlchemy, we were able to define the schemas, write queries, and manipulate the database all through Python. As a result, we were able to avoid writing any raw sql queries. So, if we were to switch from PostgresSQL to a different SQL database or to a non relational database, that migration would be smooth because we would not have to make updates to the queries.

### Defining a schema

Before we can load the final data into the database, we need to define a schema, which defines what the structure of the database will be (tables, columns, relationships, etc.). We already did this when we created the ERD, but we also need to do this programmatically. We could use raw sql to create the schema, but for consistency and to future proof this, we decided to defined the schema of the database using SQLAlchemy. To define the schema, we created Python classes where each class represents a different table within the database. Within each class, we defined the column names (these names have to be the same as the column names in the final dataframes), primary keys, column types, and foreign key relationships.

Here is an example of the class that is associated with the ```commodity_exports``` table in the database:

```bash
class CommodityExports(Base):
  __tablename__ = COMMODITY_EXPORTS_TABLE
  id = Column(Integer, primary_key=True, nullable=False, unique=True)
  year = Column(Integer, nullable=False)
  comodity_code = Column(String(255), ForeignKey(f"{COMMODITY_CODE_TABLE}.commodity_code"), nullable=False)
  trade_flow = Column(String(255), nullable=False)
  trade_value_usd = Column(String(255), nullable=False)
  weight_kg = Column(Float, nullable=False)
  quantity_name = Column(String(255), nullable=False)
  quantity = Column(Float, nullable=False)
  category_id = Column(String(255), ForeignKey(f"{COMMODITY_CATEGORY_TABLE}.category_id"), nullable=False)
  country_id = Column(Integer, ForeignKey(f"{COUNTRY_TABLE}.id"), nullable=False)
```

### Using d6tstack to load the final Pandas dataframes into the database

We are almost to the end of our ETL process. As you can see, there were many steps to get the raw data all the way into a database. This final step was challenging due to running into some database performance issues. Specifically, we first tried using Pandas' ```to_sql``` for loading the dataframes into the database. For small datasets, this works fine. But, for large datasets like the ones we were working with, loading data using this method is slow. To get around this, we used a python package called [d6tstack](https://github.com/d6t/d6tstack), which significantly helped solve performance problems for loading large files. Within the jupyter notebook file, we recorded the actual time it took to load the dataframes using d6tstack.

Using d6tstack was pretty similar to using Pandas' ```to_sql```. To install the d6tstack package, we needed to run the following command in the virtual environment:

```bash
 pip install d6tstack
 ```

### Confirming that the data has been added to the database

As a final step, we confirmed that the data was actually loaded into the database correctly without errors and loaded into the database the way we wanted it based on the schema we defined. For example, we should be able to run the following SQLAlchemy query to get the list of commodity exports:

```bash
exports_list = session.query(CommodityExports).limit(10)
  for commodity in exports_list:
      print(f"commodity: {commodity.id}, trade flow: {commodity.trade_flow}")
```

## Future considerations and notes on database performance

After performing this ETL, we definitely learned a lot about the ETL process and how time consuming it can be. But, at the same time, we learned how valuable going through this process really is to be able to get raw data to a point where the data is consistent, reliable, clean, and fast for making queries and doing analysis. Using tools like Pandas, SQLAlchemy, and d6tstack, definitely helped make ETL in Python easier.

One of the biggest takeaways for us after going through this process was thinking about database peformance and working with large, raw datasets. We were able to get around some of the performance issues by using certain tools over others and by breaking the raw data up into smaller, more manageable chunks. But, after going through the ETL, if we were to do something like this again, we think a non relational database like MongoDB would have been more reliable and faster for handling large datasets than using SQL.
