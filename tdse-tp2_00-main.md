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

### `main.c` (Programa Principal)
Aquí reside la lógica de inicialización general y el bucle infinito de la aplicación.
* **`HAL_Init()`:** Inicializa la librería HAL, resetea los periféricos y configura el SysTick para generar una interrupción cada 1 milisegundo.
* **`SystemClock_Config()`:** Configura el árbol de relojes del sistema. Enciende el oscilador interno (HSI), lo divide por 2 y lo multiplica por 16 usando el PLL para alcanzar una frecuencia mayor.
* **`MX_GPIO_Init()` y `MX_USART2_UART_Init()`:** Inicializan los pines de entrada/salida (como el botón B1 y el LED LD2) y el puerto serie USART2.
* **`app_init()` y `app_update()`:** Llamadas a funciones externas para separar la lógica de la aplicación de la configuración del hardware. `app_update()` se ejecuta continuamente en el `while(1)`.

### `stm32f1xx_it.c` (Rutinas de Servicio de Interrupción - ISR)
Este archivo maneja los eventos asíncronos (interrupciones y excepciones) del hardware.
* **Manejadores del sistema:** Incluye funciones como `HardFault_Handler` y `NMI_Handler` que entran en bucles infinitos en caso de errores graves.
* **`SysTick_Handler()`:** Se ejecuta cada 1 milisegundo. Llama a `HAL_IncTick()`, lo que incrementa un contador global usado para retardos.
* **`EXTI15_10_IRQHandler()`:** Es la interrupción externa asignada a los pines 10 al 15. Llama a la función de la librería HAL para manejar la interrupción del botón `B1_Pin`.

---

## 2. Evolución de `SystemCoreClock` y `SysTick`

A continuación, se detalla la línea de tiempo desde el reinicio hasta el bucle principal:

### Fase 1: El Reinicio (`Reset_Handler` en el archivo `.s`)
* **`SystemCoreClock`:** El microcontrolador arranca utilizando su oscilador interno por defecto (HSI) de 8 MHz. Durante la llamada a `SystemInit`, la variable global `SystemCoreClock` se inicializa con el valor 8,000,000 (8 MHz).
* **`SysTick`:** El periférico temporizador del sistema (SysTick) está apagado. Aún no cuenta ni genera interrupciones. La variable interna de la HAL que cuenta los "ticks" (`uwTick`) vale 0.

### Fase 2: Inicio de `main.c` -> `HAL_Init()`
* **`SystemCoreClock`:** Sigue siendo 8,000,000 (8 MHz).
* **`SysTick`:** La función `HAL_Init()` enciende el temporizador SysTick. Lo configura basándose en el reloj actual de 8 MHz para que desborde exactamente cada 1 ms. A partir de este instante, comienza a generar una interrupción cada milisegundo y el `SysTick_Handler` empieza a ejecutar `HAL_IncTick()`. La variable global `uwTick` comienza a evolucionar (0, 1, 2, 3...).

### Fase 3: Configuración del Reloj -> `SystemClock_Config()`
* **`SystemCoreClock`:** Se ejecuta el código de configuración que altera la velocidad del procesador:
    * Se toma el HSI (8 MHz).
    * `PLLSource` divide por 2 -> 4 MHz.
    * `PLLMUL` multiplica por 16 -> 64 MHz.
    Al finalizar esta función, la frecuencia del sistema y la variable `SystemCoreClock` pasan a valer 64,000,000 (64 MHz).
* **`SysTick`:** Debido al cambio de frecuencia, el SysTick se recalibra automáticamente para los nuevos 64 MHz dentro de la configuración de reloj. Sigue interrumpiendo cada 1 ms sin perder el ritmo. Su variable asociada (`uwTick`) sigue incrementándose secuencialmente.

### Fase 4: Bucle Principal -> `while(1)`
* **`SystemCoreClock`:** Se estabiliza permanentemente en 64,000,000 Hz.
* **`SysTick`:** Sigue funcionando de fondo, independientemente del `while(1)`. El valor de la variable de conteo (`uwTick`) refleja los milisegundos transcurridos desde la inicialización.

**En resumen:** `SystemCoreClock` salta de 8 MHz a 64 MHz en un punto específico, mientras que el `SysTick` comienza inactivo, se activa tras `HAL_Init()` e incrementa su valor cada 1 ms durante toda la ejecución.
