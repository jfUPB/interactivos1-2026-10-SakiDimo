# Unidad 8

## Bitácora de proceso de aprendizaje

### Actividad 1

- Adapters utilizados:
    - Se usaron tres adapters independientes, cada uno con su propia fuente:

      - MicrobitASCIIAdapter — recibe tramas CSV por puerto serial desde el micro:bit. Hereda de BaseAdapter y sigue el mismo patrón de las unidades anteriores.
      - StrudelAdapter — actúa como servidor WebSocket en el puerto 8080. Strudel se conecta a él y envía eventos musicales con timestamp.
      - OpenStageControlAdapter — escucha paquetes OSC por UDP en el puerto 9000. Parsea el protocolo OSC manualmente sin dependencias externas.
- El contrato de mensajes de cada fuente

Microbit
```js
{ type: "microbit", x: int, y: int, btnA: bool, btnB: bool, t: ms }
```

Strudel
```js
{ type: "strudel", timestamp: ms, payload: { s: "tr909bd", delta: 0.25, cps: 0.5, cycle: 0 } }
```

Open Stage Control
```js
{ type: "osc", payload: { address: "/rgb_bd", args: [255, 50, 80] } }
```

- Pruebas técnicas realizadas

La primera prueba fue arrancar el bridge y verificar que los tres adapters iniciaban correctamente, La segunda prueba fue abrir el sketch en http://localhost:3000, presionar Connect y verificar que el servidor recibía el {cmd:"connect"} y respondía con state:"connected", la tercera prueba fue verificar cada fuente por separado: activar Strudel con .osc() y confirmar que las animaciones aparecían en el canvas, mover los controles de Open Stage Control y confirmar los logs [OSC] en la consola del navegador, e inclinar el micro:bit para verificar que el indicador cambiaba a micro:bit conectado

- Errores encontrados: 
 
  - El primer error fue que el sketch abría con file:// en lugar de http://, lo que bloqueaba las conexiones WebSocket. Se solucionó integrando un servidor HTTP directamente en bridgeServer.js usando el módulo nativo http de Node.js, eliminando la necesidad de npx serve.
  - El segundo error fue que el sketch entraba en loop de connected → Waiting for connection. El StrudelAdapter ya estaba conectado cuando el cliente llegaba, entonces el bridge mandaba state:"connected" automáticamente sin que el usuario hubiera presionado Connect. Se solucionó cambiando el mensaje inicial al cliente a state:"ready" siempre, y dejando el state:"connected" solo como respuesta explícita al {cmd:"connect"}.
  - El tercer error fue que el bridgeClient no enviaba {cmd:"connect"} al servidor porque el onopen solo llamaba _onConnect directamente. Se corrigió para que onopen enviara el comando al servidor y el _onConnect del     sketch se disparara solo cuando llegara state:"connected" como respuesta.
  - El cuarto error fue EADDRINUSE en el puerto 9000, causado por una instancia anterior del bridge corriendo en segundo plano. Se solucionó cerrando el proceso con npx kill-port 9000 antes de reiniciar.
## Bitácora de aplicación 


## Bitácora de reflexión
