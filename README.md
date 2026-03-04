# ESP8266 + BMP280 → MQTT (JSON)

Proyecto didáctico de meteorología IoT para 1º de Bachillerato

## Descripción

Este repositorio contiene un proyecto en Arduino IDE para **ESP8266 (NodeMCU)** que lee un sensor barométrico **BMP280** por I2C (temperatura, presión y altitud estimada) y envía los datos en **formato JSON** a un **broker MQTT** cada 5 segundos.

Está pensado como práctica guiada para alumnado de 1º de Bachillerato: aprender I2C, sensores, redes WiFi, mensajería MQTT y estructuración de datos con JSON.

***

## Funcionalidad 

- Se conecta a una red **WiFi** (con reintentos si falla).
- Se conecta a un **broker MQTT** (con reconexión automática).
- Publica un mensaje de estado **Online/Offline** usando LWT (Last Will) para monitorizar si el nodo cae.[^1]
- Inicializa y configura el **BMP280**.
- Cada `SEND_PERIOD_MS`:
    - Lee `temperatura_c`, `presion_hpa`, `altitud_m`.
    - Construye un JSON con:
        - Metadatos del nodo (IP, RSSI).
        - Timestamp relativo (`millis()`).
        - Valores del sensor.
    - Publica el JSON en un topic MQTT.

***

## 1) Material necesario

