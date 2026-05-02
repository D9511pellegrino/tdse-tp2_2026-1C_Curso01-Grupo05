# Paso 09

# Análisis de Código Fuente STM32 y Evolución de Relojes

## 1. Análisis de los archivos

### `startup_stm32f103rbtx.s` (Archivo de Inicio / Ensamblador)
Este es el primer código que ejecuta el microcontrolador cuando recibe energía o es reiniciado [cite: 3]. Al estar en lenguaje ensamblador, interactúa a un nivel muy bajo con el procesador Cortex-M3 [cite: 3].
* **Tabla de Vectores (`g_pfnVectors`):** Define dónde están ubicadas las rutinas de interrupción en la memoria [cite: 3]. El primer elemento es la dirección inicial del puntero de pila (`_estack`), y el segundo es la dirección del `Reset_Handler` [cite: 3].
* **`Reset_Handler`:** Es el verdadero punto de entrada [cite: 3]. Sus tareas son:
    * Llama a `SystemInit` para configurar inicialmente el reloj base [cite: 3].
    * Copia los valores iniciales de las variables globales/estáticas desde la memoria Flash hacia la memoria RAM (sección `.data`) [cite: 3].
    * Inicializa en cero todas las variables globales/estáticas no inicializadas (sección `.bss`) [cite: 3].
    * Llama a `__libc_init_array` para inicializar constructores [cite: 3].
    * Finalmente, hace un salto (`bl main`) al archivo `main.c` [cite: 3].

### `main.c` (Programa Principal)
Aquí reside la lógica de inicialización general y el bucle infinito de la aplicación [cite: 2].
* **`HAL_Init()`:** Inicializa la librería HAL, resetea los periféricos y configura el SysTick para generar una interrupción cada 1 milisegundo [cite: 2].
* **`SystemClock_Config()`:** Configura el árbol de relojes del sistema [cite: 2]. Enciende el oscilador interno (HSI), lo divide por 2 y lo multiplica por 16 usando el PLL para alcanzar una frecuencia mayor [cite: 2].
* **`MX_GPIO_Init()` y `MX_USART2_UART_Init()`:** Inicializan los pines de entrada/salida (como el botón B1 y el LED LD2) y el puerto serie USART2 [cite: 2].
* **`app_init()` y `app_update()`:** Llamadas a funciones externas para separar la lógica de la aplicación de la configuración del hardware [cite: 2]. `app_update()` se ejecuta continuamente en el `while(1)` [cite: 2].

### `stm32f1xx_it.c` (Rutinas de Servicio de Interrupción - ISR)
Este archivo maneja los eventos asíncronos (interrupciones y excepciones) del hardware [cite: 1].
* **Manejadores del sistema:** Incluye funciones como `HardFault_Handler` y `NMI_Handler` que entran en bucles infinitos en caso de errores graves [cite: 1].
* **`SysTick_Handler()`:** Se ejecuta cada 1 milisegundo [cite: 1]. Llama a `HAL_IncTick()`, lo que incrementa un contador global usado para retardos [cite: 1].
* **`EXTI15_10_IRQHandler()`:** Es la interrupción externa asignada a los pines 10 al 15 [cite: 1]. Llama a la función de la librería HAL para manejar la interrupción del botón `B1_Pin` [cite: 1].

---

## 2. Evolución de `SystemCoreClock` y `SysTick`

A continuación, se detalla la línea de tiempo desde el reinicio hasta el bucle principal:

### Fase 1: El Reinicio (`Reset_Handler` en el archivo `.s`)
* **`SystemCoreClock`:** El microcontrolador arranca utilizando su oscilador interno por defecto (HSI) de 8 MHz [cite: 3]. Durante la llamada a `SystemInit`, la variable global `SystemCoreClock` se inicializa con el valor 8,000,000 (8 MHz) [cite: 3].
* **`SysTick`:** El periférico temporizador del sistema (SysTick) está apagado [cite: 3]. Aún no cuenta ni genera interrupciones. La variable interna de la HAL que cuenta los "ticks" (`uwTick`) vale 0.

### Fase 2: Inicio de `main.c` -> `HAL_Init()`
* **`SystemCoreClock`:** Sigue siendo 8,000,000 (8 MHz) [cite: 2].
* **`SysTick`:** La función `HAL_Init()` enciende el temporizador SysTick [cite: 2]. Lo configura basándose en el reloj actual de 8 MHz para que desborde exactamente cada 1 ms [cite: 2]. A partir de este instante, comienza a generar una interrupción cada milisegundo y el `SysTick_Handler` empieza a ejecutar `HAL_IncTick()` [cite: 1]. La variable global `uwTick` comienza a evolucionar (0, 1, 2, 3...).

### Fase 3: Configuración del Reloj -> `SystemClock_Config()`
* **`SystemCoreClock`:** Se ejecuta el código de configuración que altera la velocidad del procesador [cite: 2]:
    * Se toma el HSI (8 MHz).
    * `PLLSource` divide por 2 -> 4 MHz.
    * `PLLMUL` multiplica por 16 -> 64 MHz.
    Al finalizar esta función, la frecuencia del sistema y la variable `SystemCoreClock` pasan a valer 64,000,000 (64 MHz) [cite: 2].
* **`SysTick`:** Debido al cambio de frecuencia, el SysTick se recalibra automáticamente para los nuevos 64 MHz dentro de la configuración de reloj [cite: 2]. Sigue interrumpiendo cada 1 ms sin perder el ritmo [cite: 1]. Su variable asociada (`uwTick`) sigue incrementándose secuencialmente [cite: 1].

### Fase 4: Bucle Principal -> `while(1)`
* **`SystemCoreClock`:** Se estabiliza permanentemente en 64,000,000 Hz [cite: 2].
* **`SysTick`:** Sigue funcionando de fondo, independientemente del `while(1)` [cite: 1, 2]. El valor de la variable de conteo (`uwTick`) refleja los milisegundos transcurridos desde la inicialización [cite: 1].

**En resumen:** `SystemCoreClock` salta de 8 MHz a 64 MHz en un punto específico, mientras que el `SysTick` comienza inactivo, se activa tras `HAL_Init()` e incrementa su valor cada 1 ms durante toda la ejecución.
