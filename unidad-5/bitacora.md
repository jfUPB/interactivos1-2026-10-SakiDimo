# Unidad 5
## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 2

Durante la realización de la actividad, ocurrieron varios errores, todos, principalmente relacionados directamente con la conexión y detección del microbit.

El primer error que encontre era que el adapter en binario originalmente se llamaba microbit-bin, el cual por alguna razon no estaba conectando, por lo cual le cambie el nombre a microbitbinary y con esto solucione el primer error, como segundo error, el microbit no se conectaba por lo que yo consideru un error del puerto, ya que al cambiarlo se soluciono.

Errores especificos que salieron nuevos con las actualizaciones del programa no experimente.

## Bitácora de reflexión

### Actividad 3

### Pregunta 1 — Tabla comparativa ASCII vs Binario
 
| Pregunta | `MicrobitASCIIAdapter` | `MicrobitBinaryAdapter` |
|---|---|---|
| **Tamaño de paquete** | Variable, típicamente 15–25 bytes | Fijo, siempre 8 bytes |
| **Mecanismo de framing** | Delimitador `\n` al final de cada línea | Byte de sincronización `0xAA` al inicio de cada trama |
| **Verificación de integridad** | Ninguna — si llega `500,524,tru` no hay forma de detectarlo más allá de validación de rango | Checksum: suma de bytes 1–6 módulo 256, comparado con byte 7 |
| **Complejidad del parser** | `split(",")` y conversión de strings | Manejo de `Buffer`, `readInt16BE`, lógica de re-sincronización por byte |
| **Facilidad de depuración** | Se puede leer directamente en cualquier terminal serial (PuTTY, Arduino IDE, etc.) | Los bytes son ilegibles en un terminal de texto; se necesita un analizador hexadecimal |

### Pregunta 2

La arquitectura separa responsabilidades en capas donde cada una tiene un único trabajo.

El Adapter es el único componente que sabe cómo habla el hardware. Todo lo que ocurre antes de emitir (x, y, btnA, btnB) es un detalle interno suyo. El resto del sistema nunca ve bytes ni strings CSV, solo ese objeto 
normalizado.

Esto significa que cambiar el protocolo del hardware es equivalente a cambiar un adapter. Las capas superiores no necesitan modificarse porque su contrato de entrada nunca cambió, solo cambió la implementación interna de quien lo produce.

### Pregunta 3

Depende principalmente de dos factores: quién va a leer los datos y qué tan limitados son los recursos.

Binario tiene sentido cuando el sistema está en producción y los datos los consume una máquina directamente. Es más compacto, más eficiente y permite verificar integridad con checksum o CRC. Un sensor médico, un dispositivo Bluetooth con ancho de banda limitado, o un protocolo industrial como Modbus son ejemplos donde cada byte cuenta y nadie va a abrir un terminal a leer los datos manualmente.

ASCII tiene sentido cuando los humanos necesitan intervenir, ya sea depurando, auditando logs, o integrando sistemas heterogéneos. Un archivo de log en CSV, una API REST en JSON, o el monitor serial del Arduino IDE durante desarrollo son ejemplos donde la legibilidad vale más que la eficiencia.
