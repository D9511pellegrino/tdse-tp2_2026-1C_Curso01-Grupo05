# Análisis del Subsistema de Actuadores

Este documento detalla el funcionamiento del subsistema de actuadores (periféricos de salida), implementado mediante una Máquina de Estados Finitos (FSM) no bloqueante y temporizada.

## 1. Análisis de los Archivos del Código Fuente

* **`task_actuator_attribute.h`**: Define la estructura y el vocabulario del actuador. Contiene las enumeraciones para eventos (`EV_LED_IDLE`, `EV_LED_ACTIVE`), estados (`ST_LED_IDLE`, `ST_LED_ACTIVE`) e identificadores de hardware (`ID_LED_A`). Define las estructuras `task_actuator_cfg_t` (configuración constante) y `task_actuator_dta_t` (datos dinámicos).
* **`task_actuator.c`**: Implementa la lógica principal. Incluye la inicialización (`task_actuator_init`) que apaga los periféricos y resetea variables, y la máquina de estados (`task_actuator_statechart`) que se ejecuta dentro del ciclo de actualización (`task_actuator_update`).
* **`task_actuator_interface.c`**: Actúa como el "buzón de entrada" o API. Permite que otros módulos (como el sistema principal) soliciten acciones al actuador de forma desacoplada mediante la función `put_event_task_actuator`.

---

## 2. Evolución de Variables en `task_actuator.c`

Estas variables pertenecen a la estructura `task_actuator_dta_list[index]` y se gestionan durante la ejecución de la tarea.

| Variable | Unidad | Inicio (`task_actuator_init`) | Evolución en `task_actuator_update` |
| :--- | :---: | :--- | :--- |
| **`index`** | - | Recorre de `0` a `ACTUATOR_DTA_QTY - 1`. | Itera secuencialmente sobre todos los actuadores configurados en cada ciclo del planificador. |
| **`.tick`** | **Ticks** | `0` | En reposo es `0`. En `ST_LED_ACTIVE`, aumenta en `1` por cada ciclo hasta alcanzar `tick_max`. |
| **`.state`** | - | `ST_LED_IDLE` | Cambia a `ST_LED_ACTIVE` al recibir un evento. Regresa a `IDLE` al expirar el tiempo. |
| **`.event`** | - | `EV_LED_IDLE` | Se actualiza asíncronamente mediante la interfaz (ej. a `EV_LED_ACTIVE`). |
| **`.flag`** | - | `false` | Se pone en `true` por la interfaz. Vuelve a `false` cuando el *statechart* consume el evento. |

**Unidad de medida de `.tick`:** Corresponde a los **ciclos de ejecución del loop principal** (Application Ticks). El tiempo físico real depende de la frecuencia del planificador.

---

## 3. Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función implementa la lógica de un temporizador no bloqueante para el hardware:

1.  **Estado `ST_LED_IDLE`**:
    * Mantiene el LED apagado.
    * **Transición**: Si `flag == true` y `event == EV_LED_ACTIVE`:
        * Limpia el `flag`.
        * Enciende el LED (`HAL_GPIO_WritePin` a `led_on`).
        * Cambia al estado `ST_LED_ACTIVE`.
2.  **Estado `ST_LED_ACTIVE`**:
    * El LED permanece encendido.
    * Incrementa el contador `.tick` en cada llamada.
    * **Transición**: Si `.tick >= tick_max`:
        * Apaga el LED (`HAL_GPIO_WritePin` a `led_off`).
        * Reinicia `.tick = 0`.
        * Regresa al estado `ST_LED_IDLE`.

---

## 4. Evolución de Variables en la Interfaz

Variables involucradas al llamar a `put_event_task_actuator(event, identifier)` desde otros módulos.

| Variable | Estado Inicial | Evolución tras Ejecución |
| :--- | :--- | :--- |
| **`identifier`** | Índice del actuador (ej. `ID_LED_A`). | Se usa para indexar el arreglo de datos. |
| **`task_actuator_dta_list[identifier].event`** | `EV_LED_IDLE` | Se sobrescribe con el nuevo evento (ej. `EV_LED_ACTIVE`). |
| **`task_actuator_dta_list[identifier].flag`** | `false` | Se fuerza a `true` inmediatamente para notificar a la tarea. |

### Resumen del Ciclo de Vida
1.  **Petición**: Un módulo externo llama a la interfaz. El `flag` se activa.
2.  **Detección**: En la siguiente ejecución de `task_actuator_update`, el *statechart* ve el `flag`.
3.  **Acción**: El LED se enciende y comienza la cuenta de *ticks*.
4.  **Finalización**: Al cumplirse el tiempo, el LED se apaga y el sistema queda listo para un nuevo evento.
