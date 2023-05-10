# Sleep_ESP_NOW
Joint venture  Attiny85 + ESP8266 ESP-NOW
Este proyecto tiene como finalidad utilizar las posibilidades del protocolo ESP-NOW para enviar información entre 2 modulos de Espressif, un receptor que será un ESP32 y un mando transmisor que consta de un ESP8266 y un Attiny85.

El módulo receptor, ESP32, no va a tener restricciones en cuanto al consumo energetico, pero el mando se va a alimentar mediante 3 pilas de 1,5 voltios y por tanto es imperativo que se gestione correctamente el consumo.

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/fad429e5-4689-4e93-8c0e-9ef0d2e2f253)


He comprobado, que el ESP8266 (Wemos D1 Mini), en su modo mas sostenible y amigable con el planeta, no baja de 5 miliamperios, y supongo que aunque duerma la CPU, quedan funcionando el CH340 o similar  (integrado que se encarga de dar vida al USB) y el regulador de tensión.

Podría desoldar dichos integrados, pero las posibilidades de reprogramación futuras practicamente desaparecen...

Por tanto, he optado por utilizar un Attiny85 para gestionar el encendido del Wemos D1 Mini.
El funcionamiento del sistema es el siguiente: 
- Cuando se detecta la pulsación de alguna de las teclas, el Attiny85 se activa y, a su vez, activa el transistor 2N7000 que encenderá el Wemos D1 Mini
- Una vez en marcha este, lo primero que hace es decirle al Attiny85 que le mantenga la alimentación para poder enviar el comando. Esto se hace activando el puerto D2 del Wemos que esta conectado al puerto PB3 del Attiny.
- Acto seguido envia el comando correspondiente a la tecla pulsada, mediante el protocolo ESP-NOW hacia el ESP32 y posteriormente desactiva D2, por lo que el Attiny85 proceder a cortarle el suministro electrico (como una compañia electrica al uso por falta de pago ;)
- Y asi nos quedamos hasta la siguiente pulsación.

Este es el esquema del mando:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/edce0c1c-96fc-484a-8513-eee2198969b2)

Se puede ver que los pulsadores conectados tanto al Attiny85 como al ESP8266: Al primero lo despiertan y al segundo le dicen que comando tiene que enviar, en función de la tecla pulsada.

Y esta es la gráfica del consumo en modo Sleep:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/666f8815-a606-4857-ac3a-49e6ca002ea4)

Con este consumo la duración se extenderá a años.

Consumo durante la TX de comando ESP NOW:

![image](https://github.com/redmilenium/Sleep_ESP_NOW/assets/48222471/7cd5b9cb-edff-48c3-9996-445df605089b)

Los picos de consumo llegan a los 200 mA, pero durante un muy breve espacio de tiempo.


