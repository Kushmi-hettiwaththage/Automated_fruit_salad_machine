# Automated_fruit_salad_machine
#include <Keypad.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>
#include <math.h>
#include <HX711_ADC.h>

//------------------------ Pin Definitions ------------------------
#define beltMotorSpeed 10
#define beltMotorLeft 8
#define beltMotorRight 9
#define mixerMotorSpeed 11
#define mixerMotorLeft 12
#define mixerMotorRight 13

const int loadCellDataPin = 4;
const int loadCellClockPin = 5;

HX711_ADC loadCell(loadCellDataPin, loadCellClockPin);

Servo pusherServo;

LiquidCrystal_I2C lcdDisplay(0x27, 16, 2);

int potentiometerPin = A0;

const byte totalRows = 4;
const byte totalCols = 4;
char keyMap[totalRows][totalCols] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[totalRows] = {22, 23, 24, 25};
byte colPins[totalCols] = {26, 27, 28, 29};
Keypad keypadInput = Keypad(makeKeymap(keyMap), rowPins, colPins, totalRows, totalCols);

String enteredPassword = "";
String masterPassword = "A";
bool isSystemActive = false;

enum DisplayMode {
  WELCOME_SCREEN, PASSWORD_ENTRY, MENU_SCREEN, WEIGHT_MEASURE, SALT_SUGAR_CALC,
  PUSHER_ACTIVATE, BELT_START, MIXER_START, WASHER_START,
  BELT_STOP, MIXER_STOP, WASHER_STOP,
  EMERGENCY_HALT, SYSTEM_RESET
};
DisplayMode currentMode = WELCOME_SCREEN;

float fruitWeight = 0.0;
float saltAmount = 0.0;
float sugarAmount = 0.0;

bool isBeltRunning = false;
bool isMixerRunning = false;
bool isWasherRunning = false;

//------------------------ Setup ------------------------
void setup() {
  pinMode(beltMotorSpeed, OUTPUT);
  pinMode(beltMotorLeft, OUTPUT);
  pinMode(beltMotorRight, OUTPUT);
  pinMode(mixerMotorSpeed, OUTPUT);
  pinMode(mixerMotorLeft, OUTPUT);
  pinMode(mixerMotorRight, OUTPUT);

  pusherServo.attach(3);

  lcdDisplay.begin();
  lcdDisplay.backlight();
  lcdDisplay.clear();

  Serial.begin(9600);
  delay(1000);

  lcdDisplay.setCursor(0, 0);
  lcdDisplay.print("Welcome!");
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print("Enter Password:");
  currentMode = PASSWORD_ENTRY;
}

//------------------------ Main Loop ------------------------
void loop() {
  switch (currentMode) {
    case PASSWORD_ENTRY:
      handlePasswordEntry();
      break;
    case MENU_SCREEN:
      showMainMenu();
      break;
    case WEIGHT_MEASURE:
      measureWeight();
      break;
    case SALT_SUGAR_CALC:
      calculateSaltSugar();
      break;
    case PUSHER_ACTIVATE:
      runPusher();
      break;
    case BELT_START:
      startBelt();
      break;
    case MIXER_START:
      startMixer();
      break;
    case WASHER_START:
      startWasher();
      break;
    case BELT_STOP:
      stopBelt();
      break;
    case MIXER_STOP:
      stopMixer();
      break;
    case WASHER_STOP:
      stopWasher();
      break;
    case EMERGENCY_HALT:
      handleEmergencyStop();
      break;
    case SYSTEM_RESET:
      restartSystem();
      break;
    case WELCOME_SCREEN:
    default:
      lcdDisplay.clear();
      lcdDisplay.print("Welcome!");
      lcdDisplay.setCursor(0, 1);
      lcdDisplay.print("Enter Password:");
      currentMode = PASSWORD_ENTRY;
      break;
  }
}

//------------------------ Screen Handlers ------------------------
void handlePasswordEntry() {
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print("                ");
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print(enteredPassword);

  char keyPressed = keypadInput.getKey();
  if (keyPressed) {
    if (keyPressed == '#') {
      if (enteredPassword == masterPassword) {
        isSystemActive = true;
        lcdDisplay.clear();
        lcdDisplay.print("Access Granted!");
        delay(1000);
        currentMode = MENU_SCREEN;
      } else {
        lcdDisplay.clear();
        lcdDisplay.print("Wrong Password!");
        delay(1000);
        lcdDisplay.clear();
        lcdDisplay.print("Enter Password:");
        enteredPassword = "";
      }
    } else if (keyPressed == '*') {
      enteredPassword = "";
    } else {
      if (enteredPassword.length() < 8) enteredPassword += keyPressed;
    }
  }
}

