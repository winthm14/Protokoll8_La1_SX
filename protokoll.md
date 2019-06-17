# Protokoll 7 <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/30/HTL_Kaindorf_Logo.svg/300px-HTL_Kaindorf_Logo.svg.png" alt="">  
  
Professor: SX  
Übungsort: Kaindorf   
Übungsdatum: 02.04.2018  
Anwesend: Vollmaier Alois, Wegl Patrick, Winter Matthias, Winter Thomas  
Abwesend: Sarah Vezonik, Mercedes Wesonig  

## App.c  
```c
   1 #include "global.h" 
   3 #include <stdio.h>
   4 #include <string.h> 
   6 #include <avr/io.h>
   7 #include <avr/interrupt.h>
   8 #include <util/delay.h> 
  10 #include "app.h"
  11 #include "sys.h"
  12
  13 // declarations and definations 
  14 volatile struct App app;
  15
  16 // functions
  17 void app_init (void) 
  18   {
  19     memset((void *)&app, 0, sizeof(app));
  20   
  21     ADMUX = 8; // Multiplexer ADC8 = Temp.
  22     ADMUX |= (1 << REFS1) | (1 << REFS0); // VREF=1.1V
  23     ADMUX |= (1 << ADLAR); // Left Adj, -> Result in ADCH
  24   
  25     ADCSRA = (1 << ADEN) | 7; // fADC=125kHz
  26     ADCSRB = 0;
  27   
  28   }
  29 
  30 
  31 //--------------------------------------------------------
  32 
  33 uint16_t app_inc16BitCount (uint16_t cnt) 
  34   {
  35     return cnt == 0xffff ? cnt : cnt + 1;
  36   }
  37 
  38 uint8_t hex2int (char h) 
  39    {
  40     if (h >= '0' && h <= '9') 
  41       {
  42         return h - '0';
  43       }
  44     if (h >= 'A' && h <= 'F') 
  45       {
  46         return h - 'A' + 10;
  47       }
  48     return 0xff; // Fehler
  49    }
  50 
  51 uint8_t app_handleModbusRequest () 
  52   {
  53     
  54     // Check if valid modbus frame
  55     char *b = app.modbusBuffer;
  56     uint8_t size = app.bufIndex;
  57     
  58     if (size < 9) { return 1; }
  59     if (b[0] != ':') { return 2; }
  60     if (b[size - 1] != '\n') { return 3; }
  61     if (b[size - 2] != '\r') { return 4; }
  62     if ( (size - 3) % 2 != 0) { return 5; }
  63     for (uint8_t i = 1; i < (size - 2); i++) 
  64       {
  65         char c = b[i];
  66         if (! ((c >= '0' && c <= '9') || 
  67                (c >= 'A' && c <= 'F')))
  68             return 6;
  69       }
  70
  71     uint8_t lrc = 0;
  72
  73     for (uint8_t i = 1; i < (size - 4); i++) 
  74       {
  75         lrc += b[i];
  76       }
  77     lrc = (uint8_t)( -(int8_t)lrc);
  78     char s[3];
  79     snprintf(s, sizeof s, "%02X", lrc);
  80     if (b[size - 4] != s[0]) { return 7; }
  81     if (b[size - 3] != s[1]) { return 7; }
  82     
  83     uint8_t i, j;
  84     for (i = 1, j = 0; i < (size - 4); i += 2, j++ ) 
  85       {
  86         uint8_t hn = hex2int(b[i]);
  87         uint8_t ln = hex2int(b[i+1]);
  88         if (hn == 0xff || ln == 0xff) 
  89           {
  90             return 8;
  91           }
  92         uint8_t value = hn * 16 + ln;
  93         b[j] = value;
  94       }
  95
  96     size = j;
  97     
  98     uint8_t deviceAddress = b[0];
  99     if (deviceAddress != 1) 
  100       {
  101        return 0;
  102      }
  103    
  104     uint8_t funcCode = b[1];
  105     switch (funcCode) 
  106       {
  107        case 0x04: 
  108          {
  109            uint16_t startAddress = b[2] << 8 | b[3];
  110            uint16_t quantity = b[4] << 8 | b[5];
  111            if (quantity < 1 || quantity > 0x7d) 
  112              {
  113                 b[1] = 0x80 | b[1]; // error
  114                 b[2] = 0x03; // quantity out of range
  115                 size = 3;
  116            
  117              } 
  118                else if (startAddress != 1 || quantity != 1) 
  119                  {
  120                    b[1] = 0x80 | b[1]; // error
  121                    b[2] = 0x02; // wrong start address
  122                    size = 3; 
  123                  } 
  124                    else 
  125                      {
  126                        b[2] = 2;
  127                        b[3] = app.mbInputReg01 >> 8;
  128                        b[4] = app.mbInputReg01 & 0xff;
  129                        size = 5;
  130                      } break;
  131            }
  132        
  133         default: 
  134           {
  135            b[1] = 0x80 | b[1]; // error
  136            b[2] = 0x01; // function code not supported
  137            size = 3;
  138           } 
  139     }
  140     
  141     lrc = 0;
  142     printf(":");
  143     for (i = 0; i < size; i++) 
  144       {
  145         printf("%02X", (uint8_t)b[i]);
  146         lrc += b[i];
  147       }
  148     lrc = (uint8_t)(-(int8_t)lrc);
  149     printf("%02X", lrc);
  150     printf("\r\n");
  151     return 0;
  152     }
  153 
  154 void app_handleUartByte (char c) 
  155   {
  156     if (c == ':') 
  157       {
  158         if (app.bufIndex > 0) 
  159           {
  160             app.errCnt = app_inc16BitCount(app.errCnt);
  161           }
  162         app.modbusBuffer[0] = c;
  163         app.bufIndex = 1;
  164     
  165       } 
  166         else if (app.bufIndex == 0) 
  167           {
  168             app.errCnt = app_inc16BitCount(app.errCnt);
  169           } 
  170             else if (app.bufIndex >= (sizeof(app.modbusBuffer))) 
  171                  {
  172                app.errCnt = app_inc16BitCount(app.errCnt);
  173               } 
  174                else 
  175                  {
  176                    app.modbusBuffer[app.bufIndex++] = c;
  177                    if (c == '\n') 
  178                      {
  179                       uint8_t errCode = app_handleModbusRequest();
  180                       if (errCode > 0) 
  181                         {
  182                           app.errCnt = app_inc16BitCount(app.errCnt);
  183                         }
  184                         app.bufIndex = 0;
  185                      }
  186                  } 
  187   }
  188 
  189 void app_main (void) 
  190   {
  191     ADCSRA |= (1 << ADSC);     
  192     // Gerade: ModbusRegister = k * ADCH + d
  193     // aus Datenblatt:
  194     //     -45°C -> 242mV -> ADCH=56.79 -> MBR = -45*256 = -11520
  195     //      25°C -> 314mV -> ADCH=73.08 -> MBR =  25*256 =   6400
  196     //      85°C -> 380mV -> ADCH=88.40 -> MBR =  85*256 =  21760
  197     // daraus ergibt sich für:
  198     //      <=25°C -> MBR = k1 * ADCH + d1
  199     //      >25°C  -> MBR = k2 * ADCH + d2
  200     float k1 = 1100.061, d1 = -73992.464;
  201     float k2 = 1002.611, d2 = -66870.812;
  202 
  203     // reale Messung bei 22°C -> ADCH=88
  204     // interpoliert:
  205     //      22°C -> ?  -> ADCH=72.38 -> MBR = 22*256 = 5632
  206     // Offsetkorrektur um bei ADCH=88 auf 5632 zu kommen
  207     //     ADCH<=88 -> MBR = k1 * ADCH + d1 + o1
  208     //     ADCH>88  -> MBR = k2 * ADCH + d2 + o2
  209     float o1 = -17180.9;
  211     float o2 = -15726.956;
  212     
  213     float mbr;
  214     uint8_t adch = ADCH;
  215     // adch = 88;
  216     if (adch <= 88) 
  217       {
  218         mbr = k1 * adch + d1 + o1;
  219       } 
  220        else
  221             {
  222           mbr = k2 * adch + d2 + o2;
  223          }
  224     int16_t mbInputReg01 = (int16_t)mbr;
  225     int8_t vk = mbInputReg01 / 256;
  226     uint8_t nk = ((mbInputReg01 & 0xff) * 100) / 256;
  227          
  228     app.mbInputReg01 = (uint16_t)mbInputReg01;
  229        
  230     int c = fgetc(stdin);
  231     if (c != EOF) 
  232       {
  233         // printf("\r\n %02x\r\n", (uint8_t)c);
  234         app_handleUartByte((char) c);
  235       }
  236     
  237   }
  238 
  239 //--------------------------------------------------------
  241 
  242 void app_task_1ms (void) 
  243   {
  244     static uint16_t oldErrCnt = 0;
  245     static uint16_t timer = 0;
  246     if (app.errCnt != oldErrCnt) 
  247       {
  248         oldErrCnt = app.errCnt;
  249         timer = 2000;
  250         sys_setLed(1);
  251       }
  252     if (timer > 0) 
  253       {
  254         timer--;
  255         if (timer == 0) 
  256           {
  257             sys_setLed(0);
  258           }
  259       }
  260   }
  261 
  262 
  263 void app_task_2ms (void) {}
  264 void app_task_4ms (void) {}
  265 void app_task_8ms (void) {}
  266 void app_task_16ms (void) {}
  267 void app_task_32ms (void) {}
  268 void app_task_64ms (void) {}
  269 void app_task_128ms (void) {}
```
**Beschreibung**  
**Zeile 1-11:**  
Einbinden aller Header-dateien  
**Zeile 014:**  
deklaration der Variable app   
**Zeile 17-28:**  
Funktion app_init mit dem Rückgabedatentyp void, zur initioalisierung des systemsdDer Speicher wird geleert  
Im ADMUX Register wird der Multiplexer(die Signalweiche) auf den Temperatursensor gestellt, die Interne Referenzspannung von 1,1 Volt gewählt. Weiters wird eingestellt, dass das Messergebnis im ADLAR Register Linksbündig geschriben wird.   
Zuletzt wird noch die Frequenz des chips, herunter skalliert um eine messfrequenz von 125kHz zu erreichen.  
**Zeile 33-36**  
Es wird eine Funktion geschriben, die eine 16Bit Variable, bei jedem Aufruf der Funktion, hinauf zählt. Auser der Wert der Variable beträgt bereits 0xffff  
**Zeile 38-49**  
Eine Funktion mit dem Rückgabedatenty uint8_t und einem Parameter mit dem datentyp char. Diese Funktion wandelt Character in 8Bit-Werte um. bei einem Fehler wird 0xff zurückgegeben  
**Zeile 51-152**  
Funktion um eine Modbusframe Request vernünftig zu verarbeiten.  
In **Zeile 55** wird ein Zeiger auf app.modbusBuffer initialisiert. Mit der Variable size wird die größe des aktuell eintreffenden Modbusframes definiert.  
In **Zeile 58-63** wird geprüft ob ein sinvoller und vollständiger Modbusframe ankommt.
In **Zeile 64-69** wird sichergestellt dass nur ASCII zeichen von 0-9 und A-F ankommen.  
In **Zeile 71-81** Wird der LRC gebildet und in **Zeile 83-94** wird er in Integer werte umgewandelt und abgeglichen.  
Ab **Zeile 104** wird das Erkennen des Funktionscodes ausprogrammiert  
In **Zeile 107-131**  Wird der Funktionscode Überprüft und wenn er richtig ist das Frame gespeichert   
Ab **Zeile 154-187** wird Zeichen für zeichen des Modbusframes überprüft und gespeichert.
Zu beginn wird geprüft ob das erste Zeichen ein Doppelpunkt ist. Wenn dies nicht der Fall ist wird die Variable app.errCnt um eins Erhöht. Andernfals wird dises Zeichen an der Stelle 0 im Puffer gespeichert.
Als nächstes wird überprüft ob der buffer Index 0 ist oder ob man bereits am ende des modbusBuffers angekommen ist. in beiden Fällen wird der Error Counter um eins erhöht.
Wen soweit alles stimmt wird das Frame Zeichen für Zeichen in der modbusBuffer Gespeichert. Sobald das Zeichen ```\n``` ankommt wird die Funtion app_handleModbusRequest aufgerufen. Wenn diese Funktion 0 zurückliefert, wird der bufferIndex wider auf 0 gesetzt.
In der Funktion **main** in **Zeile 189-237** wird der Messwert aus dem ADCH Register geholt und mit kalibrierungswerten aus versuchen umgerechnet um einen Messwert zu bekommen.
Im ein Millisekunden Task ab **Zeile 242** Wird anhand der einzelnen Fehlerzustände eine LED zum leuchten gebracht.  

