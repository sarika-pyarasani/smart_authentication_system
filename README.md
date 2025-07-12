# smart_authentication_system
used at highly confidential places(person authentication)).

//code
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <MFRC522.h>
#include <SPI.h>
#include <Adafruit_Fingerprint.h>

// RFID
#define SS_PIN 53
#define RST_PIN 49
MFRC522 mfrc522(SS_PIN, RST_PIN);

// LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Keypad
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {22, 24, 26, 28};
byte colPins[COLS] = {30, 32, 34, 36};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// GSM
#define GSM Serial1

// Fingerprint
#define mySerial Serial2
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);

// Stored credentials
const String storedUID = "D3 35 34 C5";
const String storedPIN = "1234";

void setup() {
  Serial.begin(9600);
  SPI.begin();
  mfrc522.PCD_Init();

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0,0);
  lcd.print("Scan RFID card");

  GSM.begin(9600);

  finger.begin(57600);
  if (finger.verifyPassword()) {
    Serial.println("Fingerprint sensor ready");
  } else {
    lcd.clear();
    lcd.print("Fingerprint ERROR");
    Serial.println("Fingerprint sensor not found");
    delay(3000);
  }
}

void loop() {
  if (!detectRFID()) return;

  String uid = readUID();
  Serial.println("UID: " + uid);
  lcd.clear();
  lcd.print("Enter PIN:");

  String enteredPIN = readPIN(10000); // 10s timeout
  if (enteredPIN == "") {
    lcd.clear(); lcd.print("PIN Timeout");
    delay(2000); return;
  }

  if (uid == storedUID && enteredPIN == storedPIN) {
    lcd.clear(); lcd.print("PIN Verified");
    delay(1000);
    sendSMS(uid);

    lcd.clear(); lcd.print("Place Finger");
    int id = getFingerprintID();
    lcd.clear();
    if (id == -1) {
      lcd.print("Fingerprint"); lcd.setCursor(0,1); lcd.print("Not Found");
    } else {
      lcd.print("Authorized");
      Serial.println("Access Granted");
      // Trigger unlock mechanism here
    }
    delay(2000);
  } else {
    lcd.clear(); lcd.print("Auth Failed");
    delay(2000);
  }

  lcd.clear(); lcd.print("Scan RFID card");
}

bool detectRFID() {
  return mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial();
}

String readUID() {
  String content = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    content += (mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
    content += String(mfrc522.uid.uidByte[i], HEX);
  }
  content.toUpperCase(); content.trim();
  return content;
}

String readPIN(unsigned long timeoutMillis) {
  String pin = "";
  unsigned long start = millis();
  while (millis() - start < timeoutMillis) {
    char key = keypad.getKey();
    if (key) {
      if (isDigit(key)) {
        pin += key;
        lcd.setCursor(pin.length()-1,1); lcd.print("*");
        if (pin.length() == 4) break;
      } else if (key == '*') { pin = ""; lcd.setCursor(0,1); lcd.print("    "); }
    }
  }
  return pin.length() == 4 ? pin : "";
}

int getFingerprintID() {
  if (finger.getImage() != FINGERPRINT_OK) return -1;
  if (finger.image2Tz() != FINGERPRINT_OK) return -1;
  if (finger.fingerFastSearch() != FINGERPRINT_OK) return -1;
  return finger.fingerID;
}

void sendSMS(String uid) {
  GSM.println("AT+CMGF=1");
  delay(100);
  GSM.println("AT+CMGS=\"YOUR_GSM_NUMBER\"");
  delay(100);
  GSM.print("Card Accessed: "); GSM.println(uid);
  GSM.write(26); // Ctrl+Z
  delay(1000);
}
