# Smart EV Charging Station Simulation ⚡🚗

This project simulates an Electric Vehicle charging station with safety locks and serial telemetry output using a PIC18F4580 microcontroller. 

**Why this is important:** EV chargers lock the cable plug into the vehicle socket during active high-power current flow to prevent arcing. Releasing the cable requires a coordinated mechanical lock disengagement.

## 🚀 Core Features

* **Plug Connection Detection:** Simulates plugging the charger into the car. The DC motor locks the plug in place (runs forward).
* **UART Telemetry:** While charging, the system continuously broadcasts voltage, current, and state-of-charge (SOC %) information over serial TX.
* **Manual Overrides:** Toggles the charge speed (Normal vs Fast charging rates).
* **Safety Cut-off:** Emergency Stop halts the process instantly, unlocks the plug (runs motor backward for 2 seconds), and alerts the operator.

## 🔌 Hardware Connections (Matrix)

| PIC18 Pin | Target Component | Functional Purpose |
| :--- | :--- | :--- |
| **RB0** | Connector Plug Insert Switch | Simulates plug connection detection (Active LOW) |
| **RB1** | Charge Speed Button | Normal vs Fast rate selector (Active LOW) |
| **RB2** | Emergency Stop Button | Safety shutdown override (Active LOW) |
| **RB4-RB7** | LCD D4-D7 | 4-bit mode data lines (Conflict-free PORTB) |
| **RA0, RA1** | LCD RS, EN | LCD Register Select and Enable control pins |
| **RC6 / TX** | Virtual Terminal RXD | Serial telemetry broadcaster (9600 Baud) |
| **RC7 / RX** | Virtual Terminal TXD | Command input link (Unused in this simulation) |
| **RD0, RD1** | L293D IN1, IN2 | Plug Solenoid Latch Actuator (DC Motor) |
| **RD4** | Green LED | Station Available / Fully Charged (Add 220Ω resistor) |
| **RD5** | Red LED | System Fault / Lock Error (Add 220Ω resistor) |
| **RD6** | Yellow LED | Charging Active status (Blinks during charging, add 220Ω) |

## 🧠 State-Machine Architecture

The firmware implements an Event-Driven Finite State Machine (FSM) using a simple control variable: `charge_state`. The microcontroller continuously loops, executing specific actions depending on the active state and polling the button pins and UART registers for transitions.

| State Index | Nominal Description | Actions | Next State Transition |
| :--- | :--- | :--- | :--- |
| **State 0** | Standby (Ready) | Green LED ON, Red & Yellow OFF, Motor stopped. LCD prints "STATUS: READY". Polls plug sensor. | Move to State 1 if RB0 goes LOW (Plug inserted). |
| **State 1** | Locking Cable | Runs cable lock DC motor Forward (RD0=1, RD1=0) for 2s. LCD prints "LOCKING PLUG". | Move to State 2 automatically after motor stops. |
| **State 2** | Charging Active | Green LED OFF. Yellow LED toggles/flashes. Broadcasts telemetry logs over UART TX. Polls charge speed toggle. | Move to State 3 when battery level reaches 100%. |
| **State 3** | Charge Complete | Yellow LED OFF, Green LED ON. Runs cable lock DC motor in Reverse (RD0=0, RD1=1) for 2s to release cable. | Move back to State 0 after a 4-second delay. |
| **State 4** | Emergency Fault | Triggered immediately if E-Stop button (RB2) is pressed. Stops motor, unlocks cable, and locks program. | System locked in infinite hazard loop. |

*Note: This code utilizes beginner-friendly direct pin control (e.g., `LATDbits.LATD0 = 1;`) rather than binary bit masks for maximum readability.*

## 🎮 How to Use & Test the Project

You can test the simulation using two distinct interactive methods inside Proteus:

### Method 1: Interactive Serial Keyboard Inputs
1. Run the Proteus simulation and locate the **Virtual Terminal** window. (Right-click device -> *Display Virtual Terminal* if hidden).
2. Click inside the terminal to focus.
3. Press `S` to simulate plugging in the cable.
4. Press `F` while charging to toggle Normal (AC) / Fast (DC) rates.
5. Press `K` to simulate aborting the process (motor safely unlocks).
6. Press `E` to trigger an immediate emergency stop routine.

### Method 2: Physical Hardware Push-Buttons
* Press **RB0** to simulate physical connector insertion.
* Press **RB1** to manually toggle between AC and DC charging speeds.
* Press **RB2** to initiate a physical E-Stop. The motor reverses immediately to release the connector, and the system shuts down.