- 1 × ESP8266 NodeMCU (o compatible)
- 1 × Sensor BMP280 por I2C
- [Descargar aquí](https://sparks.gogo.co.nz/ch340.html) el **Driver CH340**  

**NOTA**: Si aparece algún error a la hora de instalar el driver, instalarlo con la placa Wemos D1 conectada al PC.
- Cables Dupont

***

## 2) Cableado (Wemos D1 + BMP280 por I2C)

En **Wemos D1**, los pines I2C más usados son:

- **D1 = SCL = GPIO5**
- **D2 = SDA = GPIO4**

Si no sabes ubicarlos en la placa, a pesar de que están serigrafiados, búscalos en google (Wemos D1 pinout).

### Conexiones (I2C)

![Nuevo Presentación de Microsoft PowerPoint](https://github.com/user-attachments/assets/03c42897-e598-430f-830e-7facc8c6fbce)


- Wemos **3V3** → BMP280 **VCC / VIN** (usa 3.3V)
- Wemos **G** (GND) → BMP280 **GND**
- Wemos **D1 (GPIO5 / SCL)** → BMP280 **SCL**
- Wemos **D2 (GPIO4 / SDA)** → BMP280 **SDA**


### Pines extra del BMP280 (si tu placa los tiene)

Muchos módulos BMP280 traen pines **CSB** y **SDO**:

- **CSB**: para I2C normalmente debe ir a **3V3** (en algunos módulos ya viene preparado, pero si no detecta el sensor, prueba a fijarlo a 3V3).
- **SDO**: selecciona la dirección I2C (típicamente `0x76` u `0x77`); a **GND** suele dar `0x76` y a **3V3** suele dar `0x77` (depende del módulo).

***

## 3) Software y librerías

Instala en Arduino IDE (Library Manager):

- Adafruit BMP280 Library
- ArduinoJson
- PubSubClient
- Para las características del ESP8266, como es el WiFi, tenemos que tener el core instalado en el IDE. Para hacer esto, tenemos que ir a "Archivos" o "File" y hacer click en "Preferencias" o "Preferences".
  
<img width="891" height="615" alt="Screenshot_2" src="https://github.com/user-attachments/assets/bf18bc3e-4550-4b64-a274-acbb32c2fe17" />

- Una ver ahí, tenemos que copiar en la casilla que se marca en el recuadro rojo las direcciones URLs a las placas que vamos a utilizar. Esto nos permite importar las características de los microcontroladores ESP8266 y el ESP32.
<img width="809" height="550" alt="Screenshot_3" src="https://github.com/user-attachments/assets/8a6d64c7-042d-4f54-9f75-841802146e36" />

Copia los siguientes enlaces y pégalos en el recuadro que te muestro en la imagen de arriba:

```
http://arduino.esp8266.com/stable/package_esp8266com_index.json
```
(Observa el icono con dos recuadros superpuestos, arriba a la derecha. Hacer click en este icono te permite copiar el contenido del cuadro gris, en este caso, la URL) 

***

## 4) Configuración rápida 

En la cabecera del código verás parámetros como estos (puedes cambiarlos por **tus propios valores**):

```cpp
#define WIFI_SSID     "TU_WIFI"
#define WIFI_PASSWORD "TU_PASS"

#define SET_SECURE_WIFI false

// Si SET_SECURE_WIFI == false:
#define MQTT_SERVER "136.112.103.14"
#define MQTT_PORT   1883
```

Qué significa cada uno:

- `WIFI_SSID` y `WIFI_PASSWORD`: el nombre y contraseña de la WiFi a la que se conectará el ESP8266 (pon la de tu casa o la del instituto).
- `MQTT_SERVER` y `MQTT_PORT`: la “dirección” y el puerto del servidor MQTT (broker) al que enviará los datos.

Otros parámetros útiles:

- `TYPE_NODE`: etiqueta para organizar topics (`meteorologia`, `aula1`, `grupo4`, etc.).
- `SEND_PERIOD_MS`: periodo de publicación (por defecto 5000 ms, o sea, cada 5 segundos).

Además, en la función `sendToBroker()` puedes cambiar el **topic de envío** para que identifique mejor al alumno o al grupo (por ejemplo, incluir `grupo4`, `alumno_carla`, `mesa2`, etc.) al final del topic. Así, cuando miréis los datos en Node-RED, sabréis de quién es cada sensor sin confusiones.

<img width="1083" height="732" alt="Screenshot_4" src="https://github.com/user-attachments/assets/e3a386e3-c3f0-42d9-b6ea-fbc342030d76" />




***

## 5) Topics MQTT y ejemplo de mensaje

### Topic de datos

El código publica en:

```
orchard/<TYPE_NODE>/<chipIdHex>/cjmcu8128
```

> Si tu proyecto es “solo BMP280”, puedes renombrar el sufijo `cjmcu8128` por `bmp280` en la función `sendToBroker()`.

### Ejemplo de payload JSON

```json
{
  "esp": { "ip": "10.0.0.15", "rssi": -67 },
  "sensor": "BMP280",
  "timestamp": 34795,
  "values": {
    "temperatura_c": 22.31,
    "presion_hpa": 1012.84,
    "altitud_m": 12.7
  }
}
```


### Topic de estado (conexión)

Se publica un estado retained en:

```
orchard/<TYPE_NODE>/ESP8266Client-<chipId>/connection
```

- Al conectar: `"Online"` (retained)
- Si cae sin desconectar bien: `"Offline"` (LWT, retained)[^1]

***

## 6) Estructura del código 
Funciones principales:

- `wifiConnect()`: conecta a WiFi (bloqueante con reintentos).
- `reconnectMQTT()`: conecta a MQTT y configura LWT + publica “Online”.
- `i2cScan()`: escáner I2C para verificar que el sensor responde (diagnóstico).
- `iniciaSensor()`: inicializa el BMP280 (0x76/0x77) y configura el muestreo.
- `sendToBroker()`: lee sensor → crea JSON → publica por MQTT.
- `loop()`: mantiene conexiones y ejecuta el envío periódico.

***

## 7) Cómo modificar sensores y enviar otros datos

La parte “educativa” del proyecto es que el alumnado pueda **cambiar el sensor** o **añadir más variables**. Estas son las zonas que deben tocar:

### 1) Añadir una nueva librería / objeto del sensor

- En la parte de `#include` (arriba).
- Crear una instancia global (igual que `Adafruit_BMP280 bmp;`).


### 2) Inicialización del sensor

- En `setup()` o en una nueva función tipo `iniciaSensorX()`.
- Patrón recomendado:
    - `i2cScan();` (si es I2C)
    - `sensor.begin(...)` y logs por Serial.


### 3) Lectura y publicación (la parte clave)

En `sendToBroker()`:

- Sustituir estas lecturas:

```cpp
float tempC   = bmp.readTemperature();
float pressHP = bmp.readPressure() / 100.0;
float altM    = bmp.readAltitude(SEALEVELPRESSURE_HPA);
```

- Y modificar el JSON en:

```cpp
JsonObject values = doc.createNestedObject("values");
values["temperatura_c"] = tempC;
values["presion_hpa"]   = pressHP;
values["altitud_m"]     = altM;
```


#### Ejemplo: añadir “luz” (LDR analógico) o “humedad”

- Lees el nuevo dato (p.ej. `int luz = analogRead(A0);`)
- Lo añades al JSON:

```cpp
values["luz_raw"] = luz;
```


### 4) Cambiar el topic de publicación (opcional)

También en `sendToBroker()`:

```cpp
String pub_topic = "orchard/" + TYPE_NODE + "/" + String(ESP.getChipId(), HEX) + "/bmp280";
```

Así cada sensor/proyecto queda bien organizado.

***

## 8) Node‑RED: ver datos y mandar órdenes al ESP8266

Node‑RED es una herramienta para crear “programas” uniendo **bloques** (nodos) con cables. En este proyecto lo usamos como un “panel de control”: por un lado **recibe** los datos que envía el ESP8266 por MQTT (en formato JSON) y los muestra en una web; por otro lado **envía** órdenes al ESP8266 (por ejemplo encender/apagar el LED) publicando en un topic de control.

<img width="1919" height="872" alt="Screenshot_1" src="https://github.com/user-attachments/assets/a8373c5f-c2e4-4b7e-b7e3-7f3eabf9c83d" />


### 8.1 Qué es MQTT en este proyecto

MQTT funciona como un sistema de mensajería con “canales” llamados **topics**. Un dispositivo *publica* mensajes en un topic (por ejemplo, datos del sensor) y otro dispositivo *se suscribe* a ese topic para recibirlos.

- Ejemplo de topic de datos: `.../bmp280` (aquí llegan temperatura y presión).
- Ejemplo de topic de control: `.../activate_led` (aquí mandas ON/OFF).

Al inicio del programa, vemos los topics a los que se van a enviar los datos y a los que nos hemos suscrito para poder controlar lo que queramos. Podemos crear el nuestro:

<img width="1299" height="301" alt="Screenshot_1" src="https://github.com/user-attachments/assets/659a3157-4d6a-453e-8d9d-cc889944397a" />

### 8.2 Nodos que verás en el flujo

#### `mqtt in` (recibir mensajes)

Este nodo se conecta al broker MQTT y se **suscribe** a un topic (un “canal”). Cuando llega un mensaje por ese canal, el nodo lo envía hacia la derecha para que el resto del flujo lo procese (por ejemplo, para leer el JSON y mostrarlo en el Dashboard).

**Un topic con `#` (comodín / “escuchar todo lo que cuelga de aquí”)**
La imagen del topic con `#` significa: “escucha **muchos topics a la vez** que empiezan igual”.

<img width="799" height="79" alt="Screenshot_2" src="https://github.com/user-attachments/assets/becd40f9-43ab-4e2b-9cc4-174ac6be759a" />

Ejemplo: si pones `cabrerapinto/#`, Node‑RED recibirá mensajes de **cualquier** subtopic que empiece por `cabrerapinto/…` (sirve para pruebas y depuración, porque ves todo lo que está enviando la red).

**Un topic más específico (escuchar solo un dispositivo o un sensor)**
En la imagen del topic específico se ve un camino más largo (con varias carpetas) que identifica mejor el origen: zona/proyecto → tipo de nodo → ID del dispositivo → sensor.

<img width="1455" height="337" alt="Screenshot_3" src="https://github.com/user-attachments/assets/b0e56fd3-da4e-4c0c-a8f6-1673ca40f341" />

Esto hace que Node‑RED reciba **solo** los mensajes de ese sensor/dispositivo concreto (por ejemplo, el BMP280 de un ESP en particular), evitando mezclar datos de otros compañeros.

**Configuración del nodo `mqtt in` (elegir servidor y topic correcto)**
En la ventana de configuración se hacen dos cosas clave:

<img width="717" height="616" alt="Screenshot_4" src="https://github.com/user-attachments/assets/d265b629-e082-48d8-a15e-58c31b4e2064" />

1) **Servidor (broker)**: eliges el “servidor de mensajes” al que te conectas (tiene IP y puerto). Si eliges otro broker, no verás tus mensajes.
2) **Topic**: copias/pegas el topic exacto desde el que quieres recibir datos (en tu caso, el que publica tu ESP). Importante: cada alumno/grupo puede tener un topic distinto, así que hay que mirar el vuestro y poner ese.



