# Sleep_ESP_NOW
Joint venture  Attiny85 + ESP8266 ESP-NOW
Este proyecto tiene como finalidad utilizar las posibilidades del protocolo ESP-NOW para enviar información entre 2 modulos de Espressif, un receptor que será un ESP32 y un mando transmisor que consta de un ESP8266 y un Attiny85.

El ESP-NOW (ver caracteristicas en https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html) utiliza la frecuencia de 2.4 Ghz y multiples caracteristicas de seguridad que lo hacen muy superior al protocolo utilizado en la inmensa mayoria de los mandos que utilizan 433 Mhz. 

No es raro ver como un toldo se sube o se baja sin que el propietario hay pulsado nada, o que se abra la puerta de un garaje sin razón aparente.

Los mandos de bajo coste suelen utilizar comandos muy pequeños en número de bits y en una frecuencia muy saturada, y en momentos determinados, por ejemplo, un sensor de temperatura exterior, puede enviar a la base la información de temperatura y coincidir con el comando de bajar un toldo  del vecino.

El módulo receptor, ESP32, no va a tener restricciones en cuanto al consumo energetico, pero el mando se va a alimentar mediante 3 pilas de 1,5 voltios y por tanto es imperativo que se gestione correctamente el consumo.

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/fad429e5-4689-4e93-8c0e-9ef0d2e2f253)


He comprobado, que el ESP8266 (Wemos D1 Mini), en su modo mas sostenible y amigable con el planeta, no baja de 5 miliamperios, y supongo que aunque duerma la CPU, quedan funcionando el CH340 o similar  (integrado que se encarga de dar vida al USB) y el regulador de tensión. Y en algun caso, dependiendo del fabricante del modulo, algun led que indica que esta alimentado.
Esto limita drasticamente las posibilidades de alimentarlo con pilas.
Podría desoldar dichos integrados, pero las posibilidades de reprogramación futuras practicamente desaparecen...

Gráfica del consumo de un Wemos D1 Mini en sleep (ESP.deepSleep() alimentandolo a 3V3 (por el pin de 3V3) : 

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/64c9ec1a-62e4-44f8-9d3d-20dacdd93161)

Gráfica del consumo de un Wemos D1 Mini en sleep (ESP.deepSleep() alimentandolo a 5V (por el pin de 5V): 

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/438dcc96-28a3-4150-bb73-f9f85b73c0fb)


Por tanto, he optado por utilizar un Attiny85 para gestionar el encendido del Wemos D1 Mini.
El funcionamiento del sistema es el siguiente: 
- Cuando se detecta la pulsación de alguna de las teclas, el Attiny85 se activa y, a su vez, activa el transistor 2N7000 que encenderá el Wemos D1 Mini
- Una vez en marcha este (el Wemos D1 Mini),  lo primero que hace es decirle al Attiny85 que le mantenga la alimentación para poder enviar el comando. Esto se hace activando el puerto D2 del Wemos que esta conectado al puerto PB3 del Attiny.
- Acto seguido envia el comando correspondiente a la tecla pulsada, mediante el protocolo ESP-NOW hacia el ESP32 y posteriormente desactiva D2, por lo que el Attiny85 proceder a cortarle el suministro electrico (como una compañia electrica al uso por falta de pago ;)
- Y asi nos quedamos hasta la siguiente pulsación.

La utilización de un micro como el Attiny85 simplifica, tanto en tamaño como en número de componentes, la implementación de un sistema que permite la lectura de 3 teclas, la puesta en marcha de un sistema principal, y la posibilidad de que ese sistema principal decida cuando quiere ser desconectado (Auto Power OFF). 

Este es el esquema del mando:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/edce0c1c-96fc-484a-8513-eee2198969b2)

Se puede ver que los pulsadores estan conectados tanto al Attiny85 como al ESP8266: Al primero lo despiertan y al segundo le dicen que comando tiene que enviar en función de la tecla pulsada.

Y esta es la gráfica del consumo en modo Sleep:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/666f8815-a606-4857-ac3a-49e6ca002ea4)

Con este consumo la duración de las pilas se extenderá a años.

Consumo durante la TX de comando ESP NOW:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/7cd5b9cb-edff-48c3-9996-445df605089b)

Los picos de consumo llegan a los 200 mA, pero durante un muy breve espacio de tiempo.

Por tanto, tenemos 3 programas en funcionamiento:
- El del Attiny85
- El del ESP8266 (Wemos D1 Mini)
- El del ESP32 (ESP32-DevKit)

Programa en el Attiny85

```
/* PRUEBA DE SLEEP EN ATTINY85
   pb0  input teclado
  pb1  input teclado 
  pb2  input teclado
  pb3  input no te pares please
  pb4  gate mosfet
*/
#include <Arduino.h>
#include <avr/interrupt.h>
#include <avr/sleep.h>
volatile bool capasao=0;


ISR(PCINT0_vect) 
{
  capasao=1;
}


void setup() 
{  

  DDRB  =  0b00010000; //PB4 OUT EL RESTO INPUT
  PORTB =  0b11100000; // out 0 de pb0 to pb4
  ADCSRA = 0; // ADC disabled
  PCMSK = 0b00000111;  // interrupciones para pb0 - pb1 - pb2
  GIMSK =  0b00100000;  // General Interrupt Mask Register, / Bit 5 – PCIE: Pin Change Interrupt Enable / When the PCIE bit is set (one) 
} 

void loop() 
{

   if(capasao)
     {
      GIMSK = 0b00000000;  // desactivo las interrupciones
      PORTB =  0b11110000; //  activo el mosfet
      capasao=0;           // 
      delay(100);
         
      while ((!(PINB  & 0b00000001))|| (!(PINB >>1 & 0b00000001))||(!(PINB >>2 & 0b00000001)))
        {
         delay(1);
        }
       PORTB =  0b11100000;  // desactivo el mosfet
       GIMSK = 0b00100000; // activo las interrupciones
   }
          
         // llevo a sleep al attiny85
         sleep_enable();
         set_sleep_mode(SLEEP_MODE_PWR_DOWN);
         sleep_cpu();
         

}
```

Programa en el ESP8266 (WEmos D1 Mini)

```
/**
 * ESP-NOW
 * 
*/
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <espnow.h>

// Mac address of the slave
uint8_t peer1[] = {0xd8, 0xfd, 0x9d, 0xf5, 0x08, 0xe8}; // aqui pon la mac del otro extremo

struct message 
{
   int boton;
};
struct message myMessage;

void onSent(uint8_t *peer1, uint8_t sendStatus) 
{

  if(sendStatus) Serial.println(" error en el envio");
  if(!sendStatus) Serial.println(" envio correcto");
}

void setup() 
{
 // activo D2 para que el Attiny85 no me corte la luz
 pinMode(D2,OUTPUT);
 digitalWrite(D2,HIGH);
 
 pinMode(D5,INPUT);
 pinMode(D6,INPUT);
 pinMode(D7,INPUT);
 
 // compruebo que tecla esta pulsada y lo guardo para enviar
 
 if(!digitalRead(D5))myMessage.boton=3;
 if(!digitalRead(D6))myMessage.boton=2;
 if(!digitalRead(D7))myMessage.boton=1;

 WiFi.mode(WIFI_STA);

  if (esp_now_init() != 0) 
  {
    return;
  }
  
  //configuro ESP NOW 
  esp_now_set_self_role(ESP_NOW_ROLE_CONTROLLER);
  
  // añado a quien le voy a enviar el comando
  esp_now_add_peer(peer1, ESP_NOW_ROLE_SLAVE, 1, NULL, 0);
 
  esp_now_register_send_cb(onSent);
}
void loop() 
{

  if(myMessage.boton!=0)esp_now_send(NULL, (uint8_t *) &myMessage, sizeof(myMessage));
  delay(100);
  digitalWrite(D2,LOW);
}

```
Programa en el ESP32 (ESP32-DevKit)

```
#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
 
struct message 
{
   int boton;
};
struct message myMessage;
 
void onDataReceiver(const uint8_t * mac, const uint8_t *incomingData, int len) {

  digitalWrite(LED_BUILTIN,HIGH);
  memcpy(&myMessage, incomingData, sizeof(myMessage));

 Serial.print("han pulsado el boton: ");
 Serial.println(myMessage.boton);
 delay(100);
   digitalWrite(LED_BUILTIN,LOW);
}
 
void setup() {
 Serial.begin(115200);
 WiFi.mode(WIFI_STA);
 esp_wifi_set_ps(WIFI_PS_NONE);
 pinMode(LED_BUILTIN,OUTPUT);
 
 // Get Mac Add
 Serial.print("Mac Address: ");
 Serial.print(WiFi.macAddress());
 Serial.println("ESP32 ESP-Now Broadcast");
 
 // Initializing the ESP-NOW
 if (esp_now_init() != 0) {
   Serial.println("Problem during ESP-NOW init");
   return;
 }
 esp_now_register_recv_cb(onDataReceiver);
}
 
void loop() 
{
 // pon aqui lo quieres hacer con la informacion recibida
}

```
