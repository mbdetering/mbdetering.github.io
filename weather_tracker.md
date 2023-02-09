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

**Python Packages Used:** tkinter, PIL: ImageTk, Image

Creating the GUI was fairly straightfoward. First I needed to implement a window for the user to enter their api key, and the city which they wanted to request data from. This required two methods to be created. One to check the validity of the user inputs, and one to display an error message:

```Python
    # creates temporary tkinter window indicating that an error has occured in requesting the openweathermap api
    def error():
        popup = tk.Toplevel()
        popup.title("ERROR!")

        label = tk.Label(popup, text="invalid api key or city, please try again")
        label.pack(pady=10)

        close_button = tk.Button(popup, text="Close", command=popup.destroy)
        close_button.pack(pady=10)
```

```Python
    # validates that the inputed city and api key are valid with the openweathermap api. If the two are not valid,
    # displays an error message, if the two are valid, closes the window and returns the inputs
    # allowing for the main app to run.
    def validate_key():
        nonlocal api_key
        nonlocal city
        api_key = key_entry.get()
        city = city_entry.get()
        if 'main' not in we.extract_current_weather_data(api_key, city):
            error()
        else:
            key_window.destroy()
```

These methods, along with the tkinter widgets for the main entry window would create two new windows:

<img src="images/screenshots/check.png?raw=true"/><img src="images/screenshots/error.png?raw=true"/>

Creating the main GUI was straight forward. I first used the methods created earlier to extract and transform the correct data, then displayued the data using tkinter's built in widgets. The only complicated process was to display the icons for the 5-day foreacast. In order to do this, I created a list of filepaths for each day, then created new image objects, selecting the corrresponding image.

```Python
paths = []
for desc in forecast_averages['weather']:
    line = f'pics/{desc}.png'
    paths.append(line)
imgs = []
x = 100
for i in range(len(paths)):
    imgs.append(ImageTk.PhotoImage(Image.open(paths[i])))
    tk.Label(root, image=imgs[-1], width=50, height=50).place(x=x, y=440, anchor="center")
    x += 100
```

After implementing these features the gui was completed displaying both current and forecast data for a specified location.

<img src="images/screenshots/GUI.png?raw=true"/>


