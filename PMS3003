#include "PMS.h"
#include <SoftwareSerial.h>
#include <BME280I2C.h>
#include <Wire.h>
#include <MySQL_Connection.h>
#include <MySQL_Cursor.h>
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

SoftwareSerial mySerial(D7, D8); // RX, TX
BME280I2C bme;
PMS pms(mySerial);
PMS::DATA data;

/********************************************************************/
  #define WIFIMODE WIFI_STA //(WIFI_STA);// WIFI_AP lub WIFI_AP_STA
/********************************************************************/
//parametry logowania do sieci WiFi
char ssid[50];// wpisujemy w monitorze
char password[50];// wpisujemy w monitorze i następnym razem wystarczy wcisnąć enter lub poczekać czas określony w Serial.setTimeout (dane nie są wyświetlane na Serial Monitor)
/******************************************************************/
//serwer na porcie 80
ESP8266WebServer server(80);
 
String state;  //stan
String replay; //bufor odpowiedzi serwera
String stan_wifi,zap1txt,zap2txt,zap10txt;
IPAddress ip_wifi;
String Stemp,pms_stan,bme_stan;
float t(NAN), h(NAN), p(NAN);
//float rt,ah,rp,pr,wys=114; //wysokość npm - odczytać np z gps
unsigned int i,j,k,st_pr_js,bme_st=2,pms_st=1;
const String okay="<font color=\"green\"> w normie </font>";
const String warn="<font color=\"orange\"> nie ma rewelacji </font>";
const String bad="<font color=\"red\"> JEST ŹLE </font>";
 
char hostname[] = "***"; // adres serwera mysql
char user[] = "***";        // MySQL nazwa użytkownika
char passMSQL[] = "***";           // MySQL hasło użytkownika
char INSERT_DATA[] = "INSERT INTO **.**Log (t,h,p) VALUES (%s,%s,%s)";
char INSERT_DATAPM[] = "INSERT INTO **.pm (pm1,pm25,pm10) VALUES (%s,%s,%s)";
char query[128];
char queryPM[128];
char temperature[10];
char wilgotnosc[10];
char cisnienie[10];
char p25[10], p10[10], p1[10];
float tA,hA,pA;

long sPM1=0;
long sPM2=0;
long sPM10=0;
long zap1=0;
long zap2=0;
long zap10=0;
int LED = D3;
//char buf[50];

unsigned long period = 600000; //odczyt danych z czujnika trwa 81 sekund, więc odpytanie co 10 minut da nam 6 lat laser lifetime.
unsigned long time_now = 0;
unsigned long zapamietajCzas = -600000; //ujemny period żeby na początku pętli loop nie czekać

// Begin reboot code
  int num_fails;                  // zmienna zachowująca ilość prób połaczenia
  #define MAX_FAILED_CONNECTS 5   // maksymalna ilość nieudanych prób połączenia z MySQL, po których następuje restart
 
IPAddress server_ip;
WiFiClient client;
MySQL_Connection conn((Client *)&client);
 
/********************************* strona html '********************************************/
String body ()
  {
  String b;
  String pms_stan,bme_stan;
  if (pms_st)
    {pms_stan="<font color=\"red\"> Brak danych z czujnika zapylenia. </font>";}
    else
    {pms_stan=" PMS:OK ";}
   if (bme_st)
    {bme_stan="<font color=\"red\"> Nie został wykryty czujnik THP.</font>";}
    else
    {bme_stan=" BME:OK ";}
    
    /*********************** odpowiedź html *************************************************/ 
    b="<html>\n<head>\n<meta charset=\"UTF-8\"/><meta http-equiv=refresh content=\"60\" />\n<title>Monitor</title>\n</head>\n<body bgcolor=\"#6581b3\"><center>";    
    b+="<h1>Monitoring powietrza</h1><hr width=\"50%\">";
    b+="<h3>zapylenie cząstkami 1µm: "+String(zap1) + " µg/m³ "+zap1txt+"</h3>\n";
    b+="<h3>     zapylenie cząstkami 2,5µm: "+String(zap2) + " µg/m³ "+zap2txt+"</h3>     \n";
    b+="<h3>     zapylenie cząstkami 10µm: "+String(zap10) + " µg/m³ "+zap10txt+"</h3>     \n <hr width=\"50%\">\n";
    b+="<h3>     temperatura: "+String(t) + "°C</h3>     <hr width=\"50%\">\n";
    b+="<h3>     ciśnienie: "+String(p) + " hPa</h3>     \n";
    b+="<h3>     wilgotność względna: "+String(h) + " %</h3>     \n";
    //b+="<h3>     wilgotność absolutna: "+String(ah) + " g/m³</h3>     \n";
    //b+="<h3>     temperatura punktu rosy: "+String(pr) + "°C</h3>     <hr width=\"50%\">\n";
    b+=pms_stan+"<br>"+bme_stan+"<br>wifi "+stan_wifi+" RSSI:" + String(WiFi.RSSI()) + " dB";
    b+="<br>V.1 na przykładzie ver 2.2 (C)tos\n</body>\n</html>";
 
    return b;
  }
