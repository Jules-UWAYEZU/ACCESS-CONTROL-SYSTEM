# ACCESS-CONTROL-SYSTEM
Arduino Mega-based RFID + Keypad access control system. Users can unlock access using registered RFID tags or a PIN code (12345). On success, an LED turns on for 4 seconds; on failure, it blinks. Simple dual-auth security project for learning embedded systems and IoT basics.
#include <SPI.h>
#include <MFRC522.h>
#include <Keypad.h>

#define SS_PIN 53
#define RST_PIN 5
#define LED_PIN 7

MFRC522 mfrc522(SS_PIN, RST_PIN);

// ---------------- KEYBOARD SETUP ----------------
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {22, 23, 24, 25};
byte colPins[COLS] = {26, 27, 28, 29};

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// ---------------- PASSWORD ----------------
String inputPassword = "";
String correctPassword = "12345";

// ---------------- UID LIST ----------------
byte tag1[] = {0x63, 0x8E, 0x79, 0x97};
byte tag2[] = {0x73, 0x01, 0x06, 0x80};

// ---------------- FUNCTIONS ----------------
bool matchUID(byte *uid, byte *tag) {
  for (byte i = 0; i < 4; i++) {
    if (uid[i] != tag[i]) return false;
  }
  return true;
}

bool rfidAuthorized() {
  return matchUID(mfrc522.uid.uidByte, tag1) ||
         matchUID(mfrc522.uid.uidByte, tag2);
}

void grantAccess() {
  Serial.println("ACCESS GRANTED ✔");

  digitalWrite(LED_PIN, HIGH);
  delay(4000);
  digitalWrite(LED_PIN, LOW);
}

void denyAccess() {
  Serial.println("ACCESS DENIED ❌");

  for (int i = 0; i < 5; i++) {
    digitalWrite(LED_PIN, HIGH);
    delay(100);
    digitalWrite(LED_PIN, LOW);
    delay(100);
  }
}

// ---------------- SETUP ----------------
void setup() {
  Serial.begin(9600);
  SPI.begin();
  mfrc522.PCD_Init();

  pinMode(53, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  Serial.println("System Ready (RFID + Keypad)");
}

// ---------------- LOOP ----------------
void loop() {

  // ---------------- RFID PART ----------------
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {

    Serial.print("UID: ");
    for (byte i = 0; i < 4; i++) {
      Serial.print(mfrc522.uid.uidByte[i], HEX);
      Serial.print(" ");
    }
    Serial.println();

    if (rfidAuthorized()) {
      grantAccess();
    } else {
      denyAccess();
    }

    mfrc522.PICC_HaltA();
  }

  // ---------------- KEYPAD PART ----------------
  char key = keypad.getKey();

  if (key) {
    Serial.print(key);

    if (key == '#') {
      // submit password
      Serial.println();

      if (inputPassword == correctPassword) {
        grantAccess();
      } else {
        denyAccess();
      }

      inputPassword = ""; // reset
    }
    else if (key == '*') {
      inputPassword = ""; // clear
      Serial.println("\nCleared");
    }
    else {
      inputPassword += key;
    }
  }
}
