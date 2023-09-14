/* 
  
  DEVICE: 
  Enter device name here...
  
  DESCRIPTION:
  Enter sketch overview here...

  NOTES ON WORKING WITH ARDUINO CLOUD
  1. Try to avoid delay(); it messes with the arduino looking for the wifi connection.
  2. All arduinos should have the basic wifi code and an IP address; connect via USB and check the serial monitor to get the IP address.
  3. The LED_BUILTIN will be lit when the initial wifi connection is established and the IP is good; the LED_BUILTIN is NOT a connection status light though; we need it for testing and other purposes.
  4. If the Arduino Over-The-Air port doesn't show up in the Arduino web editor, push the reset button on the arduino and the Over-The-Air port option should show up within 30-60 seconds.
  5. After a successful over-the-air update, the Over-The-Air port should show up within 30-60 seconds. See #4 if it doesn't.
  
  
*/
#include <SPI.h>
#include <WiFiNINA.h>
#include "thingProperties.h"

int status = WL_IDLE_STATUS;     // the Wifi radio's status
int keyIndex = 0;                 // your network key Index number (needed only for WEP)
WiFiServer server(80);           // The port for the local server on the arduino; always 80

// Replace delay() functions
// Delay functions keep Arduino from connecting to local network 
#define TIME_SLOT_INTERVAL 10000
unsigned long lastTickTime;

void setup() {
  
  // Initialize serial and wait for port to open:
  Serial.begin(9600);
  // This delay gives the chance to wait for a Serial Monitor without blocking if none is found
  delay(1500); 
  
  lastTickTime = millis();

  // Defined in thingProperties.h
  initProperties();

  // Connect to Arduino IoT Cloud
  ArduinoCloud.begin(ArduinoIoTPreferredConnection);
  
  pinMode(LED_BUILTIN, OUTPUT);      // set the LED pin mode

  // check for the WiFi module:

  if (WiFi.status() == WL_NO_MODULE) {

    Serial.println("Communication with WiFi module failed!");

    // don't continue

    while (true);

  }

  String fv = WiFi.firmwareVersion();

  if (fv < WIFI_FIRMWARE_LATEST_VERSION) {

    Serial.println("Please upgrade the firmware");

  }
  
  // attempt to connect to Wifi network:

  while (status != WL_CONNECTED) {
    
    unsigned long msNow = millis();
    if(msNow - lastTickTime >= TIME_SLOT_INTERVAL){
      /* do something */
      lastTickTime = msNow;
      
      Serial.print("Attempting to connect to Network named: ");

      Serial.println(SSID);                   // print the network name (SSID);
  
      // Connect to WPA/WPA2 network. Change this line if using open or WEP network:
  
      status = WiFi.begin(SSID, PASS);
    
    }

    

    // wait 10 seconds for connection:

    // delay(10000); // replaced by lastTimeTick

  }

  server.begin();                           // start the web server on port 80

  printWifiStatus();                        // you're connected now, so print out the status

  // Check if IP is ok
  IPAddress ip = WiFi.localIP();
  // Print it in the serial monitor for debugging
  Serial.print("Checking IP Address and turning on led builtin if it's ok: ");
  Serial.println(ip);
  // Then turn on LED_BUILTIN if IP is ok
  // The IP is an array with 4 numbers
  // The local ip will always be something like 192.168.2.XX
  // THIS ARDUINO'S IP SHOULD BE 192.168.2.36, so
  // ip[0] == 192 should be true
  // ip[1] == 168 should be true
  // ip[2] ==   2 should be true
  // ip[3] ==  36 should be true
  if (ip[3] == 36) {
    digitalWrite(LED_BUILTIN, HIGH);
  } else {
    // Turn LED on if the IP isn't right.
    digitalWrite(LED_BUILTIN, LOW);
  }

  
  /*
     The following function allows you to obtain more information
     related to the state of network and IoT Cloud connection and errors
     the higher number the more granular information you’ll get.
     The default is 0 (only errors).
     Maximum is 4
 */
  setDebugMessageLevel(2);
  ArduinoCloud.printDebugInfo();
}

void loop() {
  ArduinoCloud.update();
  // Your code here 
  WiFiClient client = server.available();   // listen for incoming clients

  if (client) {                             // if you get a client,

    Serial.println("new client");           // print a message out the serial port

    String currentLine = "";                // make a String to hold incoming data from the client

    while (client.connected()) {            // loop while the client's connected

      if (client.available()) {             // if there's bytes to read from the client,

        char c = client.read();             // read a byte, then

        Serial.write(c);                    // print it out the serial monitor

        if (c == '\n') {                    // if the byte is a newline character

          // if the current line is blank, you got two newline characters in a row.

          // that's the end of the client HTTP request, so send a response:

          if (currentLine.length() == 0) {

            // HTTP headers always start with a response code (e.g. HTTP/1.1 200 OK)

            // and a content-type so the client knows what's coming, then a blank line:

            client.println("HTTP/1.1 200 OK");

            client.println("Content-type:text/html");

            client.println();

            // the content of the HTTP response follows the header:

            client.print("Click <a href=\"/H\">here</a> turn the LED_BUILTIN on<br>");

            client.print("Click <a href=\"/L\">here</a> turn the LED_BUILTIN off<br>");

            // The HTTP response ends with another blank line:

            client.println();

            // break out of the while loop:

            break;

          } else {    // if you got a newline, then clear currentLine:

            currentLine = "";

          }

        } else if (c != '\r') {  // if you got anything else but a carriage return character,

          currentLine += c;      // add it to the end of the currentLine

        }

        // Check to see if the client request was "GET /H" or "GET /L":


        if (currentLine.endsWith("GET /H")) {

          digitalWrite(LED_BUILTIN, HIGH);               // GET /H turns the LED on

        }

        if (currentLine.endsWith("GET /L")) {

          digitalWrite(LED_BUILTIN, LOW);                // GET /L turns the LED off

        }

      }

    }

    // close the connection:

    client.stop();

    Serial.println("client disconnected");

  }
  
}


void printWifiStatus() {

  // print the SSID of the network you're attached to:

  Serial.print("SSID: ");

  Serial.println(WiFi.SSID());

  // print your board's IP address:

  IPAddress ip = WiFi.localIP();

  Serial.print("IP Address: ");

  Serial.println(ip);

  // print the received signal strength:

  long rssi = WiFi.RSSI();

  Serial.print("signal strength (RSSI):");

  Serial.print(rssi);

  Serial.println(" dBm");

  // print where to go in a browser:

  Serial.print("To see this page in action, open a browser to http://");

  Serial.println(ip);
}


/*
  This function is not used but 
  DO NOT DELETE THIS FUNCTION!
  It is needed for Arduino IOT to connect correctly
  over the air.
*/
void onDummyVariableForOverTheAirUpdatesChange()  {
  // Add your code here to act upon DummyVariableForOverTheAirUpdates change
}
