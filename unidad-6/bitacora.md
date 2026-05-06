# Unidad 6

## Bitácora de proceso de aprendizaje

### Parte 1

- ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?
  - R= Recibir un mensaje es solo guardarlo, ejecutarlo es hacer algo con él en el momento correcto. 
- ¿Por qué un sistema audiovisual puede necesitar timestamp además de los datos del evento?
  - R= Porque si se dispara la animación apenas llega el mensaje, depende de la latencia de red y el resultado se desincroniza del audio. Con el timestamp se puede esperar el momento exacto.
- ¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?
  - R= El bridge, el bridgeClient, la FSM y el patrón adapter. Solo cambia el adapter y el sketch.
 
### Parte 2

- Si Strudel fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?
  - R= Mensajes JSON por WebSocket con dirección OSC /dirt/play y una lista plana de pares clave-valor en args.
- ¿Qué variables mínimas necesitarías extraer para poder construir una visualización útil?
  - R= El nombre del sonido s, el delta para saber cuánto dura, y el timestamp para saber cuándo dispararlo.

### Parte 3

- ¿Qué problema resuelve la cola de eventos?
  - R= Desacoplar la recepción del mensaje de su ejecución. Los eventos llegan antes de tiempo y la cola los guarda hasta que toca dispararlos.
- ¿Por qué esta capa no pertenece al bridge sino al lado que interpreta el evento?
  - R= Porque el bridge solo transporta, no sabe nada de tiempo musical ni de animaciones. Esa lógica le pertenece al frontend.

### Parte 4

- ¿Qué papel cumple el Adapter en U4 y U5?
  - R= Traducir el protocolo del hardware a un objeto normalizado { x, y, btnA, btnB } sin que el resto del sistema sepa cómo llegaron los datos.
- ¿Qué Adapter necesitas ahora para que los eventos de Strudel no entren “crudos” al sistema visual?
  - R= Uno que reciba los mensajes de Strudel, extraiga s, delta, cps y timestamp del array de args, y los entregue en un formato limpio al bridge.

## Bitácora de aplicación 

- Cómo configuraste Strudel para emitir eventos;
  - R= En Strudel usé el siguiente patrón con .osc() para que enviara eventos por WebSocket al puerto 8080:

  ```js
  setcps(0.5)
  const pat = s("[bd*2 sd hh oh]").bank("tr909")
  $: stack(pat.gain('0.5'), pat.osc())
  ```
- Qué estructura final de mensaje decidiste usar;
  - R= El StrudelAdapter normaliza cada evento así:

  ```js
    {
      type: "strudel",
      timestamp: 1777061252744,
      payload: { s: "tr909bd", delta: 0.25, cps: 0.5, cycle: 0 }
    }
  ```
- Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning;
  - R= El StrudelAdapter actúa como servidor WebSocket en el puerto 8080 y espera que Strudel se conecte. Cuando llega un mensaje lo normaliza y lo pasa al bridgeServer, que lo reenvía por WebSocket al sketch. El bridgeClient lo recibe y lo manda a la FSM como evento STRUDEL. La FSM lo encola en updateStrudel() y processStrudelQueue() lo dispara cuando Date.now() alcanza el timestamp.
- Cómo separaste recepción, cola temporal y renderizado;
  - R= Cada capa tiene una responsabilidad única. El StrudelAdapter recibe el mensaje crudo de Strudel y lo normaliza, el bridgeClient lo transporta al sketch como un evento STRUDEL sin interpretarlo. La FSM lo recibe en updateStrudel() y lo mete a la cola ordenada por timestamp, finalmente drawRunning() llama cada frame a processStrudelQueue(), que revisa si Date.now() ya alcanzó el timestamp de algún evento y solo entonces crea la animación, el render simplemente lee las animaciones activas y las dibuja. En ningún momento una capa hace el trabajo de otra.
- Qué pruebas hiciste para verificar la sincronización;
  - R= La primera prueba fue arrancar el bridge con --verbose y verificar que los eventos llegaban con timestamps coherentes y en orden creciente, la segunda fue abrir el sketch y confirmar visualmente que las animaciones aparecían al ritmo del patrón de Strudel, la tercera fue observar la consola del navegador para confirmar que los logs de updateStrudel aparecían antes que las animaciones correspondientes, lo que confirmó que la cola estaba funcionando como buffer temporal y no disparando los eventos inmediatamente al recibirlos.
