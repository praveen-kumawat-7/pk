#include <SPI.h>
#include <MFRC522.h>

// Define GSM modem model (SIM800 in this case)
#define TINY_GSM_MODEM_SIM800
#include <TinyGsmClient.h>

// Pins for the RC522 RFID reader
#define SS_PIN 5
#define RST_PIN 22

// Initialize the MFRC522 RFID reader
MFRC522 rfid(SS_PIN, RST_PIN);

// GSM settings (modify these with your actual details)
#define SerialMon Serial
#define SerialAT Serial2  // Use Serial2 for communication with SIM800L

// APN and credentials (set according to your SIM provider)
const char apn[] = "your_apn";  // Replace with actual APN
const char user[] = "";         // Leave empty if not required
const char pass[] = "";         // Leave empty if not required

TinyGsm modem(SerialAT);  // Create modem instance for GSM
TinyGsmClient client(modem); // Create client instance for HTTP requests

// Initialization of RFID and GSM in setup
void setup() {
  // Start serial communication
  SerialMon.begin(115200);
  SerialAT.begin(9600, SERIAL_8N1, 16, 17);  // UART2 for GSM module (TX = GPIO16, RX = GPIO17)
  
  // Initialize RFID reader
  SPI.begin();
  rfid.PCD_Init();
  SerialMon.println("Place your card on the reader...");

  // Initialize GSM module
  modem.restart();
  modem.gprsConnect(apn, user, pass);
  
  if (modem.isGprsConnected()) {
    SerialMon.println("GPRS connected successfully");
  }
}

// Main loop for reading RFID and sending data via GSM
void loop() {
  // Check if a card is present and read its UID
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    String uid = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      uid += String(rfid.uid.uidByte[i], HEX);
    }

    SerialMon.print("Card UID: ");
    SerialMon.println(uid);

    // Send HTTP request to the server with card UID (example)
    if (modem.isGprsConnected()) {
      String server = "http://yourserver.com/api";  // Replace with your actual server URL
      sendHTTPPost(server, "card_uid=" + uid);
    }
  }

  delay(1000);  // Delay for 1 second before reading again
}

// Function to send HTTP POST request using the TinyGSM client
void sendHTTPPost(String server, String data) {
  if (client.connect(server.c_str(), 80)) {
    client.println("POST /api HTTP/1.1");
    client.println("Host: yourserver.com");
    client.println("Connection: close");
    client.println("Content-Type: application/x-www-form-urlencoded");
    client.print("Content-Length: ");
    client.println(data.length());
    client.println();
    client.println(data); // Send the data (e.g., card UID)
    client.stop(); // Close the connection
    SerialMon.println("Data sent to server");
  } else {
    SerialMon.println("Failed to connect to the server");
  }
}