/******************************* pakiet JSON *********************************************/
String json ()
  {
  String b;    
    /*********************** odpowiedź json **********************************************/ 
    b="{\"stan\":"+ String(st_pr_js) +",\n";  
    b+="\"RSSI\":"+ String(WiFi.RSSI()) +",\n";
    b+="\"temperatura\":"+String(t) + ",\n";
    b+="\"wilgotnosc\":"+String(h) + ",\n";
    b+="\"cisnienie\":"+String(p) + ",\n";
    b+="\"zapylenie1\":"+String(zap1) + ",\n";
    b+="\"zapylenie2\":"+String(zap2) + ",\n";
    b+="\"zapylenie10\":"+String(zap10) + "\n";
    b+="}";
 
    return b;
  }
/******************************************** ustawienia systemu **************************/

void setup()
{
  byte n = 0;
  byte mac[6];
  Serial.begin(9600);
  Serial.setTimeout(10000); // czas na wpisanie SSID i password
  Serial.println("<Urządzenie jest gotowe.>");
  mySerial.begin(9600);
  Wire.begin();
  pinMode(LED, OUTPUT);
  pms.passiveMode();

  WiFi.hostname ("ESP_sensors");
  WiFi.mode (WIFIMODE);
  Serial.println("Podaj SSID sieci WIFI");
  //while (Serial.available() == 0) {
    //wait    
  //}
  Serial.readBytesUntil(10, ssid, 50);
  Serial.println(ssid);
  Serial.println("Podaj hasło sieci WIFI");
  //while (Serial.available() == 0) {
    // wait
  //}
  Serial.readBytesUntil(10, password, 50);
  Serial.println(password);
  if(WIFIMODE == WIFI_AP)
    {
    Serial.print("Setting soft-AP ... ");
    Serial.println(WiFi.softAP("Monitor") ? "Ready" : "Failed!");
    WiFi.softAP("Monitor");
    ip_wifi=WiFi.softAPIP();
    Serial.print("Soft-AP IP address = ");
    Serial.println(ip_wifi);
   
    }
  if (WIFIMODE == WIFI_STA )
    {
    //łączymy się do sieci WiFi
    Serial.print("Łączenie do sieci: ");
    Serial.print(ssid);
    Serial.println("");
    WiFi.begin(ssid, password);
    WiFi.macAddress(mac);
    
    n = 0;
    while (WiFi.status() != WL_CONNECTED) 
      {
      Serial.print(".");
      digitalWrite(LED, HIGH);
      delay(500);
      digitalWrite(LED, LOW);
      delay(500);
      if (n > 60)  
        {
        Serial.println("restart ");
        ESP.restart();
        }
    n++;
    }
  Serial.println("");
  Serial.println("Połączenie WiFi OK");
  stan_wifi="ssid:"+String(WiFi.SSID());
  Serial.println(ip_wifi=WiFi.localIP());
  }
  /************************ konfig wyjść **********************************************/
  pinMode (D4,OUTPUT);
  digitalWrite (D4,1);
  /****************************** strona główna ***************************************/
  server.on("/", []() 
  {
  //informacja zwrotna do przeglądarki
  //digitalWrite(D4,0);
  server.send(200, "text/html", body());
  //digitalWrite(D4,1);
  
  });
  //json 
  server.on("/json", []() {
  //informacja zwrotna do przeglądarki
  //digitalWrite(D4,0);
  server.send(200, "text/html", json());
  //digitalWrite(D4,1);
  });
 
  server.begin();
  
    if(!bme.begin())
    {
    Serial.println("Nie wykryto BME280, sprawdź podłączenie.");
    delay(1000);
      if(!bme.begin())
      {bme_st=2;}
        else
        {bme_st=0;}
        }
        else
        {bme_st=0;}
 
  //switch(bme.chipModel())
  //{
  //   case BME280::ChipModel_BME280:
  //     Serial.println("Wykryto sensor BME280. Success.");
  //    break;
  //   case BME280::ChipModel_BMP280:
  //     Serial.println("Found BMP280 sensor! No Humidity available.");
  //     break;
  //   default:
  //     Serial.println("Nieznany sensor! Błąd!");
  //}
connecting();
}
/******************połączenie z bazą MYSQL************************************/
void connecting(){
WiFi.hostByName(hostname,server_ip);
Serial.println(server_ip);
 
Serial.println("Nawiązywanie połączenia ze serwerem MYSQL...");
delay(1000);
  if (conn.connect(server_ip, 3306, user, passMSQL)) {
    Serial.println("Połączenie ze serwerem MYSQL zostało ustanowione.");
    delay(1000);
  }
  else {
    Serial.println("Connecting...");
    if (conn.connect(server_ip, 3306, user, password)) {
      delay(500);
    } else {
      num_fails++;
      Serial.println("Connect failed!");
      if (num_fails == MAX_FAILED_CONNECTS) {
        Serial.println("Ok, that's it. I'm outta here. Rebooting...");
        delay(2000);
        // Here we tell the Arduino to reboot by redirecting the instruction
        // pointer to the "top" or position 0. This is a soft reset and may
        // not solve all hardware-related lockups.
        ESP.reset();
      }
    }
}
}
/******************** loop ***************************************************/