App.h
```c
   1 #ifndef APP_H_INCLUDED
   2 #define APP_H_INCLUDED
   3 
   4 // declarations
   5 
   6 struct App
   7 {
   8   uint8_t flags_u8;
   9   char modbusBuffer[32];
  10   uint8_t bufIndex;
  11   uint16_t errCnt;
  12   uint16_t mbInputReg01;
  13 };
  14 
  15 extern volatile struct App app;
  16 
  17 
  18 // defines
  19 
  20 #define APP_EVENT_0   0x01
  21 #define APP_EVENT_1   0x02
  22 #define APP_EVENT_2   0x04
  23 #define APP_EVENT_3   0x08
  24 #define APP_EVENT_4   0x10
  25 #define APP_EVENT_5   0x20
  26 #define APP_EVENT_6   0x40
  27 #define APP_EVENT_7   0x80
  28 
  29 
  30 // functions
  31 
  32 void app_init (void);
  33 void app_main (void);
  34 
  35 void app_task_1ms   (void);
  36 void app_task_2ms   (void);
  37 void app_task_4ms   (void);
  38 void app_task_8ms   (void);
  39 void app_task_16ms  (void);
  40 void app_task_32ms  (void);
  41 void app_task_64ms  (void);
  42 void app_task_128ms (void);
  43 
  44 #endif // APP_H_INCLUDED

```
**Beschreibung**  
In **Zeile 6-13** wird die Struktur App angelegt und die Einzelnen Datentypen definiert. In **Zeile15** wird eine Variable mit dem Datentyp ```extern volatile struct App```definiert. Das volatile bewirkt dass der Compiler nichts an diesem Ausdruck verändert. 





