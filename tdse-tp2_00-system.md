# Análisis del Sistema de Control y Actuadores

Este documento detalla el funcionamiento del sistema de control de eventos y la comunicación con los actuadores, basándose en los archivos proporcionados (`task_system_attribute.h`, `task_actuator_attribute.h`, `task_system.c`, `task_system_interface.c` y `task_actuator_interface.c`).

## 1. Análisis Funcional General

El sistema implementa una arquitectura dirigida por eventos para gestionar la lógica de control principal (`task_system`) y su interacción con los periféricos de salida (`task_actuator`).

* **`task_system_attribute.h`**: Define la estructura de datos `task_system_dta_t` y los estados/eventos del sistema.
* **`task_system.c`**: Implementa la lógica de la Máquina de Estados (FSM). Actúa como el consumidor de eventos de una cola.
* **`task_system_interface.c`**: Gestiona una **cola circular (FIFO)** que desacopla la generación de eventos (hecha por sensores u otras tareas) del procesamiento.
* **`task_actuator_interface.c`**: Provee una interfaz para que el sistema "excite" a los actuadores, modificando directamente su estructura de datos.

---

## 2. Evolución de Variables en `task_system`

Estas variables pertenecen a la estructura `task_system_dta_list[index]`.

| Variable | Unidad | Inicio (`task_system_init`) | Evolución en `task_system_update` |
| :--- | :---: | :--- | :--- |
| **`index`** | - | Se inicializa para el modo `NORMAL`. | Recorre el arreglo de estados del sistema (usualmente 1 solo en este contexto). |
| **`.tick`** | **Ticks** | `0` | Se incrementa o resetea dependiendo de la lógica de tiempo dentro de los estados (por ejemplo, para duraciones de activación). |
| **`.state`** | - | `ST_SYS_IDLE` | Cambia a `ST_SYS_ACTIVE` al recibir un evento válido. |
| **`.event`** | - | `EV_SYS_IDLE` | Toma el valor del evento extraído de la cola (`EV_SYS_ACTIVE`). |
| **`.flag`** | - | `false` | Se pone en `true` cuando `any_event_task_system()` detecta algo en la cola. Se limpia al procesar el evento. |

**Unidad de medida de `.tick`:** Corresponde a los **ciclos de ejecución del loop principal** (Application Ticks), definidos por la frecuencia del planificador.

---

## 3. Comportamiento de `task_system_statechart(uint32_t index)`

Esta función procesa la lógica de decisión del sistema.

1. **Captura de Eventos:** Primero verifica si hay eventos pendientes en la interfaz de la cola. Si existen, activa el `.flag` y carga el evento en la estructura de datos local.
2. **Estado `ST_SYS_IDLE`:**
    * Si el `.flag` es verdadero y el evento es `EV_SYS_ACTIVE`:
        * Limpia el flag.
        * **Comunica al Actuador:** Llama a `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`.
        * Transiciona a `ST_SYS_ACTIVE`.
3. **Estado `ST_SYS_ACTIVE`:** Permanece en este estado realizando la acción correspondiente hasta que una condición de fin o un nuevo evento lo devuelva a `IDLE`.

---

## 4. Evolución de la Cola `event_task_system_queue`

Implementada en `task_system_interface.c`.

| Variable | Inicio (`task_system_init`) | Ejecución (`put`) | Ejecución (`get`) |
| :--- | :---: | :--- | :--- |
| **`i`** | Usado para limpiar la cola. | - | - |
| **`.head`** | `0` | Aumenta en `1`. Reinicia a `0` si llega a `QUEUE_LENGTH`. | No cambia. |
| **`.tail`** | `0` | No cambia. | Aumenta en `1`. Reinicia a `0` si llega a `QUEUE_LENGTH`. |
| **`.count`** | `0` | Incrementa hasta `QUEUE_LENGTH`. | Decrementa hasta `0`. |
| **`.queue[i]`** | `EMPTY` | Se escribe el evento en la posición `head`. | Se lee el evento de la posición `tail`. |

---

## 5. Evolución de Variables de `task_actuator`

Estas variables son modificadas por `task_system` a través de la interfaz del actuador.

| Variable | Inicio (`task_system_init`) | Evolución en `task_system_update` |
| :--- | :---: | :--- |
| **`identifier`** | `ID_LED_A` | Es el índice que identifica qué periférico se desea controlar. |
| **`.event`** | `EV_LED_IDLE` | Cambia a `EV_LED_ACTIVE` cuando el sistema ejecuta `put_event_task_actuator`. |
| **`.flag`** | `false` | Se pone en `true` inmediatamente cuando el sistema envía un comando al actuador. |

### Resumen del Flujo
1. Un evento externo llega a la cola circular.
2. `task_system_update` detecta el evento (`flag = true`).
3. La FSM en `task_system_statechart` consume el evento y decide activar el LED.
4. Se llama a la interfaz del actuador, que marca el flag del LED para que la tarea del actuador sepa que debe encender el hardware.
