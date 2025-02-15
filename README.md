# CP-with-ESP32-A1S_v1.0
Child Pickup™ SYS using ESP32 A1S_v1.0
Refactor this code to use ESP32 A1S development board to play mp3 files from the onboard SD Card Reader of the ESP32 A1S instead of the DFRobot DFPlayer Mini and output the sound to a bluetooth speaker using ESP32-A2DP library by pschatzmann or a similar approach to decode MP3 from SD and stream it over Bluetooth A2DP (The ESP32 will be a A2DP Data Source, The supported audio codec in ESP32 A2DP is SBC: The API is using PCM data normally formatted as 44.1kHz sampling rate, two-channel 16-bit sample data). The ESP32 A1S board should also be connected to WIFI using WiFiClientSecure and SSL for security/HTTPS in order to parse the webhook generated over to MAKE.COM using a GET request. Include the links to any libraries used in the code.
Attached is the ESP32 A1S Board for your reference.

#include <SPI.h>
#include <MFRC522.h>
#include "Arduino.h"
#include "SoftwareSerial.h"
#include "DFRobotDFPlayerMini.h"

#define RST_PIN         9           // Configurable, see typical pin layout above
#define SS_PIN          10          // Configurable, see typical pin layout above
#define MAX_RETRIES     3

// Updated UserData structure
struct UserData {
  String parentName;
  String childName;
  String phoneNumber;
  String studentNumber;
  String cardUID;
};


const int playPauseButton = 4;
const int shuffleButton = 5;
const byte volumePot = A0;
int prevVolume; 
byte volumeLevel = 0;
bool isPlaying = false;

MFRC522 mfrc522(SS_PIN, RST_PIN);  // RFID reader instance
SoftwareSerial mySoftwareSerial(3, 6); // RX, TX
DFRobotDFPlayerMini myDFPlayer;

UserData currentUser;  // Stores parsed data

// Debounce timing
unsigned long lastButtonPressTime = 0;
const unsigned long debounceDelay = 100;

void setup() {
  Serial.begin(115200);
  mySoftwareSerial.begin(9600);
  SPI.begin();
  mfrc522.PCD_Init();

  pinMode(playPauseButton, INPUT_PULLUP);
  pinMode(shuffleButton, INPUT_PULLUP);

  Serial.println(F("Initializing DFPlayer..."));
  if (!myDFPlayer.begin(mySoftwareSerial, false)) {
    Serial.println(F("Unable to begin DFPlayer! Check connections and SD card."));
    while (true);  // Halt if DFPlayer fails
  }
  Serial.println(F("DFPlayer Mini online. Place card on reader to play a specific song"));

  volumeLevel = map(analogRead(volumePot), 0, 1023, 0, 30);
  myDFPlayer.volume(volumeLevel);
  prevVolume = volumeLevel;

  myDFPlayer.EQ(DFPLAYER_EQ_NORMAL);
}

void loop() {
  adjustVolumeIfNeeded();
  handleButtonPress(playPauseButton, togglePlayPause);
  handleButtonPress(shuffleButton, shufflePlay);
  processCardReading();
}

void adjustVolumeIfNeeded() {
  volumeLevel = map(analogRead(volumePot), 0, 1023, 0, 30);
  if (abs(volumeLevel - prevVolume) >= 3) {
    myDFPlayer.volume(volumeLevel);
    //Serial.println(volumeLevel);  
    prevVolume = volumeLevel;
    delay(1);
  }
}

void handleButtonPress(int buttonPin, void (*action)()) {
  if (digitalRead(buttonPin) == LOW) {
    unsigned long currentTime = millis();
    if (currentTime - lastButtonPressTime > debounceDelay) {
      action();
      lastButtonPressTime = currentTime;
    }
  }
}

void togglePlayPause() {
  if (isPlaying) {
    myDFPlayer.pause();
    isPlaying = false;
    Serial.println("Paused..");
  } else {
    isPlaying = true;
    myDFPlayer.start();
    Serial.println("Playing..");
  }
}

