# sensor-block
Auto-detects connected i2c sensors and publsihes data on HTTP or MQTT.

## Features
- Uses Indusrial IO (iio) to communicate with sensors, utilizing drivers already in the kernel to talk to the sensor directly
- Data published via mqtt and/or http
- Provides raw sensor data or "tranforms" the sensor data into a more standardized format 
- json output can either be one measurement per sensor or all sensor fields in one list 

## Overview/Compatibility
This block utilizes the [Linux Industrial I/O Subsystem](https://wiki.analog.com/software/linux/docs/iio/iio) ("iio") which is a kernel subsystem that allows for ease of implementing drivers for sensors and other similar devices such as ADCs, DACs, etc.  You can see a list of available iio drivers [here](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/iio?h=linux-5.4.y) but in order to save space, most OSes do not include all these drivers. The easiest way to check if your board supports a driver is to use the modinfo command on a running device. For instance:
```
modinfo ti-ads1015
```
This command searches the running kernel for all the drivers it includes and prints out a description of any that are found to match. (Use the info in the Kconfig file in each folder of the driver list above to find the proper driver name to use with this command) BalenaOS for Raspberry Pi 3 includes a small subset of these drivers which are listed in the table below. As we continue to test sensors and improve the block, more sensors should be supported and the chart will be updated accordingly. Also note that as kernel and OS versions change, the supported drivers may change somewhat as well. It's best therefore to use modinfo as described above to test compatibility with any sensor being considered for use with this block.

| Sensor Model | Sensor Name | Driver Name | Address(es) | Tested? |
| ------------ | ----------- | ----------- | ----------- | ------- |
| AD5301 | Analog Devices AD5446 and similar single channel DACs driver, TI DACs | ads5446 | 0xC, 0xD, 0xE, 0xF | Not tested |
| APDS9960 | Avago APDS9960 gesture/RGB/ALS/proximity sensor | apds9960 | 0x39 | Yes, NOT working |
| BME680 | Bosch Sensortec BME680 sensor | bme680 | 0x76, 0x77 | Yes, works |
| BMP180 | Bosch Sensortec BMP180 sensor | bmp280 | 0x77 | Not tested |
| BMP280 | Bosch Sensortec BMP280 sensor | bmp280 | 0x76, 0x77 | Yes, works |
| BME280 | Bosch Sensortec BME280 sensor | bmp280 | 0x76, 0x77 | Yes, works |
| HDC1000 | TI HDC100x relative humidity and temperature sensor | hdc100x | 0x40 - 0x43 | Not tested |
| HTU21 | Measurement Specialties HTU21 humidity & temperature sensor | htu21 | 0x40 | Yes, works |
| MS8607 | TE Connectivity PHT sensor | htu21 | 0x40, 0x76 | Not tested |
| MCP342x | Microchip Technology MCP3421/2/3/4/5/6/7/8 ADC | mcp3422 | 0x68 - 0x6F | Not tested |
| ADS1015 | Texas Instruments ADS1015 ADC | ti-ads1015 | 0x48 - 0x4B | Yes, NOT working |
| TSL4531 | TAOS TSL4531 ambient light sensors | tsl4531 | 0x29 | Not tested |
| VEML6070 | VEML6070 UV A light sensor | veml6070 | 0x38, 0x39 | Yes, works |

## Usage

**Docker compose file**

To use this image, create a container in your `docker-compose.yml` file as shown below:

```
services:
  sensor:
    image: balenablocks/sensor:raspberrypi4-64 # Use alanb128/sensor-block:latest for testing
    privileged: true
    cap_add:
      - ALL
    labels:
      io.balena.features.kernel-modules: '1'
      io.balena.features.sysfs: '1'
      io.balena.features.balena-api: '1'
    expose:
      - '7575'  # Only needed if using webserver
```
### Publishing Data

The sensor data is available in json format either as an mqtt payload or via the built-in webserver. To use mqtt, either include a container in your application that is named mqtt or provide an address for the `MQTT_ADDRESS` service variable (see below.)

If no mqtt container is present and no mqtt address is set, the webserver will be available on port 7575. To force the webserver to be active, set the `ALWAYS_USE_WEBSERVER` service variable to True.

The http data defaults to only be available to other containers in the application via `sensor:7575` - if you want this to be available externally, you'll need to map port 7575 to an external port in your docker-compose file. To configure http/mqtt, use the following service variables:

`MQTT_ADDRESS` 	Provide the address of an MQTT broker for the block to publish data to. If this variable is not set and a container on the device is named mqtt, it will publish to that instead. Either setting this or having a container named mqtt disables the internal webserver unless `ALWAYS_USE_WEBSERVER` is set to True.

`ALWAYS_USE_WEBSERVER` 	Set to True to enable the internal webserver even when it is automatically disabled due to the detection of mqtt. Default value is 0.


## Data

The JSON for raw sensor data is available in one of two formats and is determined by the `COLLAPSE_FIELDS` service variable. The default value of `0` (zero) causes each sensor to output a separate measurement:
```


`RAW_VALUES` Default value of `1` provides raw field names and values from sensors. Setting this to `0` standardizes the field names and adjusts the values as needed. See the file `transformers.py` for the full set of modifcations.

`COLLAPSE_FIELDS` The default value of `0` (zero) causes each sensor to output a separate measurement. Setting this to `1` collapses all fields into one list.

The JSON for raw sensor data will be in the following format: 
```
[{'measurement': 'htu21', 'fields': {'humidityrelative': '29700', 'temp': '23356'}}, {'measurement': 'bmp280', 'fields': {'pressure': '99.911941406', 'temp': '23710'}}]
```
Each device will have its own separate measurement, with fields for each attribute/value. This format is especially suited to use with influxDB.

## Use with other blocks

TBA
