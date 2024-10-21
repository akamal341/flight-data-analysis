# Flight Data Analysis - Department of Transportation

## Project Overview

This project involves analyzing a substantial flight data set from the Department of Transportation for the year 2015. The dataset includes nearly 300,000 flight records, along with detailed information on delays, flight times, airlines, and airports. The analysis covers various data manipulation and analytical tasks to uncover insights about flight performance and delays.

## Table of Contents
1. [Project Description](#project-description)
2. [Data Sources](#data-sources)
3. [Dataset Size](#dataset-size)
4. [Functionality](#functionality)
    - [Loading and Cleaning Data](#loading-and-cleaning-data)
    - [Flight Counts per Airport](#flight-counts-per-airport)
    - [Top Airlines by Number of Flights and Delay Percentage](#top-airlines-by-number-of-flights-and-delay-percentage)
    - [Monthly Percentage of Delays per Airport](#monthly-percentage-of-delays-per-airport)
5. [Setup Instructions](#setup-instructions)
6. [Key Results](#key-results)
7. [Contact Information](#contact-information)

---

## Project Description

This project involves the following steps:
1. **Data Loading and Cleaning:** Load and preprocess flight data to remove missing values, filter relevant airports, and handle date-time values.
2. **Flight Counts per Airport:** Calculate the number of departing flights per airport.
3. **Top Airlines by Number of Flights and Delay Percentage:** Identify top airlines based on number of flights and delay percentages.
4. **Monthly Percentage of Delays per Airport:** Calculate the monthly percentage of delays for each chosen airport.

## Data Sources

The dataset consists of flight data for the year 2015, provided by the Department of Transportation. The data includes:
- Flight details
- Airport information
- Airline information

## Dataset Size

The primary flight dataset contains nearly 300,000 flight records, making it a large and comprehensive dataset suitable for detailed analysis and insights.

## Functionality

### 1. Loading and Cleaning Data

The `data_preprocess` function performs the initial data cleaning and preprocessing steps, including filtering specific airports, handling delays, and converting data types.

```python
import pandas as pd
import numpy as np
import warnings

warnings.filterwarnings('ignore')

# Load data
flights_df_raw = pd.read_csv('C:/Users/akama/Downloads/flights.csv', low_memory=False)
airports_df = pd.read_csv('C:/Users/akama/Downloads/airports.csv')
airlines_df = pd.read_csv('C:/Users/akama/Downloads/airlines.csv')

# Data Preprocessing Function
def data_preprocess(flights_df):
    # Remove rows with missing values
    flights_df = flights_df.dropna()

    # Filter by specific airports
    airports = ['BOS', 'JFK', 'SFO', 'LAX']
    flights_df = flights_df[flights_df['ORIGIN_AIRPORT'].isin(airports)]

    # Filter out flights with more than 1 day delay
    flights_df = flights_df[flights_df['DEPARTURE_DELAY'] <= 1440]

    # Convert FLIGHT_NUMBER to string
    flights_df['FLIGHT_NUMBER'] = flights_df['FLIGHT_NUMBER'].astype(str)

    # Convert SCHEDULED_DEPARTURE to datetime
    flights_df['SCHEDULED_DEPARTURE'] = pd.to_datetime(
        flights_df[['YEAR', 'MONTH', 'DAY']].astype(str).agg('-'.join, axis=1) +
        ' ' +
        (flights_df['SCHEDULED_DEPARTURE'] // 100).astype(int).astype(str).str.zfill(2) +
        ':' +
        (flights_df['SCHEDULED_DEPARTURE'] % 100).astype(int).astype(str).str.zfill(2)
    )

    # Add IS_DELAYED column
    flights_df['IS_DELAYED'] = flights_df['DEPARTURE_DELAY'] >= 15

    # Drop unnecessary columns
    flights_df.drop(['YEAR', 'MONTH', 'DAY'], axis=1, inplace=True)

    return flights_df

# Preprocess the flight data
flights_df = data_preprocess(flights_df_raw.copy())
print(f"DataFrame Shape after cleaning: {flights_df.shape}")
```

### 2. Flight Counts per Airport

The `flights_per_airport` function merges the flight data with airport information and calculates the number of departing flights per airport.

```python
# Calculate Number of Flights per Airport
def flights_per_airport(flights_df, airports_df):
    flights_df = data_preprocess(flights_df)

    # Merge with airports data
    merged_df = flights_df.merge(airports_df, left_on='ORIGIN_AIRPORT', right_on='IATA_CODE', how='inner')

    # Calculate the number of flights per airport
    num_flights_df = merged_df.groupby('IATA_CODE').size().reset_index(name='NUM_FLIGHTS')

    # Reindex to get counts for specific airports
    num_flights_df.set_index('IATA_CODE', inplace=True)
    num_flights_df = num_flights_df.reindex(['BOS', 'JFK', 'SFO', 'LAX']).reset_index()
    num_flights_df = num_flights_df.set_index(num_flights_df.columns[0])

    return num_flights_df

# Get the number of flights per airport
num_flights_df = flights_per_airport(flights_df_raw.copy(), airports_df.copy())
print(f"Number of flights per airport:\n{num_flights_df}")
```

### 3. Top Airlines by Number of Flights and Delay Percentage

The `top_three_airlines` function identifies the top three airlines based on the number of flights and the least percentage of delays.

```python
# Identify Top Three Airlines by Number of Flights and Delay Percentage
def top_three_airlines(flights_df, airlines_df):
    flights_df = data_preprocess(flights_df)

    # Calculate number of flights per airline
    num_flights = flights_df.groupby('AIRLINE')['FLIGHT_NUMBER'].count().reset_index()
    num_flights.columns = ['AIRLINE', 'NUM_FLIGHTS']

    # Calculate number of delayed flights per airline
    num_delayed = flights_df[flights_df['IS_DELAYED']].groupby('AIRLINE')['FLIGHT_NUMBER'].count().reset_index()
    num_delayed.columns = ['AIRLINE', 'NUM_DELAYED']

    # Merge the counts
    merged_df = num_flights.merge(num_delayed, on='AIRLINE', how='left')
    merged_df['NUM_DELAYED'] = merged_df['NUM_DELAYED'].fillna(0)

    # Calculate percentage of delays
    merged_df['PERC_DELAY'] = (merged_df['NUM_DELAYED'] / merged_df['NUM_FLIGHTS']) * 100

    # Sort and get top three airlines
    sorted_df = merged_df.sort_values(['NUM_FLIGHTS', 'PERC_DELAY'], ascending=[False, True])
    top_three_airlines_df = sorted_df.head(3)

    # Merge to get airline names
    top_three_airlines_df = top_three_airlines_df.merge(airlines_df, left_on='AIRLINE', right_on='IATA_CODE', how='left')
    top_three_airlines_df = top_three_airlines_df[['AIRLINE_y', 'NUM_FLIGHTS', 'PERC_DELAY']]
    top_three_airlines_df.columns = ['AIRLINE_NAME', 'NUM_FLIGHTS', 'PERC_DELAY']

    return top_three_airlines_df

# Get top three airlines
top_three_airlines_df = top_three_airlines(flights_df_raw.copy(), airlines_df.copy())
print(f"Top three airlines:\n{top_three_airlines_df}")
```

### 4. Monthly Percentage of Delays per Airport

The `monthly_airport_delays` function calculates the monthly percentage of delays for each chosen airport.

```python
# Calculate Monthly Percentage of Delays per Airport
def monthly_airport_delays(flights_df):
    flights_df = data_preprocess(flights_df)
    flights_df['SCHEDULED_DEPARTURE'] = pd.to_datetime(flights_df['SCHEDULED_DEPARTURE'])
    flights_df['MONTH'] = flights_df['SCHEDULED_DEPARTURE'].dt.strftime('%B')
    
    # Calculate mean delay for each origin airport and month
    monthly_delays = flights_df.groupby(['ORIGIN_AIRPORT', 'MONTH'])['IS_DELAYED'].mean().round(4).reset_index()
    monthly_delays_pivot = monthly_delays.pivot(index='MONTH', columns='ORIGIN_AIRPORT', values='IS_DELAYED')
    monthly_delays_df = monthly_delays_pivot.reset_index()

    # Sort months in the correct order
    month_order = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 
                   'October', 'November', 'December']
    
    monthly_delays_df['MONTH'] = pd.Categorical(monthly_delays_df['MONTH'], categories=month_order, ordered=True)
    monthly_delays_df.sort_values('MONTH', inplace=True)
    monthly_delays_df.reset_index(drop=True, inplace=True)

    return monthly_delays_df

# Get monthly percentage of delays per airport
monthly_airport_delays_df = monthly_airport_delays(flights_df_raw.copy())
print(f"Monthly percentage of delays per airport:\n{monthly_airport_delays_df}")
```

## Setup Instructions

1. **Clone the Repository**:
    ```sh
    git clone https://github.com/your_username/flight-data-analysis.git
    cd flight-data-analysis
    ```

2. **Install Dependencies**:
    ```sh
    pip install pandas numpy
    ```

3. **Place Data Files**:
    Ensure the data files (flights.csv, airports.csv, airlines.csv) are placed in the appropriate directory or update the file paths accordingly in the notebook.

4. **Run the Jupyter Notebook**:
    Open and run the provided Jupyter Notebook to see the data manipulation and analysis.

## Key Results

1. **Number of Flights per Airport:** Calculated the number of departing flights from chosen airports.
2. **Top Airlines by Number of Flights and Delay Percentage:** Identified top airlines with the highest number of flights and the least delay percentage.
3. **Monthly Percentage of Delays per Airport:** Determined the monthly delay percentages for each selected airport.

## Contact Information

For any questions or further information, please contact:
- **Name:** Asad Kamal
- **Email:** aakamal {/@/} umich {/dot/} edu
- **LinkedIn:** [LinkedIn Profile](https://linkedin.com/in/asadakamal)
