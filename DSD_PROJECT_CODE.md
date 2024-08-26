# DSD Project Code

In deze Markdown beschrijf ik de door mij geschreven code voor het DSD-project.

## Vivado en Vitis IDE Versie

- **Versie:** Vivado 2021.2 en Vitis 2021.2

> **Let op:** De PMOD gebruikt AXI-interfaces die vanaf versie 2022.1 niet backwards compatible zijn. Gebruik daarom Vivado 2021.2 of een eerdere versie.

## PMOD Libraries

- **Repository:** [Digilent Vivado Library](https://github.com/Digilent/vivado-library/tree/master/ip/Pmods)

## PMOD Libraries Toevoegen aan Vivado en Vitis IDE

- **Handleiding:** [Getting Started with PMOD IPs](https://digilent.com/reference/programmable-logic/guides/getting-started-with-pmod-ips?srsltid=AfmBOorFdtCm8b8c7IFsYHaj8t2RQy5YApuOT9F8YyBj0P_aCtTB9swp)

In deze sectie importeer ik de libraries die ik nodig heb om het project uit te voeren. Voor dit project zijn de UART en IIC libraries het meest belangrijk.

```c
// Codefragmenten met alle libraries
#include "PmodESP32.h"
#include "PmodHYGRO.h"
#include "xparameters.h"
#include "xil_printf.h"
#include "sleep.h"
```
### Globale Variabelen

```c
// UART and I2C configurations
#define HOST_UART_DEVICE_ID XPAR_AXI_UARTLITE_0_DEVICE_ID
#define PMODESP32_UART_BASEADDR XPAR_PMODESP32_0_AXI_LITE_UART_BASEADDR
#define PMODESP32_GPIO_BASEADDR XPAR_PMODESP32_0_AXI_LITE_GPIO_BASEADDR
//init PMODs
PmodESP32 myESP32;
PmodHYGRO myHYGRO;
HostUart myHostUart;
```
### function prototypes
```c
// Function prototypes
void enableCaches();
void disableCaches();
void initialize();
void run();
void cleanup();
```

### PMOD initialize
* **IIC Setup:** Start de IIC (Two-Wire) bus.
* **UART Setup:** Start de UART-bus met adres: 0x40.
* **Netwerk Setup:** Maak verbinding met een netwerk, in dit geval een hotspot op mijn gsm (verbinding met AP WiFi werkt niet door de instellingen van de school).
```c
// Codefragmenten voor de setup van de code
void initialize() {
    HostUart_Config *CfgPtr;
    enableCaches();

    xil_printf("Initializing ESP32...\n\r");
    ESP32_Initialize(&esp32, PMODESP32_UART_BASEADDR, PMODESP32_GPIO_BASEADDR);
    CfgPtr = HostUart_LookupConfig(HOST_UART_DEVICE_ID);
    HostUart_CfgInitialize(&hostUart, CfgPtr, HostUartConfig_GetBaseAddr(CfgPtr));

    xil_printf("Initializing HYGRO Sensor...\n\r");
    HYGRO_begin(&hygro, XPAR_PMODHYGRO_0_AXI_LITE_IIC_BASEADDR, 0x40, 
                XPAR_PMODHYGRO_0_AXI_LITE_TMR_BASEADDR, XPAR_PMODHYGRO_0_DEVICE_ID, 100000000);
}
```
### ESP32 PMOD
**Referenties:**
* **ESP32 PMOD Reference Manual:** Gedetailleerde handleiding voor het gebruik van de ESP32 PMOD met specificaties, pinconfiguraties en voorbeeldimplementaties.
* **ESP32 AT Commands:** Uitgebreide documentatie van de AT-commandos voor de ESP32, inclusief voorbeelden en toepassingen.
### Uitleg ESP32 PMOD Code:
**Interval:** Om de 30 seconden worden AT-commandos verstuurd om de ESP32 te besturen.
**Verbinding:** Verbind met de MQTT broker als publisher voor het verzenden van gegevens.
**Publiceren:** Publiceer berichten en wis de gebruikte variabelen om geheugen vrij te maken.

## HYGRO PMOD
**Referenties:**
* **Pmod HYGRO Reference Manual:** Officiële handleiding van Digilent voor de Pmod HYGRO module, met details over functionaliteit, pinconfiguraties en voorbeeldprojecten.
* **HDC1080 I2C Commands:** Datasheet van de HDC1080 sensor gebruikt in de Pmod HYGRO, inclusief alle benodigde I2C-commando's voor communicatie en dataverzameling.
### Uitleg HYGRO PMOD Code:
**Commando verzenden:** Stuur een I2C-commando naar de HDC1080 om data op te halen.
**Data ophalen:** Lees de temperatuur- en vochtigheidsgegevens uit de sensor.
**Data converteren:** Converteer de ruwe sensorwaarden naar leesbare temperatuur- en vochtigheidsgegevens.
**Data opslaan:** Return de geconverteerde data en sla deze op in een variabele.


## Main Code
**Uitleg:** De main code sectie bevat de hoofdlogica van het programma, waarin de verschillende onderdelen zoals de PMODs, netwerkverbindingen, en sensoren met elkaar samenwerken. Hier wordt de hoofdloop gedefinieerd waarin de verschillende functies worden aangeroepen en de data wordt verwerkt.

**Setup:** Initialiseer alle benodigde hardware zoals de PMODs, UART, en IIC.
**Loop:** Implementeer een lus waarin periodiek gegevens worden uitgelezen, verwerkt en verzonden.
**Afhandeling:** Zorg ervoor dat verbindingen correct worden gesloten en middelen worden vrijgegeven.
```c
// main loop
void run() {
    char command[128];
    float temp_degc, hum_perrh;

    // Connect to Wi-Fi
    sprintf(command, "AT+CWJAP=\"Your_SSID\",\"Your_PASSWORD\"");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(5);

    // Configure MQTT with SSL
    sprintf(command, "AT+MQTTUSERCFG=0,1,\"AP HOGESCHOOl\",\"2324APHogeschool\",\"\",0,0,\"\"");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(2);

    sprintf(command, "AT+MQTTCONN=0,\"3816a8ccc80148e783ccdaf0467b606b.s1.eu.hivemq.cloud\",8883,1");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(5);

    while (1) {
        // Read temperature and humidity from the sensor
        temp_degc = HYGRO_getTemperature(&hygro);
        hum_perrh = HYGRO_getHumidity(&hygro);

        // Convert and publish temperature
        sprintf(command, "AT+MQTTPUB=0,\"/Temp\",\"%.2f\",0,0", temp_degc);
        ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));

        // Convert and publish humidity
        sprintf(command, "AT+MQTTPUB=0,\"/Humidity\",\"%.2f\",0,0", hum_perrh);
        ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));

        // Delay before next reading
        sleep(10);
    }
}
```

## Full Code
**Uitleg:**

In de sectie voor de volledige code voeg je alle codefragmenten samen tot één compleet programma. Dit is handig voor het overzicht en het delen van de volledige implementatie met anderen. De full code sectie bevat alle functies, setup-instellingen, globale variabelen en de main code geïntegreerd in één bestand.

**Complete Implementatie:** Alle hierboven beschreven onderdelen (libraries, variabelen, setup, specifieke functies voor PMODs, en main code) worden hier samengevoegd.
**Testen:** Het volledige programma kan hier direct getest worden door het in te laden in de IDE en uit te voeren op de hardware.
```c
#include "PmodESP32.h"
#include "PmodHYGRO.h"
#include "xparameters.h"
#include "xil_printf.h"
#include "sleep.h"

// UART and I2C configurations
#define HOST_UART_DEVICE_ID XPAR_AXI_UARTLITE_0_DEVICE_ID
#define PMODESP32_UART_BASEADDR XPAR_PMODESP32_0_AXI_LITE_UART_BASEADDR
#define PMODESP32_GPIO_BASEADDR XPAR_PMODESP32_0_AXI_LITE_GPIO_BASEADDR

PmodESP32 esp32;
PmodHYGRO hygro;
HostUart hostUart;

// Function prototypes
void enableCaches();
void disableCaches();
void initialize();
void run();
void cleanup();

void initialize() {
    HostUart_Config *CfgPtr;
    enableCaches();

    xil_printf("Initializing ESP32...\n\r");
    ESP32_Initialize(&esp32, PMODESP32_UART_BASEADDR, PMODESP32_GPIO_BASEADDR);
    CfgPtr = HostUart_LookupConfig(HOST_UART_DEVICE_ID);
    HostUart_CfgInitialize(&hostUart, CfgPtr, HostUartConfig_GetBaseAddr(CfgPtr));

    xil_printf("Initializing HYGRO Sensor...\n\r");
    HYGRO_begin(&hygro, XPAR_PMODHYGRO_0_AXI_LITE_IIC_BASEADDR, 0x40, 
                XPAR_PMODHYGRO_0_AXI_LITE_TMR_BASEADDR, XPAR_PMODHYGRO_0_DEVICE_ID, 100000000);
}

void run() {
    char command[128];
    float temp_degc, hum_perrh;

    // Connect to Wi-Fi
    sprintf(command, "AT+CWJAP=\"Your_SSID\",\"Your_PASSWORD\"");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(5);

    // Configure MQTT with SSL
    sprintf(command, "AT+MQTTUSERCFG=0,1,\"AP HOGESCHOOl\",\"2324APHogeschool\",\"\",0,0,\"\"");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(2);

    sprintf(command, "AT+MQTTCONN=0,\"3816a8ccc80148e783ccdaf0467b606b.s1.eu.hivemq.cloud\",8883,1");
    ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));
    sleep(5);

    while (1) {
        // Read temperature and humidity from the sensor
        temp_degc = HYGRO_getTemperature(&hygro);
        hum_perrh = HYGRO_getHumidity(&hygro);

        // Convert and publish temperature
        sprintf(command, "AT+MQTTPUB=0,\"/Temp\",\"%.2f\",0,0", temp_degc);
        ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));

        // Convert and publish humidity
        sprintf(command, "AT+MQTTPUB=0,\"/Humidity\",\"%.2f\",0,0", hum_perrh);
        ESP32_SendBuffer(&esp32, (u8*)command, strlen(command));

        // Delay before next reading
        sleep(10);
    }
}

void cleanup() {
    disableCaches();
}

int main() {
    initialize();
    run();
    cleanup();
    return 0;
}

void enableCaches() {
#ifdef __MICROBLAZE__
#ifdef XPAR_MICROBLAZE_USE_ICACHE
   Xil_ICacheEnable();
#endif
#ifdef XPAR_MICROBLAZE_USE_DCACHE
   Xil_DCacheEnable();
#endif
#endif
}

void disableCaches() {
#ifdef __MICROBLAZE__
#ifdef XPAR_MICROBLAZE_USE_ICACHE
   Xil_ICacheDisable();
#endif
#ifdef XPAR_MICROBLAZE_USE_DCACHE
   Xil_DCacheDisable();
#endif
#endif
}
```

### Toelichting:
- **Main Code:** In deze sectie wordt de hoofdlogica van het programma beschreven. Het is de kern van je project waar alle individuele onderdelen samenkomen en samenwerken. Hier moet je code komen voor de setup en de hoofdlus die je systeem draaiende houdt.
  
- **Full Code:** Dit is waar je alle code uit het document combineert tot één geheel. Deze sectie is handig voor een compleet overzicht en zorgt ervoor dat je de volledige implementatie eenvoudig kunt kopiëren en gebruiken.