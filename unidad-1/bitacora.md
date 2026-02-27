# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 1

#### -¿Qué es un sistema físico interactivo?

R= Es un sistema que responde a estimulos humanos o del entorno y los traduce en experiencias interactivas unicas

#### -¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

R= Podria crearse un sistema en el que al detectarse ciertos movimentos se ejecuten ciertas animaciones para que interactuen las persones con el sistema

### Atividad 2

-¿Qué es el diseño/arte generativo?

-¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

### Actividad 3

Se realizo el experimento con el microbit indicado en la pagina.

<img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/9d56bd7c-45b3-40c8-b726-510d1b954ce4" />

### Actividad 4

### Actividad 5

#### Codigo de p5.js:

Usando como base el codigo de la actividad 3, cree el siguiente codigo:

```js
let port;
let connectBtn;
let x;

function setup() 
{
  createCanvas(400, 400);
  background(220);
  port = createSerial();
  connectBtn = createButton('Connect to micro:bit');
  connectBtn.position(130, 300);
  connectBtn.mousePressed(connectBtnClick);
  x = width / 2;
}

function draw() 
{
  
    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            x +=10;
        }
        else if(dataRx == 'B')
        {
            x-=10;
        }

        background(220);
    }

    fill ('white');
    ellipse (x, height / 2, 100, 100);

    if (!port.opened()) 
    {
        connectBtn.html('Connect to micro:bit');
    }
  
    else 
    {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() 
{
    if (!port.opened()) 
    {
        port.open('MicroPython', 115200);
    } 
    
    else 
    {
        port.close();
    }
}

```
#### Codigo Microbit:

Usando directamente el codigo que usamos en la actividad 3 se llego al siguiente codigo:

```py

from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)

```

#### Explica detalladamente cómo funciona el sistema físico interactivo que has creado.

Principalmente y como mencione antes, utilice como base el codigo de la actividad 3, dejando las funciones y cambiando los condicionales de "if" que leian la información que llegaba del microbit y cambiando la funcionalidad a modificar la posición en x con una variable definida al inicio del codigo, de esta manera cando recibe la señal de A o B, se mueve a la izquierda o a la derecha respectivamente.

<img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/4d273f03-21eb-43f1-8d60-5125d8ef7b4b" />

## Bitácora de aplicación 



## Bitácora de reflexión