- Qué problemas encontraste y cómo los solucionaste.
  - R= El primer problema fue que el bridge intentaba conectarse a Strudel como cliente, cuando en realidad debía ser al revés: el adapter abre un servidor y Strudel se conecta a él. El segundo problema fue que cuando Strudel se desconectaba momentáneamente, el adapter llamaba onDisconnected y el sketch entraba en loop de reconexión. Se solucionó eliminando la llamada a onDisconnected cuando se cierra un cliente individual, dejándola solo cuando se cierra el servidor completo.
   
## Bitácora de reflexión

- Realiza un diagrama detallado del flujo de datos de tu sistema. Debe incluir al menos:
- Compara las unidades 4, 5 y 6 en una tabla. Compara al menos:

  | Aspecto | Unidad 4 | Unidad 5 | Unidad 6 |
  |---|---|---|---|
  | **Fuente de datos** | micro:bit físico | micro:bit físico | Strudel (app web) |
  | **Formato del mensaje** | CSV por serial: `500,524,true,false\n` | Binario: 8 bytes con header `0xAA` y checksum | JSON por WebSocket: array plano de args OSC |
  | **Problema técnico principal** | Parsear y validar texto CSV | Framing binario y verificación de checksum | Sincronización temporal con timestamp |
  | **Mecanismo de validación** | Validación de rango y tipo en el parser | Checksum byte 7 = suma de bytes 1–6 mod 256 | Cola ordenada por timestamp, disparo por `Date.now()` |
  | **Lugar de traducción** | `MicrobitASCIIAdapter._onChunk()` | `MicrobitBinaryAdapter._onChunk()` | `StrudelAdapter._normalize()` |
  | **Papel del tiempo** | Sin rol — datos llegan y se usan inmediatamente | Sin rol — paquetes se procesan al llegar | Central — los eventos se encolan y se disparan en su timestamp exacto |


- Explica por qué esta unidad sigue perteneciendo a la misma arquitectura del curso, aunque la fuente de datos ya no sea hardware físico.
  - R= Aunque la fuente de datos ya no es hardware físico, el problema que resuelve la arquitectura es el mismo: existe algo externo que produce datos en un formato propio, y ese formato no puede entrar crudo al frontend. El adapter sigue siendo el único componente que conoce el protocolo de la fuente. El bridge sigue siendo el canal de transporte sin lógica de dominio. La FSM sigue siendo la capa que organiza el estado. El render sigue siendo la capa que solo dibuja.
Lo que cambió fue la naturaleza del dato: en lugar de bytes de acelerómetro, ahora son eventos musicales con timestamp. Pero el flujo de responsabilidades es idéntico — y eso es precisamente lo que demuestra que la arquitectura es sólida.
- Explica qué decisiones tomaste para traducir eventos musicales en visualidad. Justifica por qué tu mapeo visual tiene sentido.
  - R= El mapeo se basó en asociar las propiedades físicas del sonido con propiedades visuales equivalentes. El bombo (bd) es el sonido más grave y de mayor impacto, entonces se traduce en un círculo que crece desde el centro, la caja (sd) es un sonido horizontal y cortante, entonces se traduce en una barra que atraviesa el ancho de la pantalla y desaparece, los hats (hh, oh) son sonidos pequeños y repartidos rítmicamente, entonces se traducen en cuadrados pequeños que aparecen en posiciones aleatorias del canvas.
- Si tuvieras que integrar una tercera aplicación en el futuro, ¿Qué partes de tu arquitectura actual conservarías y cuáles cambiarías?
  - R= Conservaría todo: BaseAdapter, bridgeServer, bridgeClient, la FSM y el sistema de cola temporal. Solo añadiría un nuevo adapter que normalice el protocolo de la nueva fuente, registraría el caso en bridgeServer, y añadiría un nuevo tipo de evento en el bridgeClient y en la FSM.

