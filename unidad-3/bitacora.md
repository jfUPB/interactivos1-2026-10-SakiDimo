# Unidad 3

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 4

```js
// ===============================
// VARIABLES GLOBALES
// ===============================

let port;
let reader;
let connectBtn;

const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",        // Botón A
  INC: "B",        // Botón B
  START: "S",      // Shake
  TICK: "Timeout",
};

let temporizador;
let renderer = new Map();

// ===============================
// CLASE TEMPORIZADOR (FSM)
// ===============================

class Temporizador extends FSMTask {

  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;

    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.sequence = [];   // buffer para detectar A-B-A

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);

    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  // ===============================
  // ESTADO CONFIG
  // ===============================

  estado_config = (ev) => {

    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
      this.sequence = [];
    }

    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue)
        this.configValue--;
    }

    else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue)
        this.configValue++;
    }

    else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  // ===============================
  // ESTADO ARMED (corriendo)
  // ===============================

  estado_armed = (ev) => {

    if (ev === ENTRY) {
      this.sequence = [];
      this.myTimer.start();
    }

    else if (ev === EVENTS.TICK) {

      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;

        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    }

    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {

      // Detectar secuencia A-B-A
      if (this.checkSequence(ev)) {
        this.transitionTo(this.estado_config);
        return;
      }

      // Si solo fue A (sin completar secuencia) → pausa
      if (ev === EVENTS.DEC) {
        this.transitionTo(this.estado_pause);
      }
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };

  // ===============================
  // ESTADO PAUSE
  // ===============================

  estado_pause = (ev) => {

    if (ev === ENTRY) {
      this.myTimer.stop();
    }

    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {

      // Detectar secuencia A-B-A
      if (this.checkSequence(ev)) {
        this.transitionTo(this.estado_config);
        return;
      }

      // Si solo fue A → reanuda
      if (ev === EVENTS.DEC) {
        this.transitionTo(this.estado_armed);
      }
    }
  };

  // ===============================
  // ESTADO TIMEOUT
  // ===============================

  estado_timeout = (ev) => {

    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    }

    else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };

  // ===============================
  // DETECTOR DE SECUENCIA A-B-A
  // ===============================

  checkSequence(ev) {

    this.sequence.push(ev);

    if (this.sequence.length > 3) {
      this.sequence.shift();
    }

    if (
      this.sequence.length === 3 &&
      this.sequence[0] === EVENTS.DEC &&
      this.sequence[1] === EVENTS.INC &&
      this.sequence[2] === EVENTS.DEC
    ) {
      this.sequence = []; // limpiar buffer
      return true;
    }

    return false;
  }
}

// ===============================
// SETUP
// ===============================

function setup() {

  createCanvas(windowWidth, windowHeight);

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );

  textAlign(CENTER, CENTER);

  renderer.set(
    temporizador.estado_config,
    () => drawConfig(temporizador.configValue)
  );

  renderer.set(
    temporizador.estado_armed,
    () => drawArmed(
      temporizador.remainingSeconds,
      temporizador.totalSeconds
    )
  );

  renderer.set(
    temporizador.estado_pause,
    () => drawPause(temporizador.remainingSeconds)
  );

  renderer.set(
    temporizador.estado_timeout,
    () => drawTimeout()
  );

  connectBtn = createButton("Conectar Micro:bit");
  connectBtn.position(20, 20);
  connectBtn.mousePressed(connectMicrobit);
}

// ===============================
// SERIAL MICRO:BIT (ROBUSTO)
// ===============================

async function connectMicrobit() {

  try {
    port = await navigator.serial.requestPort();
    await port.open({ baudRate: 115200 });

    connectBtn.html("Micro:bit Conectado");

    readLoop();

  } catch (err) {
    console.error("Error al conectar:", err);
  }
}

async function readLoop() {

  reader = port.readable.getReader();
  const decoder = new TextDecoder();

  while (true) {

    const { value, done } = await reader.read();

    if (done) {
      reader.releaseLock();
      break;
    }

    if (value) {

      const text = decoder.decode(value);
      const lines = text.split("\n");

      for (let line of lines) {
        const clean = line.trim();

        if (clean.length > 0) {
          console.log("Recibido:", clean);
          handleMicrobitEvent(clean);
        }
      }
    }
  }
}

function handleMicrobitEvent(event) {

  if (event === "A")
    temporizador.postEvent(EVENTS.DEC);

  if (event === "B")
    temporizador.postEvent(EVENTS.INC);

  if (event === "S")
    temporizador.postEvent(EVENTS.START);
}

// ===============================
// DRAW LOOP
// ===============================

function draw() {

  temporizador.update();

  let drawFunction = renderer.get(temporizador.currentState);

  if (drawFunction) {
    drawFunction();
  }
}

// ===============================
// RENDER
// ===============================

function drawConfig(val) {

  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);

  textSize(18);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {

  background(20);

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100);
  text(val, width / 2, height / 2);
}

function drawPause(val) {

  background(40, 40, 80);
  fill(255);
  textSize(100);
  text(val, width / 2, height / 2);

  textSize(25);
  text("PAUSA", width / 2, height / 2 + 100);
}

function drawTimeout() {

  background(255, 0, 0);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

// ===============================
// TECLADO
// ===============================

function keyPressed() {

  if (key === "a" || key === "A")
    temporizador.postEvent(EVENTS.DEC);

  if (key === "b" || key === "B")
    temporizador.postEvent(EVENTS.INC);

  if (key === "s" || key === "S")
    temporizador.postEvent(EVENTS.START);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```