#### `debug` (ver qué está llegando)

El nodo `debug` muestra en la barra derecha el contenido del mensaje (por ejemplo `msg.payload`). Es el mejor nodo para comprobar si el JSON llega bien y si el topic es el correcto.

<img width="1914" height="675" alt="Screenshot_5" src="https://github.com/user-attachments/assets/d36b938e-a2be-42f6-af77-7592bcdaa76c" />

Durante las prácticas conviene activar el debug al principio, haciendo click en el botón verde que se muestra en la imagen, luego desactivarlo para que no se llene la pantalla de depuración.

#### `function` (extraer un dato del JSON)

El nodo `function` te deja escribir un poco de JavaScript para transformar el mensaje. 

<img width="1569" height="383" alt="Screenshot_6" src="https://github.com/user-attachments/assets/276c2152-20ee-4ba0-a69a-3579525f934a" />

Aquí lo usamos para “pasar de JSON completo a un dato”, por ejemplo quedarnos solo con la temperatura. 

<img width="676" height="708" alt="Screenshot_8" src="https://github.com/user-attachments/assets/17d0ebe5-fbeb-49db-8c00-80c6bb32a13a" />

La idea es simple: el JSON llega en `msg.payload`, tú lees la ruta del dato y después pones ese dato como nuevo `msg.payload` (así el siguiente nodo recibe solo un número).

