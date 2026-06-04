# Whack-a-Mole Reaction Game (Arduino)

An Arduino-based reaction-time game built and simulated in Tinkercad. One of four colored LEDs lights up at a random moment — the player must hit the matching button as fast as possible. The quicker the reaction, the higher the score. After five rounds the game reports the final score and resets.

This was built as a practice project in Arduino programming, focusing on non-blocking timing with `millis()`, digital I/O, button debouncing, and state management.

<img width="662" height="561" alt="Screenshot 2026-06-04 111551" src="https://github.com/user-attachments/assets/05553582-ef2d-4d3c-9c11-0e0dd8de9440" />

## How to play

1. Power on the Arduino. The game waits a random interval (2–7 seconds) before lighting a mole.
2. One of the four colored LEDs (red, yellow, blue, green) lights up at random.
3. Press the **matching colored button** as quickly as you can.
4. Your reaction time is converted into points — a faster press earns more points (up to ~1000 per round).
5. If you don't press within **1 second**, the round scores zero and moves on.
6. After **5 rounds**, the game prints `GAME OVER NEW ROUND STARTING` to the Serial Monitor and resets the score.

Scores and round numbers are printed to the **Serial Monitor at 9600 baud**.

## Scoring

Points per round are based on reaction time using a linear mapping:

```
score += map(reactionTime, 0, 1000, 1000, 0);
```

- An instant press (≈0 ms) earns close to **1000 points**.
- A press near the **1000 ms** cutoff earns close to **0 points**.
- No press within 1 second earns **0** and the round advances.

## Components

| Component | Quantity | Purpose |
|-----------|----------|---------|
| Arduino Uno | 1 | Microcontroller running the game |
| Breadboard | 1 | Circuit assembly |
| LEDs (red, yellow, blue, green) | 4 | The "moles" the player reacts to |
| LED (red) — "lose" indicator | 1 | Reserved status LED (pin 13) |
| LED (green) — "win" indicator | 1 | Reserved status LED (pin 8) |
| Push buttons | 4 | One per colored mole LED |
| Resistors | several | LED current limiting and button pull-downs |
| Jumper wires | — | Connections |

## Pin assignments

**Mole LEDs (outputs):**

| LED | Arduino pin |
|-----|-------------|
| Red (`LEDR`) | 12 |
| Yellow (`LEDY`) | 11 |
| Blue (`LEDB`) | 10 |
| Green (`LEDG`) | 9 |

**Buttons (inputs):**

| Button | Arduino pin |
|--------|-------------|
| Red (`pb_red`) | 7 |
| Yellow (`pb_yellow`) | 6 |
| Blue (`pb_blue`) | 5 |
| Green (`pb_green`) | 4 |

**Status LEDs:**

| LED | Arduino pin |
|-----|-------------|
| Lose indicator (`LED_LOSE`) | 13 |
| Win indicator (`LED_WIN`) | 8 |

## How the code works

The program is written to be **non-blocking** — it never uses `delay()`, so the Arduino stays responsive to button presses while timing events. It tracks state with a few flags and timestamps captured from `millis()`:

- `startround` — when true, a new random wait (2–7 s) is chosen before the next mole appears.
- `waitingforpress` — true while a mole LED is lit and the program is waiting for the player to react.
- `pastbutton` — stores the previous button reading so a press is only counted once on the rising edge (edge detection / simple debouncing).

Each round:

1. A random delay elapses, then a random LED (`random(0,4)`) is lit and the on-time is recorded.
2. If the matching button is pressed, the reaction time is measured, converted to points, and the round advances.
3. If 1 second passes with no correct press, the round scores zero and advances.
4. After 5 rounds, the score and counter reset for a new game.

A random seed is taken from an unconnected analog pin (`analogRead(0)`) so the LED sequence differs each playthrough.

## The code
#define LED_LOSE 13
#define LEDR 12
#define LEDY 11
#define LEDB 10
#define LEDG 9
#define LED_WIN 8

#define pb_red 7
#define pb_yellow 6
#define pb_blue 5
#define pb_green 4

//define functions prototypes
unsigned long waitTime = 0;
unsigned long LEDonTime = 0;
unsigned long now = 0;
unsigned long buttonPressTime = 0;
unsigned long LEDwaittime = 0;
bool pastbutton = LOW;
bool startround = true;
bool waitingforpress = false;
int count = 1;
int LEDpins[4] = {12,11,10,9};
int PBpins[4] = {7,6,5,4};
int randpin;
unsigned long reactiontime = 0;
unsigned long score = 0;
//initialization
void setup()
{
 pinMode(pb_red, INPUT);
 pinMode(pb_yellow, INPUT);
 pinMode(pb_blue, INPUT);
 pinMode(pb_green, INPUT);
 pinMode(LED_LOSE, OUTPUT);
 pinMode(LED_WIN, OUTPUT);
 pinMode(LEDR, OUTPUT);
 pinMode(LEDY, OUTPUT);
 pinMode(LEDB, OUTPUT);
 pinMode(LEDG, OUTPUT);
 
 randomSeed(analogRead(0));
 Serial.begin(9600);
   //configure pin modes (inputs, outputs) etc.
}


//main program loop
void loop()
{
  now = millis();
  if(count <= 5){
    
    if(startround == true){
      waitTime = random(2000, 7000);
      startround = false;
      LEDwaittime = now;
    }
    if(now - LEDwaittime >= waitTime && waitingforpress == false){
      randpin = random(0,4);
      digitalWrite(LEDpins[randpin], HIGH);
      LEDonTime = now;
      waitingforpress = true;
    }
    int buttonstate = digitalRead(PBpins[randpin]);
    
    if(buttonstate == HIGH && pastbutton == LOW && waitingforpress == true){
      buttonPressTime = now;
      waitingforpress = false;
      reactiontime = buttonPressTime - LEDonTime;
      digitalWrite(LEDpins[randpin], LOW);
      score += map(reactiontime, 0, 1000, 1000, 0);
      Serial.println(count);
      Serial.println(score);
      count++;
      startround = true;
    }
    if(now - LEDonTime >= 1000 && waitingforpress == true){
      score += 0;
	  Serial.println(count);
      Serial.println(score);
      digitalWrite(LEDpins[randpin], LOW);
      count++;
      waitingforpress = false;
      startround = true;
    }
    pastbutton = buttonstate;
    
    
  } else {
    Serial.println("GAME OVER NEW ROUND STARTING");
    count = 1;
    score = 0;
    
  }
}

## Notes & possible improvements

This is a learning project, and a few things would make good next steps:

- **Buttons use `INPUT` (not `INPUT_PULLUP`)**, so they rely on external pull-down resistors. Switching to `INPUT_PULLUP` would simplify the wiring.
- **The win/lose LEDs (pins 8 and 13) are defined and initialized but not yet used** in the game logic — a natural addition would be lighting the win LED on a fast hit and the lose LED on a miss.
- **The final score isn't displayed before the reset** — printing a clear end-of-game summary (and perhaps a high score) would improve the feedback.
- **Only the correct button is checked.** Pressing the wrong color currently has no effect; penalizing wrong presses would make the game harder.
- A **buzzer** could add audio feedback for hits and misses.

## What I learned

- Writing non-blocking Arduino code with `millis()` instead of `delay()`
- Detecting button presses on the rising edge to avoid repeat counts
- Using `random()` and `randomSeed()` for unpredictable gameplay
- Mapping a measured value (reaction time) into a score with `map()`
- Managing game state across loop iterations with flags and timestamps

## License

Released under the MIT License — feel free to use this as a reference.
