// ============================================================
// Motor Pins (Stepper Motor)
// ============================================================
const int IN1 = 2;
const int IN2 = 4;
const int IN3 = 7;
const int IN4 = 8;

// ============================================================
// Ultrasonic Sensor Pins
// ============================================================
const int trigPin = 9;
const int echoPin = 10;

// ============================================================
// Bluetooth Pins (HC-05 / HC-06)
// ============================================================
#include <SoftwareSerial.h>
SoftwareSerial bluetooth(11, 12);  // RX = 11, TX = 12

// ============================================================
// Constants & Variables
// ============================================================
const int safetyDistance = 35;      // Safety distance (cm) - brake if closer
const int turnDistance = 25;        // If obstacle closer than this, turn
int distance = 0;
char bluetoothCommand = '';         // Command from Bluetooth
bool autoMode = true;               // Auto mode ON/OFF (toggle via Bluetooth)

// Movement states
enum MoveState {
  FORWARD,
  BACKWARD,
  TURN_LEFT,
  TURN_RIGHT,
  STOP
};
MoveState currentMove = FORWARD;

// ============================================================
// Read distance from Ultrasonic Sensor
// ============================================================
int readDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  long duration = pulseIn(echoPin, HIGH, 30000);
  int dist = duration * 0.034 / 2;
  
  if (dist < 2) dist = 1;
  if (dist > 400) dist = 400;
  
  return dist;
}

// ============================================================
// Motor Control Functions (Single Step)
// ============================================================

// One step forward
void stepForward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(10);
}

// One step backward
void stepBackward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  delay(10);
}

// One step turning left
void stepTurnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  delay(10);
}

// One step turning right
void stepTurnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(10);
}

// Stop all motors (brake)
void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  delay(5);
}

// ============================================================
// Continuous Movement Functions
// ============================================================

void moveContinuousForward(int steps = 20) {
  for (int i = 0; i < steps; i++) {
    stepForward();
  }
  stopMotors();
}

void moveContinuousBackward(int steps = 15) {
  for (int i = 0; i < steps; i++) {
    stepBackward();
  }
  stopMotors();
}

void turnLeft(int steps = 12) {
  for (int i = 0; i < steps; i++) {
    stepTurnLeft();
  }
  stopMotors();
}

void turnRight(int steps = 12) {
  for (int i = 0; i < steps; i++) {
    stepTurnRight();
  }
  stopMotors();
}

// ============================================================
// Obstacle Avoidance Logic
// ============================================================
void avoidObstacle() {
  Serial.println("Obstacle detected! Avoiding...");
  
  // Step 1: Stop immediately
  stopMotors();
  delay(100);
  
  // Step 2: Move backward a little
  Serial.println("Moving backward...");
  moveContinuousBackward(15);
  delay(100);
  
  // Step 3: Turn right (or left - you can randomize)
  Serial.println("Turning right...");
  turnRight(18);
  delay(100);
  
  // Step 4: Check if path is clear
  int newDistance = readDistance();
  if (newDistance > safetyDistance) {
    Serial.println("Path clear! Moving forward again.");
  } else {
    Serial.println("Still blocked! Turning more...");
    turnRight(10);
  }
}

// ============================================================
// Process Bluetooth Commands
// ============================================================
void processBluetoothCommand() {
  if (bluetooth.available()) {
    bluetoothCommand = bluetooth.read();
    
    switch (bluetoothCommand) {
      case 'F':  // Forward
      case 'f':
        autoMode = false;
        currentMove = FORWARD;
        Serial.println("Bluetooth: FORWARD");
        moveContinuousForward(25);
        break;
        
      case 'B':  // Backward
      case 'b':
        autoMode = false;
        currentMove = BACKWARD;
        Serial.println("Bluetooth: BACKWARD");
        moveContinuousBackward(25);
        break;
        
      case 'L':  // Left
      case 'l':
        autoMode = false;
        currentMove = TURN_LEFT;
        Serial.println("Bluetooth: LEFT");
        turnLeft(15);
        break;
        
      case 'R':  // Right
      case 'r':
        autoMode = false;
        currentMove = TURN_RIGHT;
        Serial.println("Bluetooth: RIGHT");
        turnRight(15);
        break;
        
      case 'S':  // Stop
      case 's':
        autoMode = false;
        currentMove = STOP;
        Serial.println("Bluetooth: STOP");
        stopMotors();
        break;
        
      case 'A':  // Auto mode ON
      case 'a':
        autoMode = true;
        Serial.println("Bluetooth: AUTO MODE ENABLED");
        break;
        
      case 'M':  // Manual mode (same as 'S' basically)
      case 'm':
        autoMode = false;
        currentMove = STOP;
        Serial.println("Bluetooth: MANUAL MODE - Use F/B/L/R");
        stopMotors();
        break;
        
      default:
        // Unknown command - ignore
        break;
    }
  }
}

// ============================================================
// Auto Mode Movement (with obstacle avoidance)
// ============================================================
void runAutoMode() {
  // Read distance from sensor
  distance = readDistance();
  
  // Print for debugging
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  // Check for obstacles
  if (distance <= safetyDistance) {
    // Obstacle detected! Stop and avoid
    stopMotors();
    delay(50);
    avoidObstacle();
  } 
  else {
    // Path is clear - move forward slowly (step by step)
    stepForward();
    delay(15);  // Small pause between steps
  }
}

// ============================================================
// Setup Function
// ============================================================
void setup() {
  // Motor pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  // Sensor pins
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  
  // Start serial for debugging
  Serial.begin(9600);
  
  // Start Bluetooth serial
  bluetooth.begin(9600);
  
  Serial.println("=== CAR IS READY ===");
  Serial.println("Bluetooth Commands:");
  Serial.println("F = Forward | B = Backward | L = Left | R = Right | S = Stop");
  Serial.println("A = Auto Mode (obstacle avoidance) | M = Manual Mode");
  Serial.println("=========================");
  
  // Initial stop
  stopMotors();
  delay(1000);
}

// ============================================================
// Main Loop
// ============================================================
void loop() {
  // Check for Bluetooth commands first
  processBluetoothCommand();
  
  // Run auto mode if enabled
  if (autoMode) {
    runAutoMode();
  }
  
  // Small delay to stabilize
  delay(20);
}

// ============================================================
// BLUETOOTH COMMAND REFERENCE (Send from phone app):
// ============================================================
// F or f = Move Forward
// B or b = Move Backward  
// L or l = Turn Left
// R or r = Turn Right
// S or s = Stop
// A or a = Enable Auto Mode (obstacle avoidance)
// M or m = Manual Mode (stop and wait for commands)
// ============================================================
