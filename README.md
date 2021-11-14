____
# ESPHome_Stairs
Контроллер освещения лестницы
____

# Возможности:
- Установка цвета и яркости светодиодной ленты
- Включение эффектов
- Работа с датчиками прохода 


Подсчитываем прохожих двумя vl53l0x в начале лестницы и двумя в конце.
Датчики распологаем на расстоянии 10-12 см друг от друга в подрозетниках.
Там же ставим Ардуино Нано для подсчета людей и отправки значений по rs485 в контроллер ленты.
По косоуру тянем провода:
2*2,5 - питание ленты, отходит к ступенькам в виде двух проводов в кабеле 2*0,75
2*0,75  DATA и CLK
2*0.75 RS485, соединяет нижний датчик с Контроллером ленты и верхним датчиком
 
Размечаем на лестнице место монтажа ленты
От края ленты отступаем по сантиметру в каждую сторону по 2 см на провода, то есть всего на 4 см больше
Сверлим отверстия, протягиваем кабель




Есть 2 алгоритма:

```
// Алгоритм подсчёта следующий:
// чтобы проход человека был засчитан, необходима
// последовательность срабатывания датчиков
// 00
// 10
// 11
// 01
// 00 в "прямом" проходе
// и
// 00
// 01
// 11
// 10
// 00 в "обратном" проходе
// Все прочие последовательности не засчитываются.
// Продолжительность прохода не учитывается, можно стоять в дверях 
// сколь угодно долго, а затем вернуться, либо пройти,
// проход будет незасчитан/засчитан соответственно. 
#define DEBUG 1    
//
#include <Wire.h>
#include <VL53L0X.h>
//
#define VL53L0X_XSHUT1_PIN    7 // Эти пины понадобятся для изменения адреса второго VL53L0X с 41 на 42
#define VL53L0X_XSHUT2_PIN    8 // Эти пины понадобятся для изменения адреса второго VL53L0X с 41 на 42
//
#define VL53L0X1_ADDRESS     41 // строка просто для информации, дефолтный адрес VL53L0X. 
#define VL53L0X2_ADDRESS     42 // это новый адрес второго датчика
VL53L0X VL53L0X_1;
VL53L0X VL53L0X_2;
#define minDistance         600 // ширина дверного проёма у меня 70 с чем-то см, срабатывание датчиков будем засчитывать, если измерено расстояние менее 60-ти см.
bool VL53L0X_1_state = false;   // флаг срабатывания первого датчика
bool VL53L0X_2_state = false;   // флаг срабатывания второго датчика
bool VL53L0X_1_already = false; // флаг "первый датчик уже сработал" 
bool VL53L0X_2_already = false; // флаг "второй датчик уже сработал" 
bool VL53L0X_together = false;  // флаг "оба датчика уже сработали" 
byte VL53L0X_first = 0;         // принимает значения 0,1,2 , показывает, какой из датчиков сработал первым
int HumanCounter = 0;
int OldHumanCounter = 0;

void setup()
{
/////////// На старте сразу "гасим" пины XSHUT обоих датчиков
pinMode(VL53L0X_XSHUT1_PIN, OUTPUT);
pinMode(VL53L0X_XSHUT2_PIN, OUTPUT);
//
#ifdef DEBUG
 Serial.begin(9600);
 Serial.println("");
 Serial.println(F("Two VL53L0X HumanCounter."));
#endif  
Wire.begin();
/////////// Меняем адрес второго датчика и "поднимаем" оба датчика
pinMode(VL53L0X_XSHUT2_PIN, INPUT);
delay(10);
VL53L0X_2.setAddress(VL53L0X2_ADDRESS);
pinMode(VL53L0X_XSHUT1_PIN, INPUT);
delay(10);
/////////// Запускаем оба VL53L0X
VL53L0X_1.init();
VL53L0X_2.init();
VL53L0X_1.setTimeout(250);
VL53L0X_2.setTimeout(250);
// To use continuous timed mode instead, provide a desired inter-measurement period in ms.
VL53L0X_1.startContinuous(50);
VL53L0X_2.startContinuous(50);
}

void loop()
{
////////// Читаем показания датчиков и выставляем флаги срабатывания
unsigned int Distance1 = VL53L0X_1.readRangeContinuousMillimeters();  
unsigned int Distance2 = VL53L0X_2.readRangeContinuousMillimeters();  
VL53L0X_1_state = (Distance1 < minDistance);
VL53L0X_2_state = (Distance2 < minDistance);
////////// Вот тут самое главное - анализируем флаги срабатывания и прочие флаги. Расписано неоптимально, зато подробно.
if ( !VL53L0X_1_state && !VL53L0X_2_state) // VL53L0X_1 off, VL53L0X_2 off
 {
 if ( VL53L0X_first > 0 && VL53L0X_together && VL53L0X_1_already && VL53L0X_2_already)
  {
  if ( VL53L0X_first == 1) // IN
   {
   HumanCounter++;
   }  
  else // OUT
   {
   HumanCounter--;
   }
  }
 VL53L0X_1_already = false;
 VL53L0X_2_already = false;
 VL53L0X_together = false;
 VL53L0X_first = 0;
 }  
else if ( VL53L0X_1_state && !VL53L0X_2_state) // VL53L0X_1 ON, VL53L0X_2 off
 {
 VL53L0X_1_already = true;
 if ( !VL53L0X_2_already )
  {
  VL53L0X_first = 1; 
  }
 }  
else if ( !VL53L0X_1_state && VL53L0X_2_state) // VL53L0X_1 off, VL53L0X_2 ON
 {
 VL53L0X_2_already = true;
 if ( !VL53L0X_1_already )
  {
  VL53L0X_first = 2;
  }
 }  
else if ( VL53L0X_1_state && VL53L0X_2_state) // VL53L0X_1 ON, VL53L0X_2 ON
 {
 if ( VL53L0X_1_already && VL53L0X_2_already && VL53L0X_together ) 
  {
  if ( VL53L0X_first == 1 ) 
   {
   VL53L0X_2_already = false;
   }
  if ( VL53L0X_first == 2 ) 
   {
   VL53L0X_1_already = false;
   }
  }
 VL53L0X_together = true;
 }  
////////// Конец анализа.
if ( HumanCounter != OldHumanCounter ) 
 {
 #ifdef DEBUG
 Serial.print(F("HumanCounter = "));
 Serial.println(HumanCounter);
 #endif  
 }
OldHumanCounter = HumanCounter;
}


```


