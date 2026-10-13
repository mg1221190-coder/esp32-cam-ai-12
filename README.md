/*
  AUTOMATIC WATER LEVEL INDICATOR
  + AUTOMATIC WATER PUMP CONTROLLER

  Board: Arduino UNO
  Sensor: HC-SR04 Ultrasonic Sensor
  Relay: 5V Relay Module
  Pump: Low-voltage DC water pump

  PUMP CONTROL:
  Distance > 3 cm  : Pump ON
  Distance <= 2 cm : Pump OFF
  Distance 2-3 cm  : Maintain previous state

  DISPLAY:
  Water distance, level percentage and pump status
*/

// ========================================
// PIN CONFIGURATION
// ========================================

const byte TRIG_PIN  = 9;
const byte ECHO_PIN  = 10;
const byte RELAY_PIN = 7;

// ========================================
// TANK CONFIGURATION
// ========================================

// Distance from the sensor face to the
// bottom of the empty tank, in centimetres.

const float TANK_DEPTH_CM = 60.0;

// Pump control thresholds.

const float PUMP_START_DISTANCE_CM = 3.0;
const float PUMP_STOP_DISTANCE_CM  = 2.0;

// Most 5V relay modules are active LOW.
// If your relay works in reverse, change
// true to false.

const bool RELAY_ACTIVE_LOW = true;

// ========================================
// VARIABLES
// ========================================

bool pumpOn = false;

// ========================================
// PUMP CONTROL FUNCTION
// ========================================

void setPump(bool turnOn) {

  // Avoid repeatedly switching the relay
  // when its state has not changed.

  if (turnOn == pumpOn) {
    return;
  }

  pumpOn = turnOn;

  if (pumpOn) {

    digitalWrite(
      RELAY_PIN,
      RELAY_ACTIVE_LOW ? LOW : HIGH
    );

    Serial.println("PUMP STATUS: ON");

  } else {

    digitalWrite(
      RELAY_PIN,
      RELAY_ACTIVE_LOW ? HIGH : LOW
    );

    Serial.println("PUMP STATUS: OFF");
  }
}

// ========================================
// ULTRASONIC DISTANCE MEASUREMENT
// ========================================

float readDistanceCm() {

  const byte MAX_SAMPLES = 5;
  float samples[MAX_SAMPLES];
  byte count = 0;

  // Take five readings.

  for (byte i = 0; i < MAX_SAMPLES; i++) {

    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);

    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);

    digitalWrite(TRIG_PIN, LOW);

    unsigned long duration =
      pulseIn(ECHO_PIN, HIGH, 30000UL);

    if (duration > 0) {

      float distance =
        duration * 0.0343 / 2.0;

      // Accept positive readings up to 400 cm.
      // Keeping readings below 2 cm allows a
      // reported close distance to trigger OFF.

      if (distance > 0.0 && distance <= 400.0) {

        samples[count] = distance;
        count++;
      }
    }

    delay(30);
  }

  // Require at least three valid readings.

  if (count < 3) {
    return -1;
  }

  // Sort readings to obtain the median.

  for (byte i = 0; i < count - 1; i++) {

    for (byte j = i + 1; j < count; j++) {

      if (samples[j] < samples[i]) {

        float temp = samples[i];
        samples[i] = samples[j];
        samples[j] = temp;
      }
    }
  }

  return samples[count / 2];
}

// ========================================
// SETUP
// ========================================

void setup() {

  // Configure the relay to start OFF.

  digitalWrite(
    RELAY_PIN,
    RELAY_ACTIVE_LOW ? HIGH : LOW
  );

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  digitalWrite(TRIG_PIN, LOW);

  pumpOn = false;

  Serial.begin(9600);

  Serial.println();
  Serial.println("================================");
  Serial.println(" AUTOMATIC WATER LEVEL SYSTEM");
  Serial.println("================================");

  Serial.println("Pump initially OFF");
  Serial.println("System starting...");
}

// ========================================
// MAIN LOOP
// ========================================

void loop() {

  float distance = readDistanceCm();

  // --------------------------------------
  // SENSOR ERROR HANDLING
  // --------------------------------------

  if (distance < 0) {

    // Turn the pump OFF if the sensor
    // cannot provide reliable measurements.

    setPump(false);

    Serial.println();
    Serial.println("SENSOR ERROR!");
    Serial.println("Check sensor wiring and position.");
    Serial.println("Pump commanded OFF");
    Serial.println("--------------------------------");

    delay(1000);
    return;
  }

  // --------------------------------------
  // WATER LEVEL CALCULATION
  // --------------------------------------

  float waterLevelPercent =
    ((TANK_DEPTH_CM - distance) /
     TANK_DEPTH_CM) * 100.0;

  // Restrict displayed percentage to 0-100%.

  if (waterLevelPercent < 0) {
    waterLevelPercent = 0;
  }

  if (waterLevelPercent > 100) {
    waterLevelPercent = 100;
  }

  // --------------------------------------
  // AUTOMATIC PUMP CONTROL
  // --------------------------------------

  // If the pump is running and water reaches
  // 2 cm from the sensor, turn the pump OFF.

  if (pumpOn && distance <= PUMP_STOP_DISTANCE_CM) {

    setPump(false);
  }

  // If the pump is OFF and water is farther
  // than 3 cm from the sensor, start filling.

  else if (!pumpOn &&
           distance > PUMP_START_DISTANCE_CM) {

    setPump(true);
  }

  // --------------------------------------
  // WATER LEVEL INDICATOR
  // --------------------------------------

  Serial.println();
  Serial.println("====== TANK STATUS ======");

  Serial.print("Distance from sensor: ");
  Serial.print(distance, 1);
  Serial.println(" cm");

  Serial.print("Water level: ");
  Serial.print(waterLevelPercent, 1);
  Serial.println("%");

  Serial.print("Pump: ");

  if (pumpOn) {
    Serial.println("ON");
  } else {
    Serial.println("OFF");
  }

  Serial.println("=========================");

  delay(500);
}
