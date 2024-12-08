#include <SoftwareSerial.h>
#include <DFRobot_SIM808.h>
#include <TinyGPS++.h>
#include <WiFi.h>
#include <WiFiClient.h>
#include <WebServer.h>
#include <ESPmDNS.h>
#include "homepage.h"

#define MESSAGE_LENGTH 160
char message[MESSAGE_LENGTH];
int messageIndex = 0;
char phone[16];
char datetime[24];
String text = "Pos: ";
String textFinal;
char textC[40];// buffer to hold C string with long and lat coordinates not in GoogleMaps format
char mapsLocInC[40];// buffer to hold C string with location as Google Map link
String mapsLocation = "http://maps.google.com/?q=";
DFRobot_SIM808 sim808(&Serial2);
String latCPPStr;
String lonCPPStr;
#define PHONE_NUMBER  "" //number you are texting to
const char* ssid = "POCOF3";
const char* password = "";

WebServer server(80);
const int PWR_PIN = 23; // if you want a SW reboot of Sim808

String getLat() {
  
  return latCPPStr;
}
String getLon() {
  
  return lonCPPStr;
}

void handleRoot() {
  String message = homePagePart1;
  message += getLat();
  message += homePagePart3;
  message += getLon();
  message += homePagePart5;
  server.send(200, "text/html", message);
}

void handleNotFound() {
  String message = "File Not Found\n\n";
  message += "URI: ";
  message += server.uri();
  message += "\nMethod: ";
  message += (server.method() == HTTP_GET) ? "GET" : "POST";
  message += "\nArguments: ";
  message += server.args();
  message += "\n";
  for (uint8_t i = 0; i < server.args(); i++) {
    message += " " + server.argName(i) + ": " + server.arg(i) + "\n";
  }
  server.send(404, "text/plain", message);
}

void setup() {
  Serial2.begin(9600);//Rx tx
  Serial.begin(115200);
  pinMode (PWR_PIN, OUTPUT);

  // Initialize sim808 module 
  while (!sim808.init()) {
    delay(1000);
    Serial.print("Sim808 init error\r\n");
  }
  Serial.println("Sim808 init success");
   //Thist attaches the GPS to the SIM808
  while (!sim808.attachGPS()) {
    Serial.println("wait for GPS to attach");
    delay(1000);
  }// wait for GPS to attach
  Serial.println("Open the GPS power success");

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.println("");

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  if (MDNS.begin("esp32")) {
    Serial.println("MDNS responder started");
  }

  server.on("/", handleRoot);
  server.on("/inline", []() {
    server.send(200, "text/plain", "this works as well");
  });
  server.onNotFound(handleNotFound);

  server.begin();
  Serial.println("HTTP server started");
}

void loop(void) {
  float lat, lon;
  server.handleClient();
  delay(2);//allow the cpu to switch to other tasks...
  //Code beneath taken from DFRobot Example
  //Getting longitude and latitude from Satellite
  Serial.println("Wait for gps location");
  while (!sim808.getGPS()); //wait for a GPS location

  Serial.print("latitude :");
  lat = sim808.GPSdata.lat;
  Serial.println(lat, 6);
  
  lon = -sim808.GPSdata.lon;// add minus sign as we are west of GMT
  Serial.print("longitude :");
  Serial.println(lon, 6);
  
  latCPPStr = String (lat);// conv float to String
  mapsLocation += latCPPStr;
  latCPPStr += "Lat";
  lonCPPStr = String(lon);
  mapsLocation += "," + lonCPPStr;
  lonCPPStr += "Lon";
  textFinal = text + latCPPStr + " " + lonCPPStr;
  //Code beneath taken from Natashas Example
  textFinal.toCharArray(textC, textFinal.length() + 1);//conv String to C string
  Serial.println (textC);// textC can be sent to send long and lat location in text not in GoogleMap format
  Serial.println (mapsLocation);
  //  Send text with GoogleMaps location
  mapsLocation.toCharArray(mapsLocInC, mapsLocation.length() + 1);//conv String to C string
  sim808.sendSMS((char *)PHONE_NUMBER, mapsLocInC);
  mapsLocation = "http://maps.google.com/?q="; // reset if you want to read another location

  while (1);//nothing to do. Don't want to keep sending texts and wasting credit!
}