Ejemplo típico (temperatura BMP280):

```js
var p = msg.payload;              // JSON completo
msg.payload = p.sensor.bmp280.t_c; // sacar temperatura
return msg;
```

A continuación, vamos a extraer la temperatura del mensaje JSON, paso a paso:

**1) Llega un mensaje MQTT con un JSON** dentro de `msg.payload`. Un ejemplo simple puede ser:

```json
{
  "esp": { "ip": "10.53.151.83", "rssi": -62 },
  "sensor": {
    "bmp280": { "t_c": 22.45, "p_hpa": 1012.80 }
  }
}
```

**2) Paso recomendado: mira el JSON con un `debug`**
Conecta un nodo `debug` justo después del `mqtt in` y expande el árbol: `payload → sensor → bmp280 → t_c`. El panel Debug te ayuda a entender la estructura del mensaje.

**3) En un nodo `function`, guarda el JSON en una variable**

```js
var p = msg.payload;   // p ahora es el JSON completo
```

**4) Escribe la “ruta” hasta el dato que quieres**

- La temperatura está en: `p.sensor.bmp280.t_c`

```js
var temp = p.sensor.bmp280.t_c;
```

**5) Deja ese valor como `msg.payload` para el siguiente nodo (gauge/texto)**

```js
msg.payload = temp;
return msg;
```

**Código completo del `function` (temperatura):**

```js
var p = msg.payload;
msg.payload = p.sensor.bmp280.t_c;
return msg;
```

> Si la ruta no existe (por ejemplo el JSON no trae `bmp280`), verás `undefined`. En ese caso revisa la estructura en el `debug` y ajusta la ruta.

#### `ui_text` y `ui_gauge` (mostrarlo en una web: Dashboard)

Estos nodos pertenecen al **Dashboard**, que es una página web (tipo “panel de control”) donde ves los datos del ESP8266 en tiempo real: si el ESP publica un nuevo valor por MQTT, aquí se actualiza automáticamente.

<img width="1919" height="878" alt="Screenshot_10" src="https://github.com/user-attachments/assets/ccf257fb-0afe-4be1-be63-dafaf1fc131c" />

En esta captura se ve el resultado final: varios “widgets” (bloques) colocados en la web. Normalmente:

- Arriba aparecen textos con información del ESP (por ejemplo **IP**, **RSSI**, o “Online/Offline”).
- Abajo aparecen medidores tipo reloj/contador para los sensores (por ejemplo **Temperatura** y **Presión**).

La idea es: Node-RED recibe el JSON, extrae valores, y el Dashboard los muestra de forma visual para que sea fácil entenderlos de un vistazo.

- `ui_text`: muestra texto (por ejemplo IP, RSSI o “Online/Offline”).
- `ui_gauge`: muestra un número como un medidor (por ejemplo temperatura o presión).

Esta ventana es donde decides **cómo y dónde** se verá el dato en la web.

<img width="626" height="814" alt="Screenshot_11" src="https://github.com/user-attachments/assets/e512e9e1-dced-42aa-9725-67f93915c458" />

Cosas importantes que hay que entender:

- **Group (Grupo):** es “la caja” o sección del Dashboard donde se colocará el widget.
Piensa en el Dashboard como una página dividida en bloques grandes (grupos). Si eliges otro grupo, el widget aparecerá en otra zona.
- **Label (Etiqueta/Título):** el nombre que verás al lado o encima del widget (por ejemplo “IP”, “RSSI”, “Temperatura”).
- **Format (Formato del valor):** es cómo Node-RED “imprime” el valor que le llega en `msg.payload`.
    - En `ui_text` suele ponerse `{{msg.payload}}` para mostrar el texto/número tal cual llega.
    - En `ui_gauge` normalmente se usa `{{value}}` (el gauge representa el valor numérico recibido).
Si el formato está mal, puede que no se vea nada o que salga “undefined”.
- **Units / min / max (en `ui_gauge`):** aquí defines la unidad (ºC, hPa…) y los límites del medidor. Esto **no cambia el dato real**; solo cambia cómo se representa en la web.

Aquí es donde se organiza toda la web del Dashboard.

<img width="388" height="475" alt="Screenshot_12" src="https://github.com/user-attachments/assets/f8f07f3d-e418-40f5-b27d-87793bd9e791" />


1. En Node-RED mira la barra lateral derecha.
2. Busca la sección de **Dashboard** (icono típico de “panel”).
3. Ahí verás la estructura: **Tabs** y **Groups**.

**Qué significa cada cosa:**

- **Tab (Pestaña):** como una pestaña en una web. Si tienes varias, puedes tener “Página principal”, “Grupo 1”, “Sensores”, etc. Cada tab es una “pantalla” distinta.
- **Group (Grupo):** son secciones dentro de una pestaña. Sirven para ordenar: por ejemplo un grupo para “Información del ESP”, otro para “BMP280”, otro para “Control LED”.
- **Link (Enlace):** un botón/enlace del Dashboard para saltar a otra pestaña o a otra parte. No es obligatorio para que funcione, pero ayuda a navegar.

**Reglas de clase (para no liarse con widgets)**

- **Regla 1:** Primero crea (o elige) un **Tab**, luego crea (o elige) un **Group**, y por último en cada `ui_*` selecciona su **Group**. Si no asignas Group, el widget no aparece.
- **Regla 2:** Un widget debe recibir **un solo valor**. Si quieres mostrar temperatura y presión, usa **dos** widgets (dos `ui_gauge`).
- **Regla 3:** Si no aparece nada, revisa en este orden: `mqtt in` (topic correcto) → `debug` (llega JSON) → `function` (ruta correcta) → `ui_*` (Group + Format correctos).

#### `inject`, `ui_switch` y `mqtt out` (mandar órdenes al ESP8266)

Esta parte sirve para controlar el LED del ESP8266.

<img width="1738" height="396" alt="Screenshot_13" src="https://github.com/user-attachments/assets/e17c4d3d-4880-428c-99c4-5d590c8ece4d" />


- `inject`: botones que envían un mensaje fijo, por ejemplo `"ON"` o `"OFF"`. Aquí podemos seleccionar el tipo de valor que "inyectamos" o enviamos al topic. En este caso, son valores boleanos, es decir, `true` o `false`, pero pueden ser valores numéricos, cadenas de texto...

<img width="693" height="783" alt="Screenshot_15" src="https://github.com/user-attachments/assets/7b634726-464a-4b08-9167-9f0103570371" />

- `ui_switch`: un interruptor en la web que envía `true/false` cuando lo cambias. Este objeto se crea en el dashboard y como el botón `inject`, permite enviar comandos cuando se activa.

<img width="867" height="682" alt="Screenshot_14" src="https://github.com/user-attachments/assets/fdb95f5b-923d-40ff-988b-591e7044be17" />

- `mqtt out`: publica ese mensaje en el topic de control (por ejemplo `activate_led` o un topic específico del dispositivo). Se configura como el nodo `mqtt in`


### 8.3 Cómo “entender” un flujo rápido

Para no perderse, sigue este orden:

- Empieza por los `mqtt in`: ¿qué topic están escuchando? (ahí entra la información).
- Activa un `debug`: mira qué hay dentro de `msg.payload` (ahí ves el JSON real).
- Revisa los `function`: cada uno debería sacar **un dato** y dejarlo como `msg.payload` para el Dashboard.
- Al final mira los nodos `ui_*`: eso es lo que verás en la web del Dashboard.


## Actividades sugeridas (aula)

- Cambiar `SEND_PERIOD_MS` y observar carga de red/broker.
- Añadir una variable nueva en `values` (y verla en el suscriptor MQTT).
- Calibrar altitud: ajustar `SEALEVELPRESSURE_HPA` y comprobar cambios.
- Diseñar un “dashboard” (Node-RED / Home Assistant / Grafana) con el topic.

***

## Resolución de problemas

- Si `i2cScan()` no encuentra `0x76` o `0x77`:
    - Revisa SDA/SCL, alimentación 3.3V y GND común.
- Si MQTT conecta pero no llegan datos:
    - Revisa `MQTT_SERVER`, puerto, y que el broker esté accesible desde la WiFi.
- Si el JSON es grande y falla `publish()`:
    - Asegura `client.setBufferSize(1024);` y reduce campos si hace falta.

***
