# Análisis de Tarea de Sensores e Interfaz de Sistema

Este documento analiza el funcionamiento del sistema de detección de eventos y comunicación asíncrona implementado en los archivos `task_sensor.c`, `task_system_interface.c`, `task_system_attribute.h` y `task_sensor_attribute.h`.

## 1. Análisis de los Archivos y Funcionamiento General

El sistema implementa un modelo de **Máquina de Estados Finitos (FSM)** para la lectura de sensores (típicamente pulsadores) con lógica de **antirrebote (debouncing)**, y utiliza una **cola circular** para comunicar eventos detectados a la lógica del sistema.

* **`task_sensor.c`**: Contiene la lógica activa de los sensores. Implementa la inicialización (`task_sensor_init`) y el ciclo de actualización (`task_sensor_update`). Su función principal es filtrar el ruido de las entradas digitales y, una vez confirmada una acción, notificarla.
* **`task_system_interface.c`**: Actúa como un *buffer* o puente de comunicación. Implementa una cola tipo FIFO (First-In, First-Out) que permite que la tarea de sensores "deposite" eventos y la tarea de sistema los "extraiga" de forma asíncrona.
* **`task_system_attribute.h`**: Define los tipos de datos (estructuras, enumeraciones de estados y eventos) que utiliza el sistema para procesar la lógica de control.
* **`task_sensor_attribute.h`** (inferido de `task_sensor.c`): Define la estructura `task_sensor_dta_t` que mantiene el contexto (estado actual, contador de ticks, evento) de cada sensor individual.

---

## 2. Evolución de Variables en la Tarea de Sensores

Estas variables residen en el arreglo `task_sensor_dta_list[]`.

| Variable | Unidad | Inicio (`task_sensor_init`) | Evolución en `task_sensor_update` |
| :--- | :---: | :--- | :--- |
| **`index`** | - | Recorre de `0` a `SENSOR_QTY - 1`. | Itera secuencialmente sobre todos los sensores configurados en cada ciclo del planificador. |
| **`.tick`** | **Ticks** | Se inicializa en `0`. | Se incrementa en los estados transitorios (`FALLING` y `RISING`). Se resetea a `0` al cambiar de estado o si la condición de entrada desaparece. |
| **`.state`** | - | `ST_BTN_UP` (reposo). | Cambia según la lógica: `UP` -> `FALLING` -> `DOWN` -> `RISING` -> `UP`. |
| **`.event`** | - | `EV_BTN_UP` o `DOWN`. | Se actualiza en cada ciclo leyendo directamente el hardware (`HAL_GPIO_ReadPin`). |

**Nota sobre `.tick`:** La unidad de medida es el **tick del sistema**, cuyo valor en tiempo (ms) depende de la frecuencia con la que el planificador (scheduler) ejecute la función `task_sensor_update`.

---

## 3. Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es el corazón del filtrado de señales. Su objetivo es evitar que los rebotes mecánicos de un pulsador generen múltiples eventos falsos.

**Lógica de funcionamiento:**
1.  **Lectura Inmediata:** Actualiza el `.event` leyendo el pin físico.
2.  **Estados Estables (`ST_BTN_UP` / `ST_BTN_DOWN`)**: Si detecta un cambio (ej. de UP a pulsado), no cambia inmediatamente al estado de "confirmado", sino que pasa a un estado transitorio.
3.  **Estados Transitorios (`ST_BTN_FALLING` / `ST_BTN_RISING`)**: 
    * Incrementa el contador `.tick`.
    * Si el contador alcanza el valor `p_task_sensor_cfg->tick_max` (tiempo de antirrebote), se confirma el cambio de estado.
    * **Acción Crucial:** Al confirmar la transición a `ST_BTN_DOWN`, ejecuta `put_event_task_system(EV_SYS_ACTIVE)`, enviando la señal a la cola del sistema.
4.  **Log de Depuración:** Utiliza `LOGGER_INFO` para imprimir en consola cada vez que el sensor cambia de estado, facilitando el seguimiento en tiempo real.

---

## 4. Evolución de la Cola de Eventos (`event_task_system_queue`)

La cola se encuentra en `task_system_interface.c` y gestiona cómo los eventos del sensor llegan al sistema.

| Variable | Inicio (`task_sensor_init`) | Evolución en Ejecución (`put_event_task_system`) |
| :--- | :---: | :--- |
| **`.head`** | `0` | Se incrementa en `1` cada vez que el sensor confirma una pulsación. Vuelve a `0` al llegar a `QUEUE_LENGTH`. |
| **`.tail`** | `0` | Permanece fija hasta que la tarea de sistema lee un evento. Se incrementa al "extraer" datos. |
| **`.count`** | `0` | Se incrementa con cada `put` y se decrementa con cada `get`. Indica cuántos eventos hay pendientes de procesar. |
| **`.queue[i]`** | Todos en `EMPTY` | Almacena el valor `EV_SYS_ACTIVE` en la posición indicada por `.head` antes de incrementarlo. |

**Resumen del flujo:**
Cuando presionas el botón, el **Statechart** espera unos ticks para estar seguro; una vez seguro, llama a la **Interface** para poner un evento en la **Queue**; finalmente, los índices de la cola (`head` y `count`) aumentan para que el sistema sepa que tiene trabajo pendiente.
