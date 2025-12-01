# AHT20 Temperature and Humidity Sensor - API Reference

This section provides detailed descriptions of the functions used to interact with the AHT20 digital temperature and humidity sensor via I2C interface. These functions are part of a custom library for ATMEGA328 microcontroller that handles sensor initialization, calibration verification, data acquisition, and conversion to physical units. Each function is explained with its purpose and usage, accompanied by code examples to demonstrate how they can be implemented in your projects.

> [!NOTE]
> The library and all of its APIs provided below have been developed by myself.
> This library utilizes various macros defined in the `aKaReZa.h` header file, which are designed to simplify bitwise operations and register manipulations.
> Detailed descriptions of these macros can be found at the following link:
> [https://github.com/aKaReZa75/AVR/blob/main/Macros.md](https://github.com/aKaReZa75/AVR/blob/main/Macros.md)

| Setting                    | Value                   |
| -------------------------- | ----------------------- |
| **Sensor Model**           | AHT20                   |
| **Interface**              | I2C (TWI)               |
| **I2C Address**            | 0x38 (Fixed)            |
| **Temperature Range**      | -40°C to +85°C          |
| **Temperature Accuracy**   | ±0.3°C                  |
| **Humidity Range**         | 0% to 100% RH           |
| **Humidity Accuracy**      | ±2%                     |
| **Resolution**             | 20-bit                  |
| **Supply Voltage**         | 2.0V to 5.5V            |
| **Measurement Time**       | ~80ms                   |

---

## **Sensor Overview**

The AHT20 is a high-precision digital temperature and humidity sensor that uses I2C communication protocol. It features:

- 20-bit resolution for both temperature and humidity measurements
- Built-in calibration with automatic verification
- CRC-8 checksum for data integrity validation
- Low power consumption
- Fast measurement response time

### **Key Features:**
- **Calibration Check**: Automatic verification of sensor calibration during initialization
- **Soft Reset**: Built-in reset command to ensure clean state
- **CRC Validation**: Hardware CRC-8 checksum for reliable data transmission
- **Easy Integration**: Simple two-function API for complete sensor control

---

## **API Functions**

### **1. Sensor Initialization**

```c
AHT20_Res_T aht20_Init(void);
```

**Description:**
* Initializes the AHT20 sensor with automatic calibration check and verification.
* Performs soft reset to ensure the sensor starts in a clean state.
* Reads and validates the calibration status register.
* Sends initialization command if calibration is not detected.

**Return Value:**
* `AHT20_Res_OK` → Initialization successful, sensor calibrated and ready
* `AHT20_Res_ERR` → Initialization failed, sensor not calibrated

**Initialization Sequence:**
1. Wait 40ms after power-on for sensor stabilization
2. Send soft reset command (0xBA)
3. Wait 40ms for reset completion
4. Read status register (0x71)
5. Check calibration bit (bit 3)
6. If not calibrated: send initialization command (0xBE 0x08 0x00)
7. Wait 10ms for calibration process
8. Re-read status register to verify calibration success

**Example:**

```c
#include "aKaReZa.h"
#include "i2c.h"
#include "aht20.h"

int main(void) 
{
    i2c_Init(); /**< Initialize I2C peripheral */
    
    if (aht20_Init() == AHT20_Res_OK)
    {
        /**< Sensor initialized successfully */
        /* Your application code here */
    }
    else
    {
        /**< Initialization failed - handle error */
        while(1); /**< Halt or implement error recovery */
    }
    
    while(1)
    {
        /* Main loop */
    }
}
```

---

### **2. Temperature and Humidity Measurement**

```c
AHT20_Res_T aht20_getData(AHT20_Data_T* _Data);
```

**Description:**
* Triggers a new measurement cycle and reads temperature and humidity data.
* Validates sensor status flags (BUSY and CAL).
* Verifies data integrity using CRC-8 checksum.
* Extracts 20-bit raw values and converts them to physical units.

**Parameters:**
* `_Data` → Pointer to `AHT20_Data_T` structure to store converted values

**Return Value:**
* `AHT20_Res_OK` → Measurement successful, data valid
* `AHT20_Res_ERR` → Measurement failed (sensor busy, not calibrated, or CRC error)

**Measurement Sequence:**
1. Send trigger measurement command (0xAC 0x33 0x00)
2. Wait 80ms for measurement completion
3. Read 7 bytes: [Status | Humidity MSB | Humidity Mid | Humidity LSB/Temp MSB | Temp Mid | Temp LSB | CRC]
4. Check status flags: BUSY=0 (not busy), CAL=1 (calibrated)
5. Validate CRC-8 checksum
6. Extract 20-bit humidity value from bytes 1-3
7. Extract 20-bit temperature value from bytes 3-5
8. Convert raw values to °C and %RH

**Data Format:**
```
Byte 0: Status byte [BUSY(bit7) | ... | CAL(bit3) | ...]
Byte 1: Humidity[19:12] (MSB)
Byte 2: Humidity[11:4]
Byte 3: Humidity[3:0] | Temperature[19:16]
Byte 4: Temperature[15:8]
Byte 5: Temperature[7:0] (LSB)
Byte 6: CRC-8 checksum
```

**Conversion Formulas:**
```
Temperature (°C) = (Raw_Temperature × 200 / 2^20) - 50
Humidity (%)     = Raw_Humidity × 100 / 2^20
```

---

## **Data Types**

### **AHT20_Res_T (Return Status Enum)**

```c
typedef enum
{
    AHT20_Res_OK,       /**< Operation completed successfully */
    AHT20_Res_ERR,      /**< General error (calibration failed, sensor not responding) */
    AHT20_Res_TimeOut   /**< Timeout error (sensor busy too long, no response) */
} AHT20_Res_T;
```

**Description:**
* Enumeration type for function return values
* Used for error handling and status checking

---

### **AHT20_Data_T (Measurement Data Structure)**

```c
typedef struct 
{
    float Temp;      /**< Temperature in degrees Celsius (°C), range: -40 to +85 */
    float Humidity;  /**< Relative humidity in percentage (%), range: 0 to 100 */
} AHT20_Data_T;
```

**Description:**
* Structure containing converted temperature and humidity values
* Both values are in floating-point format for precision

---

## **CRC-8 Validation**

The AHT20 uses CRC-8 checksum for data integrity verification:

**CRC Configuration:**
- **Polynomial:** 0x31 (x^8 + x^5 + x^4 + 1)
- **Initial value:** 0xFF
- **Input reflection:** No
- **Output reflection:** No
- **Final XOR:** 0x00

**Validation Process:**
The CRC is calculated over all 7 bytes (status + 5 data bytes + CRC byte). A correct transmission results in a CRC calculation of 0x00.

---

## **Complete Application Example**

```c
#include "aKaReZa.h"
#include "i2c.h"
#include "aht20.h"
#include <stdio.h>

AHT20_Data_T sensorData;

void displayData(void)
{
    char buffer[50];
    sprintf(buffer, "Temp: %.1f°C, Humidity: %.1f%%\n", 
            sensorData.Temp, sensorData.Humidity);
    /* Send to UART or display */
}

int main(void) 
{
    /**< Initialize peripherals */
    i2c_Init();
    
    /**< Initialize AHT20 sensor */
    if (aht20_Init() != AHT20_Res_OK)
    {
        /**< Initialization failed */
        while(1)
        {
            bitToggle(PORTB, 0); /**< Blink error LED */
            delay_ms(200);
        }
    }
    
    /**< Main measurement loop */
    while(1)
    {
        /**< Get temperature and humidity data */
        if (aht20_getData(&sensorData) == AHT20_Res_OK)
        {
            /**< Data valid - process measurements */
            displayData();
            
            /**< Temperature control logic */
            if (sensorData.Temp > 30.0)
            {
                bitSet(PORTB, 1);   /**< Turn ON cooling system */
            }
            else if (sensorData.Temp < 20.0)
            {
                bitClear(PORTB, 1); /**< Turn OFF cooling system */
            }
            
            /**< Humidity control logic */
            if (sensorData.Humidity < 30.0)
            {
                bitSet(PORTB, 2);   /**< Turn ON humidifier */
            }
            else if (sensorData.Humidity > 60.0)
            {
                bitClear(PORTB, 2); /**< Turn OFF humidifier */
            }
        }
        else
        {
            /**< Measurement error - implement recovery */
            bitToggle(PORTB, 0); /**< Indicate error */
        }
        
        /**< Wait before next measurement */
        delay_ms(5000); /**< 5 second interval */
    }
}
```

---

## **API Function Summary**

| API Function     | Description                                                      |
| ---------------- | ---------------------------------------------------------------- |
| `aht20_Init`     | Initializes sensor with soft reset and calibration verification |
| `aht20_getData`  | Triggers measurement and reads temperature/humidity data         |

---

## **Troubleshooting Guide**

### **Initialization Fails (AHT20_Res_ERR)**
- Check I2C connections (SDA, SCL)
- Verify pull-up resistors are present (4.7kΩ recommended)
- Ensure proper power supply voltage (2.0V - 5.5V)
- Confirm I2C address is correct (0x38)
- Check if sensor has adequate power-on stabilization time

### **getData Returns Error**
- Sensor may still be busy from previous measurement
- CRC validation failure indicates communication error
- Check I2C bus speed (max 400kHz)
- Verify sensor is properly calibrated during initialization

### **Incorrect Readings**
- Ensure adequate time between measurements (minimum 80ms)
- Verify conversion factors are correct
- Check for I2C bus interference or noise
- Confirm sensor is not exposed to condensation

> [!IMPORTANT]
> - Always call `aht20_Init()` before using `aht20_getData()`
> - Wait at least 40ms after power-on before initialization
> - Pull-up resistors (4.7kΩ) are required on SDA and SCL lines
> - Measurement time is approximately 80ms - do not read too frequently
> - The sensor requires recalibration after power cycles

---

## **Dependencies**

This library requires the following dependencies:

1. **aKaReZa.h** - Base library with utility macros
   - Repository: https://github.com/aKaReZa75/AVR_RawProject

2. **i2c.h** - I2C communication library
   - Repository: https://github.com/aKaReZa75/AVR_I2C

3. **err.h** - Error detection library (for CRC-8)
   - Repository: https://github.com/aKaReZa75/Error_Detection

---


# 🌟 Support Me
If you found this repository useful:
- Subscribe to my [YouTube Channel](https://www.youtube.com/@aKaReZa75).
- Share this repository with others.
- Give this repository and my other repositories a star.
- Follow my [GitHub account](https://github.com/aKaReZa75).

# ✉️ Contact Me
Feel free to reach out to me through any of the following platforms:
- 📧 [Email: aKaReZa75@gmail.com](mailto:aKaReZa75@gmail.com)
- 🎥 [YouTube: @aKaReZa75](https://www.youtube.com/@aKaReZa75)
- 💼 [LinkedIn: @akareza75](https://www.linkedin.com/in/akareza75)