void showMainMenu() {
  lcdDisplay.setCursor(0, 0);
  lcdDisplay.print("Hello! Choose:");
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print("1-9 0AB");

  char keyPressed = keypadInput.getKey();
  if (keyPressed) {
    switch (keyPressed) {
      case '1': currentMode = WEIGHT_MEASURE; break;
      case '2': currentMode = SALT_SUGAR_CALC; break;
      case '3': currentMode = PUSHER_ACTIVATE; break;
      case '4': currentMode = BELT_START; break;
      case '5': currentMode = WASHER_START; break;
      case '6': currentMode = MIXER_START; break;
      case '7': currentMode = BELT_STOP; break;
      case '8': currentMode = MIXER_STOP; break;
      case '9': currentMode = WASHER_STOP; break;
      case '0': currentMode = EMERGENCY_HALT; break;
      case 'A': currentMode = MENU_SCREEN; break;
      case 'B': currentMode = SYSTEM_RESET; break;
      default: break;
    }
    lcdDisplay.clear();
  }
}

void measureWeight() {
  loadCell.begin();
  loadCell.start(2000);
  loadCell.setCalFactor(353.01);
  loadCell.tare();

  lcdDisplay.clear();
  lcdDisplay.print("Place Fruit...");
  delay(1000);

  float previousWeight = -99999.0;

  lcdDisplay.clear();
  lcdDisplay.print("Weight:");
  while (true) {
    loadCell.update();
    float currentWeight = loadCell.getData();

    if (fabs(currentWeight - previousWeight) > 0.1) {
      previousWeight = currentWeight;
      fruitWeight = currentWeight;
      lcdDisplay.setCursor(0, 1);
      lcdDisplay.print("                ");
      lcdDisplay.setCursor(0, 1);
      lcdDisplay.print(fruitWeight, 2);
      lcdDisplay.print(" g");
    }

    char keyPressed = keypadInput.getKey();
    if (keyPressed == 'A') { currentMode = MENU_SCREEN; lcdDisplay.clear(); break; }
    else if (keyPressed == '2') { currentMode = SALT_SUGAR_CALC; lcdDisplay.clear(); break; }
    else if (keyPressed == 'B') { currentMode = SYSTEM_RESET; lcdDisplay.clear(); break; }

    delay(100);
  }
}

void calculateSaltSugar() {
  saltAmount = fruitWeight * 0.025;
  sugarAmount = fruitWeight * 0.075;

  lcdDisplay.clear();
  lcdDisplay.print("Salt: ");
  lcdDisplay.print(saltAmount, 2);
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print("Sugar: ");
  lcdDisplay.print(sugarAmount, 2);
  delay(7000);

  lcdDisplay.clear();
  lcdDisplay.print("A:Menu B:Reset");

  while (true) {
    char keyPressed = keypadInput.getKey();
    if (keyPressed == 'A') { currentMode = MENU_SCREEN; lcdDisplay.clear(); break; }
    else if (keyPressed == 'B') { currentMode = SYSTEM_RESET; lcdDisplay.clear(); break; }
  }
}

