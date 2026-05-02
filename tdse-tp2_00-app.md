# Análisis de Código Fuente: Planificador Bare-Metal (ETS)

Este documento detalla el análisis funcional del código fuente proporcionado, que implementa una arquitectura "Bare-Metal" mediante un planificador disparado por eventos de tiempo (Event-Triggered System o ETS).

## 1. Análisis de Archivos del Proyecto

### app.c
Es el núcleo de la aplicación y actúa como el **planificador (scheduler)**.
- **Responsabilidad:** Define la lista de tareas (`task_sensor`, `task_system`, `task_actuator`) y gestiona sus ciclos de vida.
- **Métricas:** Utiliza la estructura `task_dta_t` para almacenar estadísticas de ejecución de cada tarea.
- **Funciones clave:**
    - `app_init()`: Inicializa el hardware de conteo de ciclos, las tareas y las interrupciones.
    - `app_update()`: Es el bucle principal que verifica si ha ocurrido un "tick" de tiempo para ejecutar las tareas de forma secuencial.

### app_it.c
Gestiona las interrupciones del microcontrolador.
- **Funciones clave:**
    - `HAL_SYSTICK_Callback()`: Se ejecuta automáticamente cada vez que el temporizador del sistema (SysTick) se desborda. Su única tarea es incrementar el contador global `g_app_tick_cnt`.

### logger.c / logger.h
Implementan un sistema de registro de mensajes para depuración.
- **Características:** Soporta *semihosting* para enviar mensajes a la consola del depurador.
- **Seguridad:** Las macros como `LOGGER_INFO` deshabilitan las interrupciones (`CPSID i`) para garantizar que el mensaje se procese sin interrupciones, habilitándolas nuevamente al finalizar (`CPSIE i`).

### dwt.h
Provee acceso a la unidad **DWT (Data Watchpoint and Trace)** del procesador ARM Cortex.
- **Uso:** Permite medir ciclos de reloj de forma precisa para calcular tiempos de ejecución en microsegundos basándose en la frecuencia del sistema (`SystemCoreClock`).

### systick.c
Proporciona utilidades para retardos bloqueantes.
- **Funciones clave:**
    - `systick_delay_us()`: Implementa un delay preciso en microsegundos monitoreando directamente los registros del temporizador SysTick.

---

## 2. Evolución de Variables de Control y Métricas

A continuación se detalla el comportamiento de las variables clave durante la ejecución:

| Variable | Unidad | Inicialización (`app_init`) | Evolución en `app_update` |
| :--- | :---: | :--- | :--- |
| **`g_app_tick_cnt`** | Ticks | `0` | Aumenta en `1` por interrupción de hardware. Disminuye en `1` cuando el planificador atiende el tick. |
| **`g_app_runtime_us`** | $\mu s$ | No inicializada (global) | Se resetea a `0` en cada tick. Acumula la suma de los tiempos `LET` de todas las tareas del ciclo. |
| **`index`** | - | Recorre `0` a `TASK_QTY-1` | Iterador del ciclo `for` que recorre secuencialmente las tareas (sensor, system, actuator). |
| **`task_dta_list[index].NOE`** | - | `0` | Se incrementa en `1` tras cada ejecución exitosa de la tarea correspondiente. |
| **`task_dta_list[index].LET`** | $\mu s$ | `0` | Almacena el tiempo exacto que tomó la última ejecución de la tarea. |
| **`task_dta_list[index].BCET`** | $\mu s$ | `1000` | Se actualiza si el `LET` actual es menor al valor guardado (Mejor tiempo histórico). |
| **`task_dta_list[index].WCET`** | $\mu s$ | `0` | Se actualiza si el `LET` actual es mayor al valor guardado (Peor tiempo histórico). |

---

## 3. Impacto del Uso de `LOGGER_INFO()`

El uso de funciones de log dentro del ciclo de tareas (`task_update`) tiene consecuencias críticas en el rendimiento del sistema:

1. **Incremento del WCET:** El tiempo de ejecución de la última tarea (`LET`) aumentará significativamente debido al procesamiento de cadenas de texto y la latencia del *semihosting*. Esto provocará que el **`task_dta_list[index].WCET`** se actualice con un valor muy alto, distorsionando la métrica de rendimiento real de la lógica de la tarea.
2. **Impacto en `g_app_runtime_us`:** Al ser una suma de los tiempos de todas las tareas, el tiempo total de ejecución del ciclo se verá inflado por el tiempo que consume el logger.
3. **Bloqueo de Interrupciones:** Dado que `LOGGER_INFO` deshabilita las interrupciones para ser *thread-safe*, si el proceso de log es muy lento, el sistema podría perderse eventos de hardware o ticks del SysTick, afectando el determinismo del sistema en tiempo real.
