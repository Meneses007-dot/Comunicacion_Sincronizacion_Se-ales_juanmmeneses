# Taller de comunicación, sincronización y señales

Hecho por: Juan Manuel Meneses Daza

Implementación en C de un tablero compartido mediante un servidor TCP
concurrente. Cada cliente se atiende en un hilo y puede leer o escribir las
64 casillas del tablero.

## Requisitos

- Linux
- GCC
- `make`
- POSIX threads

## Compilación

```bash
cd src
make
```

Para eliminar los ejecutables:

```bash
make clean
```

## Ejecución

Primero se inicia el servidor, indicando la política de sincronización:

```bash
./servidor_tablero sin
./servidor_tablero mutex
./servidor_tablero rwlock
```

Las opciones significan:

- `sin`: sin protección del tablero, útil para observar condiciones de carrera.
- `mutex`: acceso exclusivo para lectores y escritores.
- `rwlock`: varios lectores simultáneos y escritores exclusivos.

En otra terminal se ejecuta un cliente:

```bash
./cliente_tablero 127.0.0.1 lector x 20
./cliente_tablero 127.0.0.1 escritor A 10
```

El servidor escucha en el puerto `5500`. Se detiene de forma ordenada con
`Ctrl-C`, mostrando el tablero y los contadores de operaciones.

## Prueba de carga

Con el servidor en ejecución:

```bash
./carga.sh [direccion_ip] [numero_de_lectores] [lecturas_por_lector]
```

Ejemplo:

```bash
./carga.sh 127.0.0.1 8 20
```

El script inicia varios lectores y dos escritores, y reporta las lecturas
incoherentes detectadas.

## Protocolo básico

El servidor acepta una orden por línea:

```text
LEER
ESCRIBIR A
SALIR
```

Las respuestas exitosas comienzan con `OK` e incluyen la versión del tablero.

## Archivos principales

- `src/servidor_tablero.c`: servidor TCP, hilos, señales y sincronización.
- `src/cliente_tablero.c`: cliente para lecturas y escrituras.
- `src/carga.sh`: prueba concurrente de carga.
- `bitacora.md`: resultados y análisis del taller.
# Comunicacion_Sincronizacion_Senales_juanmmeneses