Второй алгоритм:

```
if ( !s1 && !s2) // IR1 off, IR2 off.
 //if ( !IR1_state && !IR2_state) // IR1 off, IR2 off.
    {
      if ( IR1IR2_first > 0 && IR1IR2_already && IR1_already && IR2_already)
         {
          if ( IR1IR2_first == 1) {hn++;}
          else {hn--;}
         }
     IR1_already = 0;
     IR2_already = 0;
     IR1IR2_already = 0;
     IR1IR2_first = 0;
     } 
  if ( s1 && !s2) // IR1 on, IR2 off.
     {
     IR1_already = 1;
     if ( !IR2_already ) {IR1IR2_first = 1;}
     } 
  if ( !s1 && s2) // IR1 off, IR2 on.
     {
     IR2_already = 1;
     if ( !IR1_already ) {IR1IR2_first = 2;}
     }
  if ( s1 && s2) // IR1 on, IR2 on.
     {IR1IR2_already = 1;} 

  if ( hn < 0 ) {hn = 0;}
  if ( hn != ho ){
    Serial.print("Chel:");
    Serial.println(hn);
    client.publish(topic_in,String(hn)); // отправляем в топик для людей количество людей
ho = hn;
```
Но его качество работы сильно зависит от скорости как самого скетчя так и прохода через датчики.

На ардуино все работало нормально а вот перенес на ESP8266 и библиотека VL53L0X.h стала намертво вешать еспшку да так что и через сторожевой таймер не перезапускается. 

Есть еще библиотека от VL53L0X от адафрут, она не вешает есп, но крайне убога, тормозна и датчики с ней работают не стабильно.


