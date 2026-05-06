# Unidad 7

## Bitácora de proceso de aprendizaje

### Parte 1

- ¿Qué diferencia hay entre un evento musical y un mensaje de control?
  - R= Un evento musical ocurre en un instante específico y tiene timestamp, un mensaje de control no tiene tiempo asociado, simplemente cambia un parámetro que permanece activo hasta que algo lo modifique de nuevo.
- ¿Qué quiere decir que un parámetro del sistema sea persistente?
  - R= Que no desaparece después de procesarse. Si cambio el color del bombo a rojo, todos los bombos siguientes siguen siendo rojos hasta que llegue otro mensaje OSC que lo cambie.
- ¿Qué partes del sistema de la unidad 6 permanecen intactas en este nuevo caso?
  - R= El StrudelAdapter, el bridgeServer, el bridgeClient, la FSM y toda la lógica de cola temporal. Solo se añaden cosas nuevas sin romper lo que ya funciona.

### Parte 2

- Si Open Stage Control fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?
  - R= Mensajes OSC por UDP. Cada mensaje tiene una dirección como /rgb_bd y una lista de argumentos con los valores del control.
- ¿Qué parte de ese protocolo te interesa conservar y cuál te gustaría normalizar?
  - R= La dirección OSC la conservo porque identifica el parámetro. Los argumentos los normalizo a un array simple de números para que el resto del sistema no sepa nada del formato UDP crudo.

### Parte 3

- ¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?
  - R= Porque Strudel produce eventos efímeros que ocurren en un instante y se descartan. OSC produce parámetros que deben mantenerse activos. Si lo metiera en la cola temporal se ejecutaría una sola vez y el sistema quedaría desactualizado.
- ¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?
  - R= Los colores por familia de sonido, la escala de las animaciones y el toggle del fondo. Todos viven en oscColor, oscScale y oscBackgroundActive dentro de la FSM.

### Parte 4

- ¿Qué componentes de la arquitectura necesitas conservar obligatoriamente?
  - R= El StrudelAdapter, el bridgeServer, el bridgeClient, la FSM y la cola de eventos. Todo eso se mantiene intacto.
- ¿Qué nuevas estructuras de estado necesitas introducir para soportar control paramétrico?
  - R= El objeto de parámetros persistentes en la FSM — variables que el render lee en cada frame y que solo se actualizan cuando llega un mensaje OSC.

## Bitácora de aplicación 

- Cómo configuraste Open Stage Control;
  - R= Open Stage Control se configuró con el send target apuntando a 127.0.0.1:9000, que es el puerto UDP donde escucha el OpenStageControlAdapter. Se crearon tres widgets en la interfaz.
- Qué widgets decidiste usar y por qué;
  - R= Un widget RGB para el color del bombo, porque es la animación más dominante visualmente y cambiar su color tiene impacto inmediato. Un fader para la escala global de las animaciones, porque permite ajustar la intensidad visual sin cambiar la lógica de los eventos. Un toggle para activar o desactivar el trail de fondo, porque cambia radicalmente la estética del sistema entre acumulación y limpieza
- Qué estructura final de mensaje decidiste usar para los controles;
  - R=
```js
      { type: "osc", payload: { address: "/rgb_bd", args: [255, 120, 30] }
```
  La dirección identifica el parámetro y los args llevan los valores. Es la misma estructura para todos los controles, lo que hace el sistema predecible y fácil de extender.
- Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning;
  - R= El OpenStageControlAdapter escucha UDP en el puerto 9000, parsea los paquetes OSC manualmente sin dependencias externas, y emite onData con el mensaje normalizado. El bridgeServer hace broadcast del mensaje con type:"osc". El bridgeClient lo detecta y llama _onOsc. El sketch lo recibe como evento OSC y la FSM llama updateOsc(), que actualiza las variables persistentes. El drawRunning las lee en cada frame.
- Cómo integraste ambas fuentes de datos en el mismo frontend;
  - R= Los dos flujos convergen en la capa de estado de la FSM pero nunca se mezclan. Strudel alimenta eventQueue y genera activeAnimations. OSC actualiza oscColor, oscScale y oscBackgroundActive. El render usa ambas fuentes simultáneamente: dibuja las animaciones activas con los colores y escalas que vienen del estado OSC.
- Qué pruebas hiciste para verificar que el control paramétrico funciona sin romper la sincronización de Strudel;
  - R= Primero verifiqué con --verbose que los mensajes OSC llegaban al servidor con la dirección y args correctos. Luego moví cada control mientras Strudel corría y confirmé en la consola del navegador que los logs [OSC] aparecían correctamente. Finalmente verifiqué visualmente que los cambios de color, escala y trail se reflejaban en el canvas sin interrumpir el ritmo de Strudel.
- Qué problemas encontraste y cómo los solucionaste.
  - R= El primer problema fue que el puerto UDP del adapter no coincidía con el send target de Open Stage Control. Se solucionó alineando ambos al puerto 9000. El segundo fue que la librería osc no estaba disponible en el entorno, así que se reescribió el parser OSC manualmente usando el Buffer de Node.js, igual que el parser binario del micro:bit en la unidad 5.

## Bitácora de reflexión

- Compara en una tabla las fuentes de datos que has trabajado en las unidades 4, 5, 6 y 7. Compara al menos:


- Explica por qué Open Stage Control no debe tratarse igual que Strudel dentro de la arquitectura.
  - R= Strudel produce eventos que ocurren en un instante y se consumen. Open Stage Control produce parámetros que describen cómo debe comportarse el sistema a partir de ahora. Si un mensaje OSC entrara a la cola temporal, se dispararía una sola vez en su momento de llegada y luego el sistema perdería el valor. El color volvería al default en el siguiente frame. La diferencia no es de protocolo sino de semántica: uno produce eventos, el otro produce estado.
- Justifica los tres controles que elegiste:
  - R= El control RGB /rgb_bd modifica el color del bombo porque es la animación visualmente más dominante y cambiar su color es inmediatamente perceptible. Permite al performer usar el color como dimensión expresiva adicional sincronizada con el ritmo.
    
  - El fader /scale modifica la escala global de las animaciones porque permite ajustar la intensidad visual del sistema en tiempo real sin cambiar la lógica de los eventos. A escala baja las animaciones son sutiles, a escala alta son explosivas. Es el equivalente visual de un fader de volumen.

  - El toggle /background_toggle activa o desactiva el trail oscuro de fondo, lo que cambia radicalmente la estética entre dos modos: acumulación, donde las formas se superponen y persisten, y limpieza, donde cada frame borra el anterior y las animaciones son discretas. Es una decisión composicional de alto nivel que afecta toda la pieza.

- Si tuvieras que integrar una tercera fuente de control en el futuro, ¿Qué partes de tu arquitectura actual conservarías y cuáles extenderías?
  - R= Conservaría todo: BaseAdapter, bridgeServer, bridgeClient, la FSM, la cola temporal y el objeto de estado persistente. Solo añadiría un nuevo adapter que normalice el protocolo de la nueva fuente, registraría el caso en bridgeServer, añadiría un nuevo tipo de evento en bridgeClient y en la FSM, y definiría cómo esa fuente actualiza el estado. Si produce eventos efímeros va a la cola, si produce parámetros va al estado persistente. La arquitectura ya tiene espacio para ambos sin modificar nada existente.
