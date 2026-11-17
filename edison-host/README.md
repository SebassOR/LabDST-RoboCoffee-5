# RoboCoffee en Intel Edison

Este host sustituye la página embebida en `rasphost.py`: el Intel Edison
sirve el panel web y se comunica con el ESP32 usando el protocolo TCP de
OAK9 (`IP:8080`). Los clientes y la Raspberry Pi Pico W acceden al Edison
por HTTP. La Pico W puede publicar su telemetría en
`POST /api/pico/telemetry`; además de la telemetría del MPU, el firmware
publica el estado del interruptor de carga y recibe los comandos de los servos
en la respuesta del Edison.

## Requisitos

- Node.js 0.10.28 o posterior en el Edison (el servidor usa APIs nativas y
  no instala dependencias externas).
- Edison, ESP32 y dispositivos que abrirán el panel en la misma red.
- El ESP32 accesible desde el Edison en el puerto TCP 8080.

El servidor no requiere paquetes npm externos.

## Ejecutar

En el Edison, desde esta carpeta:

```sh
ROBOT_IP=192.168.100.9 ROBOT_PORT=8080 PORT=3000 node server.js
```

`PORT` configura el puerto web del Edison y `ROBOT_PORT` el puerto del ESP32.
En el ejemplo, el menú de compra queda en `http://<IP-del-Edison>:3000/`
y `http://<IP-del-Edison>:3000/cliente`. El panel del trabajador queda en
`http://<IP-del-Edison>:3000/trabajador`.
En Windows PowerShell para pruebas:

```powershell
$env:ROBOT_IP = "192.168.100.9"
$env:PORT = "3000"
node server.js
```
local --
$env:HOST = "127.0.0.1"
$env:PORT = "3000"
node server.js

Abre la dirección correspondiente desde el teléfono o computadora.
El puerto web predeterminado es 80; si el proceso no puede enlazarlo, usa un
puerto alto como 3000. Si la dirección del ESP32 cambia, establece `ROBOT_IP`
al iniciar el servidor.

## Panel y flujo de pedidos

- El menú público solo muestra bebidas, la recomendación, la temperatura y el
  formulario para ordenar; no muestra telemetría ni controles del robot.
- La pantalla inicial del trabajador prioriza los pedidos y después muestra
  el Dashboard. Los pedidos se actualizan automáticamente; la orientación del IMU
  se consulta solo al abrir DEBUG MPU, para evitar sondeos continuos al robot.
  Batería y posición se leen al abrir el panel o al pulsar actualizar.
- La recomendación es un modelo de reglas local, no un servicio de IA ni un
  pronóstico meteorológico: sugiere Americano a 28 °C o más, Capuccino a 17 °C
  o menos y Latte entre ambas temperaturas. El comprador puede ajustar la
  temperatura de referencia en el menú. Si no se indica, se usa
  `AMBIENT_TEMP_C` (22 °C por defecto); no hay un sensor de clima conectado.
- El menú ofrece Espresso, Latte, Capuccino y Americano. Solo el Americano se
  puede pedir frío; no se muestran precios porque aún no están configurados.
- Al crear un pedido, aparece como pendiente en el panel del trabajador. Al
  pulsar «Enviar pedido», queda en cola si el robot está ocupado; cuando le toca,
  abre los servos para cargar el café. El interruptor confirma la taza, el host
  cierra los servos y comienza el recorrido; en la mesa los abre para entregar.
  Al confirmar la recepción, los cierra y regresa al origen. Los pedidos
  siguientes esperan a que termine todo ese ciclo.
- El interruptor de carga usa GP6 como común (salida a nivel bajo), GP4 como
  NO y GP5 como NC; el firmware habilita pull-up en GP4 y GP5 y considera
  presionado el interruptor cuando NO está bajo y NC alto.
- El estado del robot y los controles manuales están en una pantalla separada,
  oculta por defecto. El botón «Developer» de la barra lateral cambia a esa
  pantalla; es un modo de interfaz, no un mecanismo de autenticación.
- La primera entrega fija la referencia de origen. Inicia con el robot quieto
  en su punto de partida y despeja la ruta antes de enviar una orden.
- El control manual limita PWM a ±70, requiere mantener presionado, mantiene
  una conexión TCP mientras se maneja y tiene un watchdog de 700 ms en el host,
  además de un botón de parada.
- El botón «DEBUG MPU» abre una gráfica en vivo de roll, pitch y yaw; mientras
  está abierta, el panel solicita lecturas del IMU cada 250 ms y muestra también
  los valores crudos más recientes. La gráfica conserva hasta 30 segundos.
- El estado y pedidos se guardan en memoria; al reiniciar el proceso se
  reinician.
- Las páginas están separadas por rol, pero la API no tiene cuentas ni
  autenticación. Usa una red local de confianza; no publiques el host en
  Internet sin añadir control de acceso.

## API para la Pico W

La Pico W puede enviar JSON por HTTP al Edison:

```http
POST /api/pico/telemetry
Content-Type: application/json

{"status":"lista","temperatura":24}
```

La telemetría más reciente aparece en el modo Developer del panel y en
`GET /api/status`. También están disponibles `GET /api/menu?temperature=24`,
`GET /api/health`, `GET /api/robot/imu` (lectura cruda y orientación actual del
IMU), `GET /api/orders`, `GET /api/customer/orders/:id`,
`POST /api/orders`, `POST /api/orders/:id/deliver`,
`POST /api/orders/:id/received`, `POST /api/robot/motor` y
`POST /api/robot/stop`.

## Inicio automático

El repositorio incluye `robocoffee-edison.service`, una unidad de `systemd`
para ejecutar el servidor en cada arranque del Edison. Una vez copiada la
carpeta a `/home/root/edison-host`, instálala y actívala como root:

```sh
cp /home/root/edison-host/robocoffee-edison.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable robocoffee-edison.service
systemctl start robocoffee-edison.service
systemctl status robocoffee-edison.service
```

`enable` configura el arranque automático; `start` lo inicia ahora.
Para ver el registro usa `journalctl -u robocoffee-edison.service -f`.
Si cambian la IP del ESP32 o la carpeta de instalación, actualiza las
variables `ROBOT_IP` y `ExecStart` en la unidad y repite `daemon-reload`.
Reserva la IP del Edison en el router para que la dirección del menú no cambie.