void loop()
{
  time_now = millis(); // daje idealny odstęp czasu wyznaczany przez zmienną period.
  if (time_now - zapamietajCzas >= period){
  zapamietajCzas=time_now;

  k=0;
  i=0;
  Serial.println("Budzenie czujnika. Czekaj 30 sekund na ustabilizowanie pomiaru...");
  pms.wakeUp();
  delay(30000);

  Serial.println("Wysyłam żądanie odczytu...");
  for (int a=0; a < 25; a++)
  {
    pms.requestRead();
    if (pms.readUntil(data))
    {
      digitalWrite(D4,0);
      pms_st=0;
      Serial.print("PM 1.0 (ug/m3): ");    Serial.print(data.PM_AE_UG_1_0);
      Serial.print("\tPM 2.5 (ug/m3): ");    Serial.print(data.PM_AE_UG_2_5);
      Serial.print("\tPM 10.0 (ug/m3): ");    Serial.print(data.PM_AE_UG_10_0);
      digitalWrite(LED, HIGH);
      delay(1000);
      digitalWrite(LED, LOW);
      delay(1000);
      k++;
      if(k>=4)//pierwsze pomiary są zaniżone
      {
        zap1=data.PM_AE_UG_1_0;
        zap2=data.PM_AE_UG_2_5;
        zap10=data.PM_AE_UG_10_0;
        sPM2+=zap2; sPM10+=zap10; sPM1+=zap1;
      }
      //Serial.println("Oczekuj 2 sekundy na dane...");
      Serial.print("\t\t");
      Serial.println(k);
      //Serial.print(sPM1);Serial.print("/");
      //Serial.print(sPM2);Serial.print("/");
      //Serial.print(sPM10);Serial.print("/");
      if (zap1 < 36)
             {
              zap1txt=okay;
              }
              else if (( zap1 > 35)& (zap1 < 60))
              {zap1txt=warn;}
              else
              {zap1txt=bad;}
          if (zap2 < 36)
              {zap2txt=okay;}
              else if (( zap2 > 35)& (zap2 < 84))
              {zap2txt=warn;}
              else
              {zap2txt=bad;}
          if (zap10 < 60)
              {zap10txt=okay;}
              else if (( zap10 > 59)& (zap10 < 140))
              {zap10txt=warn;}
              else
              {zap10txt=bad;}
    }
    else
    {
      Serial.println("No data.");
      //Serial.println("Oczekuj 2 sekundy na dane...");
      delay(2000);
    }
  }
  unsigned long usypianie = period - (millis() - zapamietajCzas);
  Serial.print("Usypianie PMS3003 na ");
  Serial.print(usypianie/1000);
  Serial.println(" sek.");
  pms.sleep();

  Serial.print("Temp (°C)"); Serial.print("\tWilg (% RH)"); Serial.println("\tCiśn (hPa)");
  for (int b=0; b < 25; b++)
  {
  BME280::TempUnit tempUnit(BME280::TempUnit_Celsius);
  BME280::PresUnit presUnit(BME280::PresUnit_hPa);
  bme.read(p, t, h, tempUnit, presUnit);

  Serial.print(t);
  Serial.print("\t\t");
  Serial.print(h);
  Serial.print("\t\t");
  Serial.print(p);
  if (isnan(p)|isnan(t)|isnan(h))
    {bme_st=2;}
   else
    {bme_st=0;
     i++;
     if(i>=4)
     {
        tA+=t;
        hA+=h;
        pA+=p;
        Serial.print("\t\t");
        Serial.println(i);
     }
     else
     {
        Serial.print("\t\t");
        Serial.println(i);
     }
    }
    delay(2000);
  }
  
  if (conn.connected()) {
          Serial.print(sPM1/(k-3));Serial.print("/");
          Serial.print(sPM2/(k-3));Serial.print("/");
          Serial.print(sPM10/(k-3));Serial.print(" - ");
          MySQL_Cursor *cur_mem1 = new MySQL_Cursor(&conn);
          dtostrf(sPM1/(k-3), 1, 2, p1);
          dtostrf(sPM2/(k-3), 1, 2, p25);
          dtostrf(sPM10/(k-3), 1, 2, p10);
          sprintf(queryPM, INSERT_DATAPM, p1, p25, p10);
          cur_mem1->execute(queryPM);
          delete cur_mem1;
          Serial.println("Pył - Dane zapisane.");
          sPM2=0; sPM10=0; sPM1=0;

          MySQL_Cursor *cur_mem2 = new MySQL_Cursor(&conn);
          dtostrf(tA/(i-3), 1, 3, temperature);
          dtostrf(hA/(i-3), 1, 3, wilgotnosc);
          dtostrf(pA/(i-3), 1, 2, cisnienie);
          sprintf(query, INSERT_DATA, temperature, wilgotnosc, cisnienie);
          cur_mem2->execute(query);
          delete cur_mem2;
          Serial.println("THP - Dane zapisane.");
          tA=0; hA=0; pA=0;
        } else {
          Serial.println("Ponowne łączenie...");
          if (conn.connect(server_ip, 3306, user, password)) {
            delay(500);
            MySQL_Cursor *cur_mem1 = new MySQL_Cursor(&conn);
            dtostrf(sPM1/k, 1, 2, p1);
            dtostrf(sPM2/k, 1, 2, p25);
            dtostrf(sPM10/k, 1, 2, p10);
            sprintf(queryPM, INSERT_DATAPM, p1, p25, p10);
            cur_mem1->execute(queryPM);
            delete cur_mem1;
            Serial.println("Pył - Dane zapisane.");
            sPM2=0; sPM10=0; sPM1=0;

            MySQL_Cursor *cur_mem2 = new MySQL_Cursor(&conn);
            dtostrf(tA/(i-3), 1, 3, temperature);
            dtostrf(hA/(i-3), 1, 3, wilgotnosc);
            dtostrf(pA/(i-3), 1, 2, cisnienie);
            sprintf(query, INSERT_DATA, temperature, wilgotnosc, cisnienie);
            cur_mem2->execute(query);
            delete cur_mem2;
            Serial.println("THP - Dane zapisane.");
            tA=0; hA=0; pA=0;
          } else {
            num_fails++;
            Serial.println("Połączenie nieudane!");
            if (num_fails == MAX_FAILED_CONNECTS) {
            Serial.println("Ok, rozumiem. Nic innego nie pozostaje. Restart...");
            delay(2000);
        // Here we tell the Arduino to reboot by redirecting the instruction
        // pointer to the "top" or position 0. This is a soft reset and may
        // not solve all hardware-related lockups.
            ESP.reset();
          }
        }
      }
if (WIFIMODE == WIFI_STA )
    {
    stan_wifi="SSID:"+String(WiFi.SSID()); 
    }
  if(WIFIMODE == WIFI_AP)
    {
    stan_wifi="AP:"+String(WiFi.softAPgetStationNum()) +" conn"; 
    } 
    st_pr_js=pms_st+bme_st;
  server.handleClient();
}
}
