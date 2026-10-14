# Bitácora — Taller: Señales en un servidor concurrente

- **Estudiante:** Juan Manuel Meneses Daza
- **Usuario:** juanmmeneses
- **Correo:** juanmmeneses@unicauca.edu.co

## Punto 1 — Comportamiento del servidor sin modificaciones ante Ctrl-C

Se ejecutó el servidor original (sin el manejo de señales) con `./servidor_original rwlock`,
se conectó un cliente lector con `./cliente_tablero 127.0.0.1 lector x 500` y, con el
cliente conectado, se envió `SIGINT` (equivalente a Ctrl-C).

**Salida del servidor al terminar:**

```
Tablero de 64 casillas, politica rwlock, escuchando en el puerto 5500
```

La única línea es la de arranque. El servidor muere de forma inmediata y silenciosa
(terminado por la señal, sin acción de limpieza): **no cierra el socket de escucha, no
presenta contadores y no produce despedida alguna**. El proceso deja de existir en el
momento exacto en que llega la señal, en medio de lo que estuviera haciendo.

**Salida del cliente que estaba conectado:**

```
La conexion se interrumpio en la repeticion 70
69 lecturas, 0 incoherentes, 1.000 segundos
```

El cliente quedó incomunicado: al morir el servidor, su socket se cierra y el cliente
detecta el fin de la conexión a mitad de su serie de lecturas (repetición 70 de 500).
Con esto se evidencia la carencia que el taller resuelve: un servidor que se pierde sin
cerrar el puerto y sin informar cuántas operaciones atendió.

## Punto 4 — Protección de los contadores globales

Los contadores `total_lecturas` y `total_escrituras` son memoria compartida entre todos
los hilos que atienden a los clientes. Cada hilo los incrementa al despachar una orden
`LEER` o `ESCRIBIR`, de modo que un incremento sin protección es una suma no atómica en la
que un hilo puede perderse (`read-modify-write` interrumpido).

**Mecanismo elegido:** un mutex dedicado exclusivamente a los contadores,
`candado_contadores`, inicializado de forma estática:

```c
static long total_lecturas = 0;
static long total_escrituras = 0;
static pthread_mutex_t candado_contadores = PTHREAD_MUTEX_INITIALIZER;
```

**Criterio:** los contadores se protegen **con un candado propio y separado** de los
candados del tablero (`candado` / `candado_rw`). De esta manera:

1. El incremento `lock → ++ → unlock` queda blindado contra condiciones de carrera.
2. No se mezcla la política de acceso al tablero con la contabilidad: un lector con
   `rwlock` no bloquea la suma de su propio contador, y viceversa, con lo que se evita
   contendencia innecesaria entre mecanismos con propósitos distintos.

El mismo candado protege el **informe final**, en desarrollo del cierre ordenado.

## Punto 6 — Envío de señales con `kill`

Se desplegó el servidor modificado con `./servidor_tablero rwlock &` y, contra su PID,
se enviaron `SIGALRM`, `SIGINT` y `SIGKILL`.

```console
$ ./servidor_tablero rwlock &
$ kill -ALRM $!   # informe periódico inmediato
$ kill -INT $!    # cierre ordenado
```

**Salida del servidor:**

```
Tablero de 64 casillas, politica rwlock, escuchando en el puerto 5500
[ALARMA] Version: 20 | Tablero: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
[ALARMA] Version: 20 | Tablero: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB

=== Servidor finalizado de forma ordenada ===
Total lecturas atendidas: 30
Total escrituras atendidas: 20
```

Efecto de cada señal:

- **`SIGALRM`:** el manejador `a_la_alarma` solo activa la bandera `alarma_activada`; el
  ciclo principal la observa, lee el tablero aplicando la política vigente (aquí
  `rwlock` en modo lectura) e imprime la versión y el contenido. La primera línea
  corresponde a la alarma periódica automática (5 s) y la segunda a la disparada con
  `kill -ALRM`.
- **`SIGINT`:** con `sa_flags = 0` la llamada bloqueante `accept` regresa con `EINTR`,
  el hilo principal sale del ciclo al observar la bandera `terminar`, cierra el socket de
  escucha y presenta los contadores. El proceso termina con código 0.
