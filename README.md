# PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO

En este repositorio se muestra el proyecto del Modulo V "Interfases persona-máquina (HMI)" del Diplomado Automatizacion Industrial y Mecatronica, el cual consiste en un **Sistema de riego automatizado**, utilizando las herramentas de NodeRed, WOKWI y MySQL


## Introducción

Para la propuesta de este sistema de riego utilizaremos el modelo de placa arduino ESP32, el cual es una placa electrónica en la cual viene montado un microcontrolador con todo lo necesario para realizar su programación, este sistema se enfoca en acercar y facilitar el uso de la tecnología electrónica y programación de sistemas para una diversidad de tareas. En nuestro caso se trata de un sistema de riego el cual volverá más eficaz y aumentara el nivel de productividad en el trabajo realizado con el riego y el monitoreo de la cantidad de agua que este almacenada para poder hacer la funcion.


### Descripción

La ```Esp32``` la utilizamos en un entorno de adquision de datos, lo cual en esta practica ocuparemos un sensor (```DTH22```) para la obtención de datos de temperatura y humedad, ademas de un sensor (```Ultrasonico HC-SR04```)  medicion de distancia, que en este caso se usara para medir la capacidad de agua alamacenada; Esta practica se usara el simulador llamado [WOKWI](https://wokwi.com/), junto con NodeRed y MySQL.


## Material Necesario

Para realizar este flow necesitas lo siguiente

- [Node.js](hhttps://nodejs.org/en)
- [WOKWI](https://https://wokwi.com/)
- [MySQL](https://www.apachefriends.org/)
- Tarjeta ESP 32
- Sensor DHT22
- Sensor ultrasonico HC-SR04
- Motor 
  

## Instrucciones

### Requisitos previos

Se debe cumplir con los siguientes requisitos previos
1. Para realizar este proyecto de este repositorio se necesita entrar a la plataforma [WOKWI](https://https://wokwi.com/).
2. Tener instalado Node.js (version 20.11.0 LTS).
3. Crear una base de datos de MySQL. [XAMPP](https://www.apachefriends.org/)

### Instrucciones de preparación del entorno en wokwi

1. Abrir la terminal de programación y colocar la siguente codigo:

```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int Ap = 12;    // Pin para A+
const int Am = 13;   // Pin para A-
const int Bp = 14;    // Pin para B+
const int Bm = 27;   // Pin para B-
volatile int velocidad; //Variable de velocidad
volatile int dt;    //Variable de Delay
int L;    //Variable para el calculo de litros
const int DHT_PIN = 15;
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 2;   //Pin digital 3 para el Echo del sensor
DHTesp dhtSensor;
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "18.193.219.109";
String username_mqtt="educatronicosiot";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0

  pinMode(Ap, OUTPUT);  //Pin como salida
  pinMode(Am, OUTPUT);  //Pin como salida
  pinMode(Bp, OUTPUT);  //Pin como salida
  pinMode(Bm, OUTPUT);  //Pin como salida
  velocidad = 100;  //Velocidad de la bomba
  dt=(600/(4*abs(velocidad)));  //Delay entre cada paso
  Serial.println(velocidad);
}

void loop() {


delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
long t; //timepo que demora en llegar el eco
long d; //distancia en centimetros

digitalWrite(Trigger, HIGH);
delayMicroseconds(10);          //Enviamos un pulso de 10us
digitalWrite(Trigger, LOW);
  
t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
d = t/59;             //escalamos el tiempo a una distancia en cm
L = ((400-(d))*200*200)/1000; //Conversión de los valores de "d" a Litros

if (data.humidity>=0 && data.humidity<=50){   //Condición para accionamiento de bomba de agua              
  if (d>=0 && d<=350){    //Valores de niel de agua entre los que la bomba de agua funcionara
    delay(dt);
    digitalWrite(Bm,LOW) ; digitalWrite(Ap,HIGH);
    delay(dt);
    digitalWrite(Ap,LOW) ; digitalWrite(Bp,HIGH);
    delay(dt);
    digitalWrite(Bp,LOW) ; digitalWrite(Am,HIGH);
    delay(dt);
    digitalWrite(Am,LOW) ; digitalWrite(Bm,HIGH);
  }
}
else if (data.humidity=100){    //A un 100% de humedad la bomba de agua se apagará
  digitalWrite(Bm,LOW) ; digitalWrite(Ap,LOW);
  digitalWrite(Ap,LOW) ; digitalWrite(Bp,LOW);
  digitalWrite(Bp,LOW) ; digitalWrite(Am,LOW);
  digitalWrite(Am,LOW) ; digitalWrite(Bm,LOW);
}
  
Serial.print("Distancia: ");
Serial.print(d);      //Enviamos serialmente el valor de la distancia
Serial.print("cm");
Serial.println();
Serial.print("Nivel: ");
Serial.print(L);      //Enviamos serialmente el valor de los litros
Serial.print("L");
Serial.println();
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    doc["LITROS"] = String(L);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("AbrahamDHT", output.c_str());
  }
}
```
**NOTA**:En las siguientes partes que se mostraran a continuación se debe de realizar un pequeño cambio a base de como es configurado en NodeRed
```
const char* mqtt_server = "18.193.219.109";  **se colocara ip que se encuentra en mqtt in del NodeRed **
String username_mqtt="educatronicosiot";
```

```
client.publish("AbrahamDHT", output.c_str());  **Aqui va el Topic que igual corresponde al mqtt in** 

```

2. Instalar la libreria de **ArduinoJson** **DTH sensor library for ESPx** **PubSubClient** **AccelStepper** **Stepper** .

   - Seleccionar pestaña de Librery Manager --> Add a New library --> Colocamos el nombre de libreria
 ![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Librerias.jpeg)
  
4. Realizar la conexion de **DTH22** **HC-SR04** **MOTOR** con la **ESP32** de la siguiente manera.
 
 ![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Conexiones.jpeg)
     
  **Conexión DTH22**
  -VCC --> GND
  -SDA --> esp:15
  -NC 
  -GND  --> VCC

  **Conexión HC-SR04**
  -GND --> GND
  -ECHO --> esp: 2
  -TRIG --> esp: 4
  -VCC --> 5V

**Conexión Motor**
  -Stepeper1:A- --> esp: 13
  -Stepeper1:A+ --> esp: 12 
  -Stepeper1:B+ --> esp: 14 
  -Stepeper1:B- --> esp: 27 

### Instrucciones de preparación del entorto MySQL

1. Una vez intalado el xampp nos vamos a recien agregados en nuestra PC, abrimos como administrador --> se abrira un recuadro en la cual damos click en start de Apache y MySQL -->  damos click en admin del apartado de MySQL ( nos redigira a una pagina (http://localhost/phpmyadmin/))

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/XAMPP.jpeg)

3. En la parte izquierda de la pantalla vermos unas opciones, le damos click en nuevo --> En nombre de base se le coloca lo que uno desee en nuestro caso se llama practica 1 --> crear

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/nuva%20base%20de%20datos.jpeg)
   
5. Una vez creada nos mandara a colocar nombre de la tabla --> numero de columnas seleccionamos 5 --> crear

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/nombre%20base%20de%20datos.jpeg)
  
7. Editamos las columnas que creamos de la siguiente manera
   - Columna 1
   - Columna 2
   - Columna 3
   - Columna 4
   - Columna 5
  
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/configuracion%20columnas.jpeg)

8. Para la base de datos que se genero ponemos a trabajar el proyecto de WOKWI y en la plataforma **phpMyadmin**  podremos observar la informacion generada y se observa de la siguiente manera 
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/base%20de%20datos.jpeg)


### Instrucciones de preparación del entorno en NodeRed

Para ejecutar este proyecto, es necesario lo siguiente
1. Arrancar el contenedor de NodeRed en **cmd** con el comando
        
        node-red

2. Dirigirse a [localhost:1880](localhost:1880)

3. En la parte izqierda de la pantalla seleccionaremos
   - MQTT
   - json
   - 5 fuction
   - 3 gauge
   - 3 chart
   - inject
   - text
   - data picker
     
 ![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Nodos.jpeg)
  
4. Para la configuracion del mqtt in necesitaremos saber nuestra ip, que se saca de la siguiente manera:
   cmd --> nslookup broker.hivemq.com --> copiamos los numeros de la parte que dice addresses --> nos dirigimos a nodered --> seleccionamos mqtt in --> damos click en el icono del lapiz --> en la parte de server se pegara la direccion ip que copiamos --> update --> done

![](https://github.com/YasminZagal/PRACTICA-N-8-NodeRed-CON-DHT22/blob/main/direccion%20ip.png)   

![](https://github.com/YasminZagal/PRACTICA-N-8-NodeRed-CON-DHT22/blob/main/conf1.png)  

5. En la configuracion del **JSON** lo seleccionamos y en la parte de **action** se despelgaran unas opciones en donde colocaremos **always convert JavaScript Object**

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/configuracion%20json.jpeg)

6. En el primer function se colocara el mobre de **TEMPERATURA** y posteriormente se pondra el siguiente codigo

```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/nodo%20temperatura.jpeg)

7.En el segundo fuction se llamara **HUMEDAD** e igualmente se colocara un codigo

```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/nodo%20humedad.jpeg)

8. En el terecer fuction se llamara **DISTANCIA** que coresponde al nivel de agua en LITROS

```
msg.payload = msg.payload.LITROS;
msg.topic = "LITROS";
return msg;
```
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/nodo%20litros.jpeg)

9. Para el apartado del MySQL se requiere de colocar el siguiente codigo

```
var query = "INSERT INTO `practica 1`(`ID`, `FECHA`, `DEVICE`, `TEMPERATURA`, `HUMEDAD`) VALUES (NULL, current_timestamp(), '";
query = query+msg.payload.DEVICE + "','";
query = query+msg.payload.TEMPERATURA + "','";
query = query+msg.payload.HUMEDAD + "');'";
msg.topic=query;
return msg;

```
**NOTA**: En el siguiente apartado en donde dice **practica 1** se le cambiara el nombre dependiendo de como se llame su servidor creado en MySQL
```
var query = "INSERT INTO `practica 1`(`ID`, `FECHA`, `DEVICE`, `TEMPERATURA`, `HUMEDAD`) VALUES (NULL, current_timestamp(), '";
```
10. En el nodo de mysql le damos click nos vamos a la opcion del lapiz en la opción de Name y Database colocaremos el nombre con el que fue creado la base de datos. En la parte de Host se le coloca 127.0.0.1 en nuestro caso y por último en User pondremos root

![]()

11. En la parte de dashboard se agrega una nueva tabla con el nombre de SISTEMA DE RIEGO (HMI), posteriormente se añaden 5 grupos con los nombre de

    -NIVEL DE AGUA
    -INDICADORES
    -GRAFICAS
    -Sistema de riego.
    
12. En los nodo de gauge de los tres functions se colocara en el grupo de indicadores y en el chart de los  function sus nodos corresponderan al grupo  graficas

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/chart%20humedad.jpeg)

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/chart%20temperatura.jpeg)

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/gauge%20temperatura.jpeg)

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/gauge%20humedad.jpeg)

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/gauge%20agua.jpeg)



   
### Instrucciones de operación
1. Nos vamos a nuestro wokwi en donde nuestro simulador ya esta corriendo

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/1Resultado.jpg?raw=true)

1. Nos vamos a nuestro nod-red en donde nos queda de la siguiente manera
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Resultado.jpg?raw=true)

## Resultados

A continuación se puede observar una vista previa del resultado del proyecto y la interacciones con las diferentes plataformas usadas .

Interaccion Temperatura, Humedad y Distancia 
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/ResultadoH.jpg?raw=true)

![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/ResultadoD.jpg?raw=true)

Resultado Node-Red
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Resultado2.jpg?raw=true)

Resultado Xampp
![](https://github.com/YasminZagal/PROYECTO-FINAL-DIPLOMADO-SISTEMA-DE-RIEGO-AUTOMATIZADO/blob/main/Resultado1.jpg?raw=true)

## Conclusiones
Esta propuesta de proyecto ha cumplido con sus objetivos, habiendo sido creado con el fin de desarrollar un sistema de riego automático administrado de manera inalámbrica que satisfaga las necesidades del usuario, demostrando por medio de un módulo poder conseguir un óptimo manejo de los recursos de la misma y así llegar a conseguir  una producción de excelente calidad, siendo de gran beneficio a la comunidad, promoviendo y ayudando a preservar los recursos naturales del lugar.
Se puede afirmar que, respecto al sondeo inicial, se mejoraron las habilidades y conocimientos en los temas de electrónica y programación y se cuentan con las bases para desarrollar otros proyectos. 

### Créditos
 # Desarrollado por 
 
 - Abraham Contreras Herrera
 - [GitHub](https://github.com/AbrahamCH1)
 
 - Armenta Ocampo Daniel de Jesus
 - [GitHub](https://github.com/DanielX834)
 
 - Nestor Ivan Jimenez Sanchez
 - [GitHub](https://github.com/nijs17)
 
 - Zagal Hernández Ali Yasmin
 - [GitHub](https://github.com/YasminZagal)
 
 