void runPusher() {
  lcdDisplay.clear();
  lcdDisplay.print("Pushing...");
  operatePusher();
  lcdDisplay.clear();
  lcdDisplay.print("Done!");
  delay(1000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void startBelt() {
  lcdDisplay.clear();
  lcdDisplay.print("Belt Starting...");
  digitalWrite(beltMotorLeft, HIGH);
  digitalWrite(beltMotorRight, LOW);
  int potValue = analogRead(potentiometerPin);
  int motorSpeed = map(potValue, 0, 1023, 0, 255);
  analogWrite(beltMotorSpeed, motorSpeed);
  isBeltRunning = true;
  delay(2000);

  lcdDisplay.clear();
  lcdDisplay.print("7:Stop A:Menu");
  while (isBeltRunning) {
    char keyPressed = keypadInput.getKey();
    if (keyPressed == '7') { currentMode = BELT_STOP; break; }
    else if (keyPressed == 'A') { currentMode = MENU_SCREEN; break; }
    else if (keyPressed == '0') { currentMode = EMERGENCY_HALT; break; }
    potValue = analogRead(potentiometerPin);
    motorSpeed = map(potValue, 0, 1023, 0, 255);
    analogWrite(beltMotorSpeed, motorSpeed);
    delay(100);
  }
  lcdDisplay.clear();
}

void startWasher() {
  lcdDisplay.clear();
  lcdDisplay.print("Washer Not Impl");
  delay(2000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void startMixer() {
  lcdDisplay.clear();
  lcdDisplay.print("Mixer Starting");
  digitalWrite(mixerMotorLeft, HIGH);
  digitalWrite(mixerMotorRight, LOW);
  analogWrite(mixerMotorSpeed, 200);
  isMixerRunning = true;
  delay(2000);
  lcdDisplay.clear();
  lcdDisplay.print("8:Stop A:Menu");
  while (isMixerRunning) {
    char keyPressed = keypadInput.getKey();
    if (keyPressed == '8') { currentMode = MIXER_STOP; break; }
    else if (keyPressed == 'A') { currentMode = MENU_SCREEN; break; }
    else if (keyPressed == '0') { currentMode = EMERGENCY_HALT; break; }
    delay(100);
  }
  lcdDisplay.clear();
}

void stopBelt() {
  lcdDisplay.clear();
  lcdDisplay.print("Stopping Belt");
  digitalWrite(beltMotorLeft, LOW);
  digitalWrite(beltMotorRight, LOW);
  analogWrite(beltMotorSpeed, 0);
  isBeltRunning = false;
  delay(1000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void stopMixer() {
  lcdDisplay.clear();
  lcdDisplay.print("Stopping Mixer...");
  digitalWrite(mixerMotorLeft, LOW);
  digitalWrite(mixerMotorRight, LOW);
  analogWrite(mixerMotorSpeed, 0);
  isMixerRunning = false;
  delay(1000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void stopWasher() {
  lcdDisplay.clear();
  lcdDisplay.print("Washer Not Impl");
  delay(1000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void handleEmergencyStop() {
  lcdDisplay.clear();
  lcdDisplay.print("EMERGENCY STOP!");
  digitalWrite(beltMotorLeft, LOW);
  digitalWrite(beltMotorRight, LOW);
  analogWrite(beltMotorSpeed, 0);
  digitalWrite(mixerMotorLeft, LOW);
  digitalWrite(mixerMotorRight, LOW);
  analogWrite(mixerMotorSpeed, 0);
  isBeltRunning = false;
  isMixerRunning = false;
  isSystemActive = false;
  delay(2000);
  currentMode = MENU_SCREEN;
  lcdDisplay.clear();
}

void restartSystem() {
  lcdDisplay.clear();
  lcdDisplay.print("Restarting...");
  delay(1500);
  fruitWeight = 0.0;
  saltAmount = 0.0;
  sugarAmount = 0.0;
  isBeltRunning = false;
  isMixerRunning = false;
  isWasherRunning = false;
  enteredPassword = "";
  isSystemActive = false;
  lcdDisplay.clear();
  lcdDisplay.print("Welcome!");
  lcdDisplay.setCursor(0, 1);
  lcdDisplay.print("Enter Password:");
  currentMode = PASSWORD_ENTRY;
}

void operatePusher() {
  int pushStart = 10, pushEnd = 170, moveSteps = 100, moveDelay = 20;
  for (int i = 0; i <= moveSteps; i++) {
    float progress = (float)i / moveSteps;
    float easedProgress = (1 - cos(progress * PI)) / 2;
    int angle = pushStart + (pushEnd - pushStart) * easedProgress;
    pusherServo.write(angle);
    delay(moveDelay);
  }
  delay(800);
  for (int i = 0; i <= moveSteps; i++) {
    float progress = (float)i / moveSteps;
    float easedProgress = (1 - cos(progress * PI)) / 2;
    int angle = pushEnd - (pushEnd - pushStart) * easedProgress;
    pusherServo.write(angle);
    delay(moveDelay);
  }
  delay(800);
}
