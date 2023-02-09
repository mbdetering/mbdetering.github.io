## Weather Tracker Application

**Project description:** This project was created as a challenge for myself to practice my python development, data engineering, and data analysis skills. The first part of this project was to create a weather application that pulled current and forecast weather data from an api, then displayed in a simple gui. The second part was to implement scheduled python tasks to collect weather dat throughout the day in order to build up a database of weather data that could then be used to see small and large scale weather patterns through data analytic techniques.

**Technologies Used:** Python, Power BI, MySQL, Apache-airflow

### Part 1. Requesting and Preparing Weather Data

**Python Packages Used:** pandas, requests, json, datetime

The first problem I decided to tackle was finding an api that could be used to gather weather data. After doing research, I decided on using the OpenWeatherMap api, an api that allows for requests of current weather data as well as forecast data for a selected region. This worked perfectly for my needs and only required a few simple request methods to get all of the data I needed. These methods were:

```Python
# Takes in an api key and city and requests the current weather from the openweathermap api as a JSON object
def extract_current_weather_data(api_key, city):
    c_url = f"https://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}"
    response_c = requests.get(c_url)
    current_weather = json.loads(response_c.text)
    return current_weather
```

```Python
# Takes in an api key and city and requests a 5 day 3 hour forecast from the openweathermap api as a JSON object
def extract_forecast(api_key, city):
    f_url = f"https://api.openweathermap.org/data/2.5/forecast?q={city}&appid={api_key}"
    response_f = requests.get(f_url)
    forecast = json.loads(response_f.text)
    return forecast
```

The next step was to subset and convert the requested data into a format that would be easily readable, and to convert the units of some of the data typers. To do this I first created a method to convert from Kelvin to fahrenheit, then created two methods that subset each type of extracted data, converting them into pandas DataFrame objects.

```Python
# Takes in a temperature in Kelvin and converts to Fahrenheit rounding to 2 decimal places
def k2f(temp):
    temp = 1.8 * (temp - 273) + 32
    return round(temp, 2)
```

```Python
# Takes in a JSON current weather object and extracts and converts the data within into a pandas DataFrame object
def transform_current_weather_data(data):
    current_weather = data['main']
    current_weather['temp'] = k2f(current_weather['temp'])
    current_weather['feels_like'] = k2f(current_weather['feels_like'])
    current_weather['temp_min'] = k2f(current_weather['temp_min'])
    current_weather['temp_max'] = k2f(current_weather['temp_max'])
    current_weather['pressure'] = round(current_weather['pressure'] / 33.684, 2)
    weather_main = data['weather'][0]['main']
    weather_main = {'main': weather_main}
    current_weather.update(weather_main)
    weather_desc = data['weather'][0]['description']
    weather_desc = {'description': weather_desc}
    current_weather.update(weather_desc)
    return pd.DataFrame(current_weather, index=[0])
```

```Python
# Takes in a JSON object of 5 day forecast data and extracts and converts to a pandas DataFrame object
def transform_weather_forecast(data):
    forecast_data = [(d['dt_txt'], k2f(d['main']['temp']), k2f(d['main']['temp_max']), k2f(d['main']['temp_min']),
                      d['main']['humidity'], d['weather'][0]['main'], d['weather'][0]['description']) for d in data['list']]
    return pd.DataFrame(forecast_data, columns=["Time", "temp", "temp_max", "temp_min", "humidity", "weather", "description"])
```

The final method I needed to create was method to aggregate the forecast data. The forecast data that was pullled from the OpenWeatherMap api contained a 5-day 3-hour forecast. This was too much for a simple weather app so I wanted to aggregate the data to see the overall forecast for each day.

```Python
# Takes in a dataframe of hourly weather data and aggregates summarizing by day before returning summarized DataFrame
def mode_description(df):
    df_subset = df[['Time', 'temp_max', 'temp_min', 'weather', 'description']]
    df_subset['Time'] = pd.to_datetime(df_subset['Time']).dt.date
    df_grouped = df_subset.groupby('Time', as_index=False).agg({
        'temp_max': max,
        'temp_min': min,
        'weather': lambda x: x.mode().iloc[0],
        'description': lambda x: x.mode().iloc[0]
    })
    today = datetime.datetime.now().date()
    if today in df_grouped['Time'].unique():
        df_grouped = df_grouped[df_grouped['Time'] != today]
    result = df_grouped.reset_index()
    return result
```

### Part 2. Creating the weather app GUI
