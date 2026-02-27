# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 04

``` py
from microbit import *
import utime
import music

# -------------------------------------------------
# IMÁGENES DE LLENADO
# -------------------------------------------------
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# -------------------------------------------------
# TIMER
# -------------------------------------------------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# -------------------------------------------------
# MÁQUINA DE ESTADOS
# -------------------------------------------------
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        # Timer de 1 segundo
        self.myTimer = self.createTimer("Timeout", 1000)

        self.valor = 20  # valor inicial
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # Actualizar timers
        for t in self.timers:
            t.update()

        # Procesar eventos
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # ---------------------------------------------
    # ESTADO CONFIGURACIÓN
    # ---------------------------------------------
    def estado_configuracion(self, ev):

        if ev == "ENTRY":
            self.valor = 20
            display.show(FILL[self.valor])

        elif ev == "A":
            if self.valor < 25:
                self.valor += 1
                display.show(FILL[self.valor])

        elif ev == "B":
            if self.valor > 15:
                self.valor -= 1
                display.show(FILL[self.valor])

        elif ev == "S":
            self.transicion_a(self.estado_armado)

    # ---------------------------------------------
    # ESTADO ARMADO (CUENTA REGRESIVA)
    # ---------------------------------------------
    def estado_armado(self, ev):

        if ev == "ENTRY":
            self.myTimer.start(1000)

        elif ev == "Timeout":
            if self.valor > 0:
                self.valor -= 1
                display.show(FILL[self.valor])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_explosion)

        elif ev == "EXIT":
            self.myTimer.stop()

    # ---------------------------------------------
    # ESTADO EXPLOSIÓN
    # ---------------------------------------------
    def estado_explosion(self, ev):

        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.DADADADUM)

        elif ev == "A":
            self.transicion_a(self.estado_configuracion)

# -------------------------------------------------
# CICLO PRINCIPAL
# -------------------------------------------------
task = Task()

while True:

    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)

```
codigo PlantML

```

@startuml

<style>
root {
  BackgroundColor #4a556822
  FontColor #718096
  LineColor #718096
  Margin 30
  Padding 10
}

stateDiagram {
  state {
    BackgroundColor #edf2f7
    LineColor #4a5568
    FontColor #2d3748
    RoundCorner 10
  }
  arrow {
    LineColor #718096
    FontColor #718096
  }
}
</style>

title Micro:bit Countdown - UML State Machine

[*] --> Config : Task() (constructor)

Config : entry /\n  count = 20\n  display.show(FILL[count]) \n A / if count>15 then count-=1; display.show(FILL[count]) \n B / if count<25 then count+=1; display.show(FILL[count])
Config --> Armado : S /

Armado : entry /\n  myTimer.start(1000) \nTimeout / \n if count>0 then count-=1; display.show(FILL[count]); myTimer.start(1000) \nexit / \n self.myTimer.stop()


Armado --> Explosion : Timeout [count==0] /

Explosion : entry /\n  display.show(Image.SKULL)\n  music.play(music.DADADADUM)
Explosion --> Config : A /

@enduml

```

<img width="515" height="569" alt="image" src="https://github.com/user-attachments/assets/8989371f-8f1c-420c-81b7-6c3606796e42" />


## Bitácora de aplicación 



## Bitácora de reflexión

### Actividad 5

``` py
from microbit import *
import utime
import music

# -----------------------------
# IMÁGENES
# -----------------------------
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# -----------------------------
# TIMER
# -----------------------------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# -----------------------------
# TASK (MÁQUINA DE ESTADOS)
# -----------------------------
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.myTimer = self.createTimer("Timeout", 1000)

        self.valor = 20
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()

        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # -----------------------------
    # CONFIGURACIÓN
    # -----------------------------
    def estado_configuracion(self, ev):
        if ev == "ENTRY":
            self.valor = 20
            display.show(FILL[self.valor])

        elif ev == "A":
            if self.valor < 25:
                self.valor += 1
                display.show(FILL[self.valor])

        elif ev == "B":
            if self.valor > 15:
                self.valor -= 1
                display.show(FILL[self.valor])

        elif ev == "S":
            self.transicion_a(self.estado_armado)

    # -----------------------------
    # ARMADO
    # -----------------------------
    def estado_armado(self, ev):
        if ev == "ENTRY":
            self.myTimer.start(1000)

        elif ev == "Timeout":
            if self.valor > 0:
                self.valor -= 1
                display.show(FILL[self.valor])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_explosion)

        elif ev == "EXIT":
            self.myTimer.stop()

    # -----------------------------
    # EXPLOSIÓN
    # -----------------------------
    def estado_explosion(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.DADADADUM)

        elif ev == "A":
            self.transicion_a(self.estado_configuracion)

# -----------------------------
# LOOP PRINCIPAL
# -----------------------------
task = Task()

while True:

    # BOTONES FÍSICOS
    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    # MENSAJES DESDE p5.js
    if uart.any():
        mensaje = uart.read().decode().strip()
        if mensaje == "A":
            task.post_event("A")
        elif mensaje == "B":
            task.post_event("B")
        elif mensaje == "S":
            task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```
Codigo de p5.js

``` js
let port;
let writer;

async function connectSerial() {
  port = await navigator.serial.requestPort();
  await port.open({ baudRate: 115200 });

  const encoder = new TextEncoder();
  writer = port.writable.getWriter();
}

async function writeSerial(data) {
  const encoder = new TextEncoder();
  await writer.write(encoder.encode(data));
}

function setup() {
  createCanvas(400, 400);

  let btn = createButton("Conectar micro:bit");
  btn.position(20, 20);
  btn.mousePressed(connectSerial);
}

function draw() {
  background(220);
  textSize(18);
  text("Presiona A, B o S", 100, 200);
}

function keyPressed() {
  if (key === 'A') writeSerial("A\n");
  if (key === 'B') writeSerial("B\n");
  if (key === 'S') writeSerial("S\n");
}
```