- **`SIGKILL`:** prueba con otra instancia:

```
Tablero de 64 casillas, politica sin, escuchando en el puerto 5500
(código de salida del servidor: -9)
```

**¿Por qué `SIGKILL` no se puede capturar?** `SIGKILL` (9) y `SIGSTOP` (19) no se pueden
atrapar, bloquear ni ignorar porque su terminación está reservada al núcleo: el sistema
operativo los elimina de la tabla de disposiciones del proceso y los aplica de forma
incondicional, sin pasar por ningún manejador de usuario. Esa negativa beneficia al
sistema porque garantiza que **siempre** existe una manera de terminar (u detener) un
proceso, aunque su código esté dañado, colgado o bloqueado en un ciclo infinito: el
núcleo conserva siempre la autoridad de recuperar los recursos de cualquier proceso,
incluso de uno que haya hecho un uso incorrecto de señales o candados.

## Punto 7 — Verificación de retorno y errores con `perror`

Todas las llamadas agregadas por el taller verifican su retorno y, ante el fallo,
informan el error con `perror`:

- `sigaction(SIGINT, ...)` y `sigaction(SIGALRM, ...)` → `perror("sigaction ...")`.
- `pthread_create(...)` → `perror("pthread_create")` y se libera el descriptor.

Para provocar un fallo a propósito se arrancó un **segundo servidor mientras el primero
seguía escuchando** en el puerto 5500. El mensaje producido, con el segundo servidor
abandonando con código 1:

```
bind: Address already in use
```

El error se origina en `crear_socket_escucha`, que devuelve `-1`, y el servidor sale de
`main` con `EXIT_FAILURE`, sin intentar continuar. Obsérvese que el mensaje se genera
gracias a `perror`, no a un mensaje inventado por el programa: es exactamente el motivo
reportado por la llamada `bind` (la inclusión de `errno.h` y la comprobación
explícita del retorno de cada llamada al sistema).

## Punto 8 — Efecto de `signal(SIGPIPE, SIG_IGN)`

Cuando se escribe en un socket cuyo extremo remoto ya se cerró, el sistema genera la
señal `SIGPIPE`. Su acción por omisión es **terminar el proceso**. Como el servidor es un
proceso multihilo único, si un cliente cierra la conexión de golpe y un hilo de atención
intenta responderle, ese `SIGPIPE` no mataría al hilo sino al **proceso completo**, es
decir, al servidor de los demás clientes.

La línea `signal(SIGPIPE, SIG_IGN)` en `main` cambia esa disposición para que el servidor
**ignore** la señal: entonces `write` al socket roto no termina el proceso, simplemente
devuelve `-1`. El hilo ve el retorno negativo, cierra el socket local y finaliza su
trabajo:

```c
if (write(socket_cliente, respuesta, strlen(respuesta)) == -1) {
    break;   /* el cliente se fue: se cierra y el hilo termina */
}
close(socket_cliente);
return NULL;
```

Sin esa línea, el primer cliente que cerrara la conexión de forma brusca derribaría el
servidor entero con una terminación por `SIGPIPE`, sin cierre ordenado, sin contadores y
dejando el puerto sin servicio para los demás clientes. Con ella, la caída brusca de un
cliente queda contenida en su propio hilo.

## Salida del informe periódico

El informe periódico se produce cada 5 segundos. Cada vez que el ciclo principal observa
`alarma_activada`, lee el tablero con la política vigente en modo lectura, imprime versión
y contenido, y vuelve a programar la alarma con `alarm(5)` (rearme en cada iteración, no
una sola vez al inicio). En la corrida de prueba, con escritores A y B activos, el tablero
llegó íntegro a la versión 20:

```
[ALARMA] Version: 20 | Tablero: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
```

La lectura se realizó con la política `rwlock` en modo lectura (`leer_tablero`), idéntica
a la que emplea la orden `LEER` de un cliente, por lo que el informe nunca espera ni
interfiere con los candados del tablero.

Y que el Padre del Entendimiento nos guíe a todos.