# Sleep_ESP_NOW
Joint venture  Attiny85 + ESP8266 ESP-NOW
Este proyecto tiene como finalidad utilizar las posibilidades del protocolo ESP-NOW para enviar información entre 2 modulos de Espressif, un receptor que será un ESP32 y un mando transmisor que consta de un ESP8266 y un Attiny85.

El módulo receptor, ESP32, no va a tener restricciones en cuanto al consumo energetico, pero el mando se va a alimentar mediante 3 pilas de 1,5 voltios y por tanto es imperativo que se gestione correctamente el consumo.

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/fad429e5-4689-4e93-8c0e-9ef0d2e2f253)


He comprobado, que el ESP8266 (Wemos D1 Mini), en su modo mas sostenible y amigable con el planeta, no baja de 5 miliamperios, y supongo que aunque duerma la CPU, quedan funcionando el CH340 o similar  (integrado que se encarga de dar vida al USB) y el regulador de tensión. Y en algun caso ademas, algun led que indica que esta alimentado.

Podría desoldar dichos integrados, pero las posibilidades de reprogramación futuras practicamente desaparecen...

Por tanto, he optado por utilizar un Attiny85 para gestionar el encendido del Wemos D1 Mini.
El funcionamiento del sistema es el siguiente: 
- Cuando se detecta la pulsación de alguna de las teclas, el Attiny85 se activa y, a su vez, activa el transistor 2N7000 que encenderá el Wemos D1 Mini
- Una vez en marcha este, lo primero que hace es decirle al Attiny85 que le mantenga la alimentación para poder enviar el comando. Esto se hace activando el puerto D2 del Wemos que esta conectado al puerto PB3 del Attiny.
- Acto seguido envia el comando correspondiente a la tecla pulsada, mediante el protocolo ESP-NOW hacia el ESP32 y posteriormente desactiva D2, por lo que el Attiny85 proceder a cortarle el suministro electrico (como una compañia electrica al uso por falta de pago ;)
- Y asi nos quedamos hasta la siguiente pulsación.

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

Programa del Attiny85

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

ISR(PCINT0_vect) 
{
    digitalWrite(PB4, LOW);
  if (digitalRead(PB0) == LOW)           // # PB0 = pin 5 pressed => LED on
    digitalWrite(PB4, HIGH);
  else if (digitalRead(PB1) == LOW)      // # PB1 = pin 6 pressed => LED off
    digitalWrite(PB4, HIGH);
  else if (digitalRead(PB2) == LOW)      // # PB1 = pin 6 pressed => LED off
    digitalWrite(PB4, HIGH);
  else if (digitalRead(PB3) == LOW)      // # PB1 = pin 6 pressed => LED off
    digitalWrite(PB4, LOW);
}

void setup() {  
  pinMode(PB4,OUTPUT); // mosfet
  pinMode(PB0,INPUT);  //tecla 1
  pinMode(PB1,INPUT);  // tecla 2
  pinMode(PB2,INPUT); // tecla 3
  pinMode(PB3,INPUT); // sigue encendido

  ADCSRA = 0; // ADC disabled
  GIMSK = 0b00100000;  
  PCMSK = 0b00001111;  
} 

void loop() 
{
  sleep_enable();
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);  
  sleep_cpu(); 
 
}

```

Programa instalado en el ESP8266 (WEmos D1 Mini)

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
