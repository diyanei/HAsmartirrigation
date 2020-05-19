[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg)](https://github.com/custom-components/hacs)

[![Support the author on Patreon][patreon-shield]][patreon]

[![Buy me a coffee][buymeacoffee-shield]][buymeacoffee]

[patreon-shield]: https://frenck.dev/wp-content/uploads/2019/12/patreon.png
[patreon]: https://www.patreon.com/dutchdatadude

[buymeacoffee]: https://www.buymeacoffee.com/dutchdatadude
[buymeacoffee-shield]: https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png


# Smart Irrigation
![](logo.png?raw=true)

Smart Irrigation custom component for Home Assistant. Partly based on the excellent work at https://github.com/hhaim/hass/.
This component calculates the time to run your irrigation system to compensate for moisture lost by evaporation / evapotranspiration. Using this component you water your garden, lawn or crops precisely enough to compensate what has evaporated. It takes into account precipitation (rain,snow) and adjusts accordingly, so if it rains or snows less or no irrigation is required.

The component keeps track of hourly precipitation and at 23:00 (11:00 PM) local time stores it in a daily value. It then calculates the exact runtime in seconds to compensate for the net evaporation. You can get this value from `sensor.smart_irrigation.daily_adjusted_run_time`. See the example automation below.

This component uses reference evapotranspiration values and calculates base schedule indexes and water budgets from that. This is an industry-standard approach. Information can be found at https://www.rainbird.com/professionals/irrigation-scheduling-use-et-save-water, amongst others.
The component uses the [PyETo module to calculate the evapotranspiration value (fao56)](https://pyeto.readthedocs.io/en/latest/fao56_penman_monteith.html).

## Configuration

### Step 1: configuration of component
Install the custom component (preferably using HACS) and then use the Configuration --> Integrations pane to search for 'Smart Irrigation'.
You will need to specify the following:
- API Key for Open Weather Map. See [Getting Open Weater Map API Key](#getting-open-weather-map-api-key) below for instructions.
- Reference Evapotranspiration for all months of the year. See [Getting Monthly ET values](#getting-monthly-et-values) below for instructions. Note that you can specify these in inches or mm, depending on your Home Assistant settings.
- Number of sprinklers in your irrigation system
- Flow per spinkler in gallons per minute or liters per minute. Refer to your sprinkler's manual for this information.
- Area that the sprinklers cover in square feet or m2

### Step 2: checking entities
After successful configuration, you should end up with three entities and their attributes, listed below.
#### `sensor.smart_irrigation_base_schedule_index`
The number of seconds the irrigation system needs to run assuming maximum evapotranspiration and no rain / snow. This value and the attributes are static for your configuration.
Attributes:
| Attribute | Description |
| --- | --- |
|`number of sprinklers`|number of sprinklers in the system|
|`flow`|amount of water that flows through a single sprinkler in liters or gallon per minute|
|`throughput`|total amount of water that flows through the irrigation system in liters or gallon per minute.|
|`reference evapotranspiration`|the reference evapotranspiration values provided by the user - one for each month.|`peak evapotranspiration`|the highest value in `reference evapotranspiration`|
|`area`|the total area the irrigation system reaches in m<sup>2</sup> or sq ft.|
|`precipitation rate`|the output of the irrigation system across the whole area in mm or inch per hour|

Sample screenshot:

![](images/bsi_entity.png?raw=true)

#### `sensor.smart_irrigation_hourly_adjusted_run_time`
The adjusted run time in seconds to compensate for any net moisture lost. Updated approx. every 60 minutes.
Attributes:
| Attribute | Description |
| --- | --- |
|`rain`|the predicted rainfall in mm or inch|
|`snow`|the predicted snowfall in mm or inch|
|`precipitation`|the total predicted precipitation in mm or inch|
|`evapotranspiration`|the expected evapotranspiration|
|`netto precipitation`|the net evapotranspiration in mm or inch, negative values mean more moisture is lost than gets added bu rain/snow, while positive values mean more value is added by rain/snow than evaporates|
|`water budget`|percentage of expected `evapotranspiration` vs `peak evapotranspiration`|

Sample screenshot:

![](images/hart.png?raw=true)

#### `sensor.smart_irrigation_daily_adjusted_run_time`

The adjusted run time in seconds to compensate for any net moisture lost. Updated every day at 11:00 PM / 23:00 hours local time. Use this value for your automation (see step 3, below).
Attributes:
| Attribute | Description |
| --- | --- |
|`water budget`|percentage of net precipitation / base schedule index|
|`bucket`|running total of net precipitation. Negative values mean that irrigation is required. Positive values mean that more moisture was added than has evaporated yet, so irrigation is not required. Should be reset to `0` after each irrigation, using the `smart_irrigation.reset_bucket` service|

Sample screenshot:

![](images/dart.png?raw=true)

You will use `sensor.smart_irrigation_daily_adjusted_run_time` to create an automation (see step 3, below).

The [How this works section](#how-this-works) describes the entities, the attributes and the calculations

### Step 3: creating automation
Since this component does not interface with your irrigation system directly, you will need to use the data it outputs to create an automation that will start and stop your irrigation system for you. This way you can use this custom component with any irrigation system you might have, regardless of how that interfaces with Home Assistant.

Here is an example automation:
```
- alias: Smart Irrigation
  description: 'Start Smart Irrigation at 06:00 and run it only if the adjusted_run_time is >0 and run it for precisely that many seconds'
  trigger:
  - at: 06:00
    platform: time
  condition:
  - above: '0'
    condition: numeric_state
    entity_id: sensor.smart_irrigation_daily_adjusted_run_time
  action:
  - data: {}
    entity_id: switch.irrigation_tap1
    service: switch.turn_on
  - delay:
      seconds: '{{states("sensor.smart_irrigation_daily_adjusted_run_time")}}'
  - data: {}
    entity_id: switch.irrigation_tap1
    service: switch.turn_off
  - data: {}
    service: smart_irrigation.reset_bucket
```

> **The last step in this automation is important, since you will need to let the component know you have finished irrigating and the evaporation counter can be reset.**

## How this works
This section describes how this component works and how the input values you specify when setting up the component result in the values and attributes of the three entities created.

### Rationale
This component uses the industry-standard approach of calculating a Base Schedule Index (BSI) based on reference evapotranspiration values. It then compares these reference values to the actual current values and calculates the amount of time the irrigation needs to run to compensate for evapotranspiration. Information can be found at https://www.rainbird.com/professionals/irrigation-scheduling-use-et-save-water, amongst others.

### Definitions
The following definitions are used in this component:
- **Area**: total area that the irrigation system reaches in m<sup>2</sup> or sq ft
- **Adjusted Run Time** : time in seconds that the irrigation system needs to run to compensate for any net moisture lost.
- **Base Schedule Index (BSI)**: is the number of seconds the irrigation needs to run to compensate for the highest reference evapotranspiration value provided
- **Bucket**: Running total of net precipitation. Negative values mean that irrigation is required. Positive values mean that more moisture was added than has evaporated yet, so irrigation is not required. Should be reset to `0` after each irrigation, using the `smart_irrigation.reset_bucket` service.
- **Evapotranspiration**: the moisture evaporating from the soil and plants due to factors such as wind speed, sun exposure and temperature
- **Flow**: the amount of water that runs through a single sprinkler in liters or gallons per minute
- **Peak Evapotranspiration**: the highest evapotranspiration value in the reference evapotranspiration values.
- **Precipitation**: sum of rain and snow fall in mm or inch
- **Precipitation Rate**: the output of the irrigation system across the whole area in mm / hour or inch / hour
- **Netto precipitation**: the net evapotranspiration in mm or inch, negative values mean more moisture is lost than gets added bu rain/snow, while positive values mean more value is added by rain/snow than evaporates.
- **Reference evapotranspiration**: the reference evapotranspiration values provided by the user - one for each month.
- **Throughput**: the output of the irrigation system in liters or gallons per minute
- **Water budget**: percentage of current evapotranspiration vs peak evapotranspiration


### Calculating Base Schedule Index
The Base Schedule Index (BSI) is the number of seconds the irrigation needs to run to compensate for the highest reference evapotranspiration value provided. This is a static value based on the user input, calculated once at setup.
It is defined as:
![](images/bsi.png?raw=true)

### Calculating Adjusted Run Time
The Adjusted Run Time (ART) is the number of seconds the irrigation needs to run to compensate for moisture lost that is not compensated by rain or snow. It is defined as:

![](images/art.png?raw=true)

Evapotranspiration is calculated using the Penman - Monteith method. More details are in [Allen RG, Pereira LS, Raes D, Smith M (1998) Crop evapotranspiration](http://www.fao.org/3/X0490E/x0490e00.htm). This component uses the [PyETo module to calculate the fao56 evapotranspiration value](https://pyeto.readthedocs.io/en/latest/fao56_penman_monteith.html).

## Getting Open Weather Map API key
Go to https://openweathermap.org and create an account. You can enter any company and purpose while creating an account. After creating your account, go to API Keys and get your key. If the key does not work right away, no worries. The email you should have received from OpenWeaterMap says it will be activated 'within the next couple of hours'. So if it does not work right away, be patient a bit.

## Getting Monthly ET values
To get the monthly ET values use Rainmaster (US only), World Water & Climate Institute (worldwide) or another source that has this information for your area. 
> **When entering the ET values in the configuration of this component, keep in mind that the component will expect inches or mm depending on the settings in Home Assistant (imperial vs metric system).**

### Using Rainmaster (US only)
Go to http://www.rainmaster.com/historicET.aspx and enter your 5-digit zipcode and click 'Find'. The values you are looking for are listed for each month in inch/day:
![](images/rainmaster.png?raw=true)


### World Water & Climate Institute (International)
Go to http://wcatlas.iwmi.org/ and create an account. Once logged in, enter the locations you are interested in and click 'Submit'. Be sure to select N/S and E/W according to your coordinates:
![](images/iwmi1.PNG?raw=true)

The values you are looking for are in the last column (Penman ETo (mm/day)):
![](images/iwmi2.png?raw=true)

