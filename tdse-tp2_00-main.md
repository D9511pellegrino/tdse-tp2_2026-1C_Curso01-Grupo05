# Paso 09

# Análisis de Código Fuente STM32 y Evolución de Relojes

## 1. Análisis de los archivos

### `startup_stm32f103rbtx.s` (Archivo de Inicio / Ensamblador)
Este es el primer código que ejecuta el microcontrolador cuando recibe energía o es reiniciado. Al estar en lenguaje ensamblador, interactúa a un nivel muy bajo con el procesador Cortex-M3.

* **Tabla de Vectores (`g_pfnVectors`):** Define dónde están ubicadas las rutinas de interrupción en la memoria. El primer elemento es la dirección inicial del puntero de pila (`_estack`), y el segundo es la dirección del `Reset_Handler`.
* **`Reset_Handler`:** Es el verdadero punto de entrada. Sus tareas son:
    * Llama a `SystemInit` para configurar inicialmente el reloj base.
    * Copia los valores iniciales de las variables globales/estáticas desde la memoria Flash hacia la memoria RAM (sección `.data`).
    * Inicializa en cero todas las variables globales/estáticas no inicializadas (sección `.bss`).
    * Llama a `__libc_init_array` para inicializar constructores.
    * Finalmente, hace un salto (`bl main`) al archivo `main.c`.

---

### `main.c` (Programa Principal)
Aquí reside la lógica de inicialización general y el bucle infinito de la aplicación.

* **`HAL_Init()`:** Inicializa la librería HAL, resetea los periféricos y configura el SysTick para generar una interrupción cada 1 milisegundo.
* **`SystemClock_Config()`:** Configura el árbol de relojes del sistema. Enciende el oscilador interno (HSI), lo divide por 2 y lo multiplica por 16 usando el PLL para alcanzar una frecuencia mayor.
* **`MX_GPIO_Init()` y `MX_USART2_UART_Init()`:** Inicializan los pines de entrada/salida (como el botón B1 y el LED LD2) y el puerto serie USART2.
* **`app_init()` y `app_update()`:** Separan la lógica de la aplicación de la configuración del hardware. `app_update()` se ejecuta continuamente en el `while(1)`.

---

### `stm32f1xx_it.c` (Rutinas de Servicio de Interrupción - ISR)
Este archivo maneja los eventos asíncronos del hardware.

* **Manejadores del sistema:** Incluye funciones como `HardFault_Handler` y `NMI_Handler` que entran en bucles infinitos en caso de errores graves.
* **`SysTick_Handler()`:** Se ejecuta cada 1 milisegundo. Llama a `HAL_IncTick()`, lo que incrementa un contador global usado para retardos.
* **`EXTI15_10_IRQHandler()`:** Interrupción externa para los pines 10 al 15. Maneja la interrupción del botón `B1_Pin`.

---

## 2. Evolución de `SystemCoreClock` y `SysTick`

### Fase 1: El Reinicio (`Reset_Handler`)
* **`SystemCoreClock`:** Inicialmente 8 MHz (HSI).
* **`SysTick`:** Apagado. `uwTick = 0`.

---

### Fase 2: `HAL_Init()`
* **`SystemCoreClock`:** 8 MHz.
* **`SysTick`:** Se activa y comienza a generar interrupciones cada 1 ms. `uwTick` empieza a incrementarse.

---

### Fase 3: `SystemClock_Config()`
* **`SystemCoreClock`:**
    * HSI = 8 MHz  
    * /2 → 4 MHz  
    * ×16 → 64 MHz  

→ Resultado: **64 MHz**

* **`SysTick`:** Se recalibra automáticamente para mantener interrupciones cada 1 ms.

---

### Fase 4: `while(1)`
* **`SystemCoreClock`:** Estable en 64 MHz.
* **`SysTick`:** Sigue corriendo en segundo plano. `uwTick` cuenta milisegundos.

---

**En resumen:** `SystemCoreClock` salta de 8 MHz a 64 MHz en un punto específico, mientras que el `SysTick` comienza inactivo, se activa tras `HAL_Init()` e incrementa su valor cada 1 ms durante toda la ejecución.