## Circuit Diagram
![image](https://github.com/Gokulxu/SMART-EV-CHARGING-STN/blob/652b4720eb1daee9ebb032570329c8796b194915/Screenshot%202026-06-16%20201814.png)

## Code
# Smart EV Charging Station Controller (PIC18F4580) ⚡🚗

## 💡 Overview
This repository contains a bare-metal firmware solution for a **Smart Electric Vehicle (EV) Charging Station Controller** developed for the **PIC18F4580** microcontroller using the **Microchip XC8 compiler toolchain**. 

The application utilizes a deterministic finite state machine (FStateMachine) to safely manage automated cable latching/locking actuators, real-time State-of-Charge (SoC) calculations, parallel 4-bit LCD diagnostic diagnostics, and bidirectional **USART serial telemetry** for remote system control.

---

## 🔬 System States & Operation
The controller sequences through 5 operational states to manage vehicle charging cycles deterministically:

1. **Standby/Ready (`State 0`):** Station is active. Green status LED is lit. The system polls the physical proximity sensor (`RB0`) or an incoming UART command to detect a plugged-in vehicle.
2. **Locking Connector (`State 1`):** Initiates a 2-second forward DC motor actuation sequence via an **L293D H-Bridge** driver to physically lock the charging plug in place.
3. **Active Charging (`State 2`):** Green LED shuts off and a Yellow status LED flashes. State of Charge (SoC) percentages increment iteratively. Telemetry statistics (Voltage, Current, SoC) stream continuously out over the USART tx line at **9600 Baud**. Users can dynamically toggle between **Normal AC** and **Fast DC** charging models via physical hardware inputs (`RB1`) or remote terminal keypresses.
4. **Completed / Aborted (`State 3`):** Once a 100% capacity threshold is reached or an abort command is captured, the H-Bridge motor runs in reverse for 2 seconds to disengage the plug lock.
5. **Fault / Emergency Stop (`State 4`):** Activated instantly by a hardware E-Stop push button (`RB2`) or a remote hazard override. Safely cuts all vehicle power streams, triggers the reversing motor sequence to free the cable link, and traps execution in a permanent Red LED hazard flashing cycle.

---

## 📌 Hardware Pin Architecture

| Hardware Link | Peripheral Function / Register Group | Selected Pin | Data Direction | Functional Assignment |
| :--- | :--- | :--- | :--- | :--- |
| **Plug Insertion Sensor** | Digital GPIO Input | `RB0` | Input | Detects physical cable link connections |
| **Manual Speed Selector** | Digital GPIO Input | `RB1` | Input | Local hardware toggle for Fast/Normal charging |
| **Emergency Stop Button** | Digital GPIO Input | `RB2` | Input | High-priority active-low fault trigger switch |
| **L293D Motor Lines** | GPIO Port Drivers | `RD0`, `RD1` | Output | Drives H-Bridge direction inputs (IN1, IN2) |
| **Status LEDs Group** | GPIO Port Drivers | `RD4`, `RD5`, `RD6` | Output | Green (Ready), Red (Fault), Yellow (Charging) |
| **4-Bit LCD Data Lines** | High Nibble Data Bus | `RB4–RB7` | Output | Interfaces with `D4–D7` parallel LCD data pins |
| **LCD Controls** | Register Select / Enable | `RA0`, `RA1` | Output | Synchronizes 4-bit character data latching |
| **Serial Communication** | USART Transceiver Module | `RC6`, `RC7` | Bidirectional | `RC6` (TX Telemetry Out) / `RC7` (RX Command In) |

---

## 🧑‍💻 Complete Firmware Source Code

```c
#pragma config OSC = HS
#pragma config WDT = OFF
#pragma config PBADEN = OFF
#pragma config MCLRE = ON
#pragma config LVP = OFF

#include <xc.h>
#include <stdio.h>         // Required for sprintf()

#define _XTAL_FREQ 20000000 // 20 MHz Crystal Frequency

// --- FUNCTION PROTOTYPES ---
void delay_ms(unsigned int ms);
void delay_short(void);
void lcd_write_nibble(unsigned char nibble);
void command(unsigned char cmd);
void data(unsigned char val);
void lcd_print(const char* s);
void lcd_init(void);
void usart_init(void);
char usart_read(void);
void usart_write(char ch);
void usart_print(const char* s);

// --- CALIBRATED SOFTWARE DELAY MODULES ---
void delay_ms(unsigned int ms) {
    for (unsigned int i = 0; i < ms; i++) {
        for (int j = 0; j < 165; j++);
    }
}

void delay_short() {
    for (int i = 0; i < 500; i++);
}

// --- 4-BIT PARALLEL LCD DRIVER ---
void lcd_write_nibble(unsigned char nibble) {
    LATB = (LATB & 0x0F) | (nibble & 0xF0);
    PORTAbits.RA1 = 1; // EN = 1
    delay_short();
    PORTAbits.RA1 = 0; // EN = 0
}

void command(unsigned char cmd) {
    PORTAbits.RA0 = 0; // RS = 0
    lcd_write_nibble(cmd & 0xF0);
    lcd_write_nibble(cmd << 4);
    delay_ms(2);
}

void data(unsigned char val) {
    PORTAbits.RA0 = 1; // RS = 1
    lcd_write_nibble(val & 0xF0);
    lcd_write_nibble(val << 4);
    delay_ms(2);
}

void lcd_print(const char* s) {
    while(*s) {
        data(*s++);
    }
}

void lcd_init() {
    delay_ms(15);
    lcd_write_nibble(0x30);
    delay_ms(5);
    lcd_write_nibble(0x30);
    delay_ms(1);
    lcd_write_nibble(0x30);
    delay_ms(1);
    lcd_write_nibble(0x20); // Select 4-bit mode
    delay_ms(1);
    
    command(0x28); // 4-bit operation, 2-line array
    command(0x0C); // Display ON, Cursor OFF
    command(0x06); // Auto-increment index
    command(0x01); // Flush display buffer
    delay_ms(2);
}

// --- USART SERIAL TELEMETRY DRIVERS ---
void usart_init() {
    TRISCbits.TRISC7 = 1; // RX as input
    TRISCbits.TRISC6 = 0; // TX as output
    TXSTA = 0x24;         // Transmit Enabled, High speed
    RCSTA = 0x90;         // Continuous receive enabled, Serial Port Enabled
    SPBRG = 129;          // 9600 Baud at 20MHz crystal oscillator
}

char usart_read() {
    if (OERR) {
        CREN = 0;
        CREN = 1;
    }
    while (RCIF == 0);
    return RCREG;
}

void usart_write(char ch) {
    while (!TXIF); // Wait for buffer empty
    TXREG = ch;
}

void usart_print(const char* s) {
    while (*s) {
        usart_write(*s++);
    }
}

// --- MAIN CONTROLLER STATE MACHINE ---
void main() {
    ADCON1 = 0x0F; // Digital mode configurations
    
    TRISD = 0x00;  // PORTD as outputs (Motor & LEDs)
    TRISB = 0x07;  // RB0-RB2 inputs (buttons), RB4-RB7 outputs (LCD)
    TRISA = 0xFC;  // RA0-RA1 outputs (LCD RS, EN)
    
    // Set all outputs low initially
    LATDbits.LATD0 = 0; // Motor IN1
    LATDbits.LATD1 = 0; // Motor IN2
    LATDbits.LATD4 = 0; // Green LED
    LATDbits.LATD5 = 0; // Red LED
    LATDbits.LATD6 = 0; // Yellow LED
    LATDbits.LATD7 = 0; // Emergency Warning LED
    LATB = 0x00;
    
    lcd_init();
    usart_init();
    
    unsigned char charge_state = 0; // 0=Ready, 1=Locking, 2=Charging, 3=Completed, 4=Fault
    int battery_level = 0;          // State of Charge percentage
    int charge_rate = 1;            // 1=Normal, 2=Fast
    char buffer[32];
    
    command(0x80);
    lcd_print("EV CHARGING STN ");
    command(0xC0);
    lcd_print("STATUS: READY   ");
    
    usart_print("\r\n==========================================\r\n");
    usart_print("   Smart EV Charging Station Online\r\n");
    usart_print("==========================================\r\n");
    usart_print("Keyboard Commands (Type in Virtual Terminal):\r\n");
    usart_print("  [S] - Plug Cable & Start Charging\r\n");
    usart_print("  [F] - Toggle Normal/Fast Rate\r\n");
    usart_print("  [K] - Kill/Abort Charging Cycle\r\n");
    usart_print("  [E] - Trigger Emergency Stop\r\n");
    usart_print("------------------------------------------\r\n");
    
    while(1) {
        // --- 1. EMERGENCY STOP CHECK (RB2 Active LOW) ---
        if (PORTBbits.RB2 == 0) {
            charge_state = 4; // Force Fault state
        }
        
        // --- 2. SCAN SERIAL RX FOR KEYBOARD COMMANDS ---
        if (RCIF) {
            char cmd = usart_read();
            
            if ((cmd == 'S' || cmd == 's') && charge_state == 0) {
                charge_state = 1; 
            }
            else if ((cmd == 'F' || cmd == 'f') && charge_state == 2) {
                if (charge_rate == 1) {
                    charge_rate = 2;
                    usart_print("UART Command: FAST Charge Activated\r\n");
                } else {
                    charge_rate = 1;
                    usart_print("UART Command: NORMAL Charge Activated\r\n");
                }
            }
            else if ((cmd == 'K' || cmd == 'k') && charge_state == 2) {
                usart_print("UART Command: Aborting Charge Cycle...\r\n");
                charge_state = 3; 
            }
            else if (cmd == 'E' || cmd == 'e') {
                charge_state = 4; 
            }
        }
        
        // --- 3. FAULT / EMERGENCY SYSTEM DISCONNECT ---
        if (charge_state == 4) {
            LATDbits.LATD6 = 0; // Kill charging status indicator
            
            command(0x01);
            lcd_print("! EMERGENCY !");
            command(0xC0);
            lcd_print("UNLOCKING PLUG  ");
            usart_print("ALERT: Emergency stop triggered! Releasing connector plug...\r\n");
            
            // Reverse motor to unlock actuator (RD0=0, RD1=1)
            LATDbits.LATD0 = 0;
            LATDbits.LATD1 = 1;
            delay_ms(2000); 
            
            LATDbits.LATD0 = 0;
            LATDbits.LATD1 = 0;
            LATDbits.LATD5 = 1; // Solid red alert indicator lock
            
            command(0x01);
            lcd_print("SAFE TO UNPLUG  ");
            command(0xC0);
            lcd_print("RE-HOME REQUIRED");
            
            while(1) {
                // Persistent hazard loop indicator strobe
                LATDbits.LATD5 = !LATDbits.LATD5;
                delay_ms(250);
            }
        }
        
        // --- 4. CHARGER STANDBY STATE ---
        if (charge_state == 0) {
            LATDbits.LATD4 = 1; // Green LED ON
            LATDbits.LATD5 = 0; 
            LATDbits.LATD6 = 0; 
            
            if (PORTBbits.RB0 == 0) {
                delay_ms(50); // Software switch debounce
                if (PORTBbits.RB0 == 0) {
                    charge_state = 1; 
                }
            }
        }
        
        // --- 5. CONNECTOR PLUG LOCKING ACTUATOR ACTION ---
        if (charge_state == 1) {
            usart_print("Plug inserted. Locking connector pin...\r\n");
            command(0x01);
            lcd_print("CABLE CONNECTED ");
            command(0xC0);
            lcd_print("LOCKING PLUG... ");
            
            // Drive locking motor forward (RD0=1, RD1=0)
            LATDbits.LATD0 = 1;
            LATDbits.LATD1 = 0;
            delay_ms(2000); 
            
            LATDbits.LATD0 = 0;
            LATDbits.LATD1 = 0;
            
            battery_level = 0; 
            charge_state = 2; 
        }
        
        // --- 6. ACTIVE VEHICLE CHARGING LOOP ---
        if (charge_state == 2) {
            LATDbits.LATD4 = 0; 
            LATDbits.LATD6 = !LATDbits.LATD6; // Strobe yellow loop indicator
            
            if (PORTBbits.RB1 == 0) {
                delay_ms(50); 
                if (PORTBbits.RB1 == 0) {
                    if (charge_rate == 1) {
                        charge_rate = 2; 
                        usart_print("User Input: Fast Charge Activated\r\n");
                    } else {
                        charge_rate = 1; 
                        usart_print("User Input: Normal Charge Activated\r\n");
                    }
                }
            }
            
            // Charge Accumulation Mechanics
            if (charge_rate == 1) {
                battery_level = battery_level + 2; 
            } else {
                battery_level = battery_level + 5; 
            }
            
            if (battery_level >= 100) {
                battery_level = 100;
                charge_state = 3; 
            }
            
            command(0x80);
            if (charge_rate == 1) {
                lcd_print("RATE: NORMAL AC ");
            } else {
                lcd_print("RATE: FAST DC   ");
            }
            
            command(0xC0);
            sprintf(buffer, "BATTERY: %d%%    ", battery_level);
            lcd_print(buffer);
            
            // Output Serial Telemetry
            if (charge_rate == 1) {
                sprintf(buffer, "TX_DATA: SOC=%d%%, V=230V, I=32A\r\n", battery_level);
            } else {
                sprintf(buffer, "TX_DATA: SOC=%d%%, V=400V, I=120A\r\n", battery_level);
            }
            usart_print(buffer);
            
            delay_ms(800); 
        }
        
        // --- 7. CHARGING CYCLE COMPLETED ---
        if (charge_state == 3) {
            LATDbits.LATD6 = 0; 
            LATDbits.LATD4 = 1; 
            
            command(0x01);
            lcd_print("CHARGE COMPLETE!");
            command(0xC0);
            lcd_print("UNLOCKING PLUG  ");
            usart_print("Vehicle battery fully charged! Releasing connector plug...\r\n");
            
            // Reverse motor tracking sequence (RD0=0, RD1=1)
            LATDbits.LATD0 = 0;
            LATDbits.LATD1 = 1;
            delay_ms(2000); 
            
            LATDbits.LATD0 = 0;
            LATDbits.LATD1 = 0;
            
            command(0x01);
            lcd_print("SAFE TO REMOVE  ");
            command(0xC0);
            lcd_print("THANK YOU       ");
            delay_ms(4000);
            
            charge_state = 0; 
        }
        delay_ms(100);
    }
}