void shufflePlay() {
  myDFPlayer.randomAll();
  Serial.println("Shuffle Play");
  isPlaying = true;
}

// Modified RFID processing function
void processCardReading() {
  if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) return;

  MFRC522::MIFARE_Key key;
  for (byte i = 0; i < 6; i++) key.keyByte[i] = 0xFF;

  // Read user data from Blocks 4-6
  currentUser.parentName = readBlock(5, key);
  currentUser.childName = readBlock(6, key);
  currentUser.phoneNumber = readBlock(4, key);
  currentUser.cardUID = getRFIDUID();
  currentUser.studentNumber = readBlock(1, key);

  
  if (currentUser.studentNumber != "") {
    myDFPlayer.play(currentUser.studentNumber.toInt());
    isPlaying = true;
  }


  // Generate and print webhook URL
  String webhookURL = constructWebhookURL(currentUser);
  Serial.println("Webhook URL:");
  Serial.println(webhookURL);
  
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1();
}


// New helper function to read any block
String readBlock(byte block, MFRC522::MIFARE_Key key) {
  byte retry = MAX_RETRIES;
  byte buffer[18];
  byte len = 18;

  while (retry--) {
    MFRC522::StatusCode status = mfrc522.PCD_Authenticate(
      MFRC522::PICC_CMD_MF_AUTH_KEY_A, block, &key, &(mfrc522.uid));
      
    if (status != MFRC522::STATUS_OK) {
      Serial.print(F("Auth failed Block "));
      Serial.print(block);
      Serial.print(": ");
      Serial.println(mfrc522.GetStatusCodeName(status));
      continue;
    }

    status = mfrc522.MIFARE_Read(block, buffer, &len);
    if (status == MFRC522::STATUS_OK) {
      return parseData(buffer);
    }
  }
  return "";
}

// Data parser with trimming (fixed)
String parseData(byte* buffer) {
  String result = "";
  for (uint8_t i = 0; i < 16; i++) {
    if (buffer[i] == 0 || buffer[i] == ' ') break;
    result += (char)buffer[i];
  }
  result.trim();  // Trim the string in-place
  return result;   // Return the modified string
}

// New function to get RFID UID as string
String getRFIDUID() {
  String uidString = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    uidString += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : "");
    uidString += String(mfrc522.uid.uidByte[i], HEX);
  }
  return uidString;
}

// Updated URL constructor with new parameters
String constructWebhookURL(const UserData& data) {
  const String baseURL = "https://hook.eu2.make.com/pe7n6h0cpbuio3vcs9vugd93l1p2s4aw";
  String url = baseURL + "?";
  
  // Original parameters
  url += "ChildName=" + urlEncode(data.childName);
  url += "&ParentName=" + urlEncode(data.parentName);
  url += "&ParentPhone=" + urlEncode(data.phoneNumber);
  url += "&StudentNumber=" + urlEncode(data.studentNumber);
  url += "&CardUID=" + urlEncode(data.cardUID);

  return url;
}

// URL encoding function (replaces JSON escaping)
String urlEncode(const String& input) {
  String encoded = "";
  const char* chars = input.c_str();
  
  for (unsigned int i = 0; i < input.length(); i++) {
    char c = chars[i];
    
    // Keep alphanumeric and safe characters unchanged
    if (isalnum(c) || c == '-' || c == '_' || c == '.' || c == '~') {
      encoded += c;
    }
    // Encode spaces as %20
    else if (c == ' ') {
      encoded += "%20";
    }
    // Encode '+' as %2B for phone numbers
    else if (c == '+') {
      encoded += "%2B";
    }
    // Encode other special characters
    else {
      char buf[4];
      sprintf(buf, "%%%02X", c);
      encoded += buf;
    }
  }
  return encoded;
}
