# Unidad 4

## Bitácora de aplicación 

### Actividad 1

El caso de estudio presentaba un sistema en tres capas: el micro:bit enviando datos CSV por serial, un servidor Node.js que los recibe y los retransmite por WebSocket, y un sketch de p5.js que los consume para dibujar. Lo más importante que observé fue que el servidor no sabe nada del hardware — solo espera recibir { x, y, btnA, btnB } sin importar de dónde venga. Esa separación es la que permite cambiar el adapter sin tocar nada más.
El protocolo original del micro:bit era simple: 500,524,true,false\n. Cuatro valores separados por coma, terminados en salto de línea. El terminal serial permitía leerlo directamente y verificar que los datos llegaban bien antes de escribir código.

### Actividad 2

Diferencias respecto al protocolo original: tiene un carácter de inicio $, usa pares clave-valor separados por |, incluye un timestamp T y un checksum CHK para verificar integridad. El checksum se calcula como |X| + |Y| + A + B, entonces para el ejemplo: |-245| + |12| + 1 + 0 = 258.
MicrobitV2Adapter.js
El adapter hereda de BaseAdapter y sigue el mismo patrón que MicrobitASCIIAdapter: acumula texto en un string buffer, busca \n como delimitador, y cuando encuentra una línea completa la pasa a parseFrame().
La función parseFrame() hace el trabajo de validación: verifica que la trama empiece con $, separa los pares clave-valor, y calcula el checksum. Si no coincide, lanza un ParseError que el adapter captura y registra como warning en consola sin actualizar el estado. El objeto que emite al resto del sistema es idéntico al adapter original: { x, y, btnA, btnB }, lo que garantiza que bridgeServer.js no necesita ningún cambio.
Registro en bridgeServer.js
Solo se agregaron dos líneas: el import del nuevo adapter y el caso "microbit" en createAdapter() que instancia MicrobitV2Adapter.
sketch.js
El sketch de p5.js original usaba controles de mouse y teclado.

## Bitácora de reflexión