```py

from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.was_pressed():
        uart.write('A')
        
    if button_b.was_pressed():
        uart.write('B')
        
    if accelerometer.was_gesture('shake'):
        uart.write('S')

```

## Bitácora de reflexión

### Codigo microbit que manda

```py

from microbit import *
import radio

radio.config(group=77)
radio.on()

while True:
    if button_a.was_pressed():
        radio.send("A")
    if button_b.was_pressed():
        radio.send("B")
    if accelerometer.was_gesture("shake"):
        radio.send("S")

    sleep(20)

```

### Codigo microbit que recibe

```py

from microbit import *
import radio

# Configurar radio
radio.config(group=77)
radio.on()

# Configuración serial
uart.init(baudrate=115200)

while True:
    # Eventos locales
    if button_a.was_pressed():
        uart.write("A")
    if button_b.was_pressed():
        uart.write("B")
    if accelerometer.was_gesture("shake"):
        uart.write("S")

    # Eventos remotos vía radio
    message = radio.receive()
    if message:
        uart.write(message + "\n")

    sleep(20)

```
Codigo p5.js

```js

// ===============================
// VARIABLES GLOBALES
// ===============================

let port;
let reader;
let connectBtn;

const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",        // Botón A
  INC: "B",        // Botón B
  START: "S",      // Shake
  TICK: "Timeout",
};

let temporizador;
let renderer = new Map();

// ===============================
// CLASE TEMPORIZADOR (FSM)
// ===============================

class Temporizador extends FSMTask {

  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;

    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.sequence = [];   // buffer para detectar A-B-A

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);

    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  // ===============================
  // ESTADO CONFIG
  // ===============================

  estado_config = (ev) => {

    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
      this.sequence = [];
    }

    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue)
        this.configValue--;
    }

    else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue)
        this.configValue++;
    }

    else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  // ===============================
  // ESTADO ARMED (corriendo)
  // ===============================

  estado_armed = (ev) => {

    if (ev === ENTRY) {
      this.sequence = [];
      this.myTimer.start();
    }

    else if (ev === EVENTS.TICK) {

      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;

        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    }

    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {

      // Detectar secuencia A-B-A
      if (this.checkSequence(ev)) {
        this.transitionTo(this.estado_config);
        return;
      }

      // Si solo fue A (sin completar secuencia) → pausa
      if (ev === EVENTS.DEC) {
        this.transitionTo(this.estado_pause);
      }
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };

  // ===============================
  // ESTADO PAUSE
  // ===============================

  estado_pause = (ev) => {

    if (ev === ENTRY) {
      this.myTimer.stop();
    }

    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {

      // Detectar secuencia A-B-A
      if (this.checkSequence(ev)) {
        this.transitionTo(this.estado_config);
        return;
      }

      // Si solo fue A → reanuda
      if (ev === EVENTS.DEC) {
        this.transitionTo(this.estado_armed);
      }
    }
  };

  // ===============================
  // ESTADO TIMEOUT
  // ===============================

  estado_timeout = (ev) => {

    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    }

    else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };

  // ===============================
  // DETECTOR DE SECUENCIA A-B-A
  // ===============================

  checkSequence(ev) {

    this.sequence.push(ev);

    if (this.sequence.length > 3) {
      this.sequence.shift();
    }

    if (
      this.sequence.length === 3 &&
      this.sequence[0] === EVENTS.DEC &&
      this.sequence[1] === EVENTS.INC &&
      this.sequence[2] === EVENTS.DEC
    ) {
      this.sequence = []; // limpiar buffer
      return true;
    }

    return false;
  }
}

// ===============================
// SETUP
// ===============================

function setup() {

  createCanvas(windowWidth, windowHeight);

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );

  textAlign(CENTER, CENTER);

  renderer.set(
    temporizador.estado_config,
    () => drawConfig(temporizador.configValue)
  );

  renderer.set(
    temporizador.estado_armed,
    () => drawArmed(
      temporizador.remainingSeconds,
      temporizador.totalSeconds
    )
  );

  renderer.set(
    temporizador.estado_pause,
    () => drawPause(temporizador.remainingSeconds)
  );

  renderer.set(
    temporizador.estado_timeout,
    () => drawTimeout()
  );

  connectBtn = createButton("Conectar Micro:bit");
  connectBtn.position(20, 20);
  connectBtn.mousePressed(connectMicrobit);
}

// ===============================
// SERIAL MICRO:BIT (ROBUSTO)
// ===============================

async function connectMicrobit() {

  try {
    port = await navigator.serial.requestPort();
    await port.open({ baudRate: 115200 });

    connectBtn.html("Micro:bit Conectado");

    readLoop();

  } catch (err) {
    console.error("Error al conectar:", err);
  }
}

async function readLoop() {

  reader = port.readable.getReader();
  const decoder = new TextDecoder();

  while (true) {

    const { value, done } = await reader.read();

    if (done) {
      reader.releaseLock();
      break;
    }

    if (value) {

      const text = decoder.decode(value);
      const lines = text.split("\n");

      for (let line of lines) {
        const clean = line.trim();

        if (clean.length > 0) {
          console.log("Recibido:", clean);
          handleMicrobitEvent(clean);
        }
      }
    }
  }
}

function handleMicrobitEvent(event) {

  if (event === "A")
    temporizador.postEvent(EVENTS.DEC);

  if (event === "B")
    temporizador.postEvent(EVENTS.INC);

  if (event === "S")
    temporizador.postEvent(EVENTS.START);
}

// ===============================
// DRAW LOOP
// ===============================

function draw() {

  temporizador.update();

  let drawFunction = renderer.get(temporizador.currentState);

  if (drawFunction) {
    drawFunction();
  }
}

// ===============================
// RENDER
// ===============================

function drawConfig(val) {

  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);

  textSize(18);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {

  background(20);

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100);
  text(val, width / 2, height / 2);
}

function drawPause(val) {

  background(40, 40, 80);
  fill(255);
  textSize(100);
  text(val, width / 2, height / 2);

  textSize(25);
  text("PAUSA", width / 2, height / 2 + 100);
}

function drawTimeout() {

  background(255, 0, 0);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

// ===============================
// TECLADO
// ===============================

function keyPressed() {

  if (key === "a" || key === "A")
    temporizador.postEvent(EVENTS.DEC);

  if (key === "b" || key === "B")
    temporizador.postEvent(EVENTS.INC);

  if (key === "s" || key === "S")
    temporizador.postEvent(EVENTS.START);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

```
