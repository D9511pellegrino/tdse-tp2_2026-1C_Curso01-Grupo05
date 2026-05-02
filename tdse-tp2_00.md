# Paso 07

## Conceptos Clave para la Codificación

### Estados
Las condiciones en las que puede estar el sistema (ej. Apagado, Encendido, Espera).

### Eventos (o Entradas)
Lo que hace que el sistema cambie de estado (ej. Botón presionado, Temporizador expirado).

### Transiciones y Acciones
El cambio de un estado a otro y lo que hace el sistema durante ese cambio (ej. Prender un LED, activar un motor).

---

## Ejemplo Práctico: El Molinete del Subte

Para ilustrarlo, imaginemos el clásico ejemplo de un molinete (o torniquete) de estación de tren.

Tiene dos estados:
- Bloqueado
- Desbloqueado

Tiene dos eventos:
- Insertar Moneda
- Empujar

---

### 1) Definir los Estados y Eventos con `enum`

En C, la mejor manera de representar estados y eventos es usando tipos enumerados para que el código sea legible.

```c
#include <stdio.h>

// 1. Definimos los posibles estados
typedef enum {
    ESTADO_BLOQUEADO,
    ESTADO_DESBLOQUEADO
} EstadoMolinete;

// 2. Definimos los posibles eventos de entrada
typedef enum {
    EVENTO_MONEDA,
    EVENTO_EMPUJE
} EventoMolinete;

```

### 2) Crear la lógica de transición con `switch-case`

La forma más limpia de codificar esto es crear una función que evalúe el estado actual mediante un `switch`, y dentro de cada estado (`case`), evaluar los eventos con condicionales `if`.

```c
// Función que actualiza la máquina de estados
void procesar_evento(EstadoMolinete *estado_actual, EventoMolinete evento) {
    
    switch (*estado_actual) {
        
        case ESTADO_BLOQUEADO:
            if (evento == EVENTO_MONEDA) {
                printf("Acción: Moneda aceptada. Desbloqueando mecanismo...\n");
                *estado_actual = ESTADO_DESBLOQUEADO; // Transición de estado
            } else if (evento == EVENTO_EMPUJE) {
                printf("Acción: Alarma! Por favor inserte una moneda.\n");
                // Se mantiene en ESTADO_BLOQUEADO
            }
            break;

        case ESTADO_DESBLOQUEADO:
            if (evento == EVENTO_EMPUJE) {
                printf("Acción: Pasajero cruzó. Bloqueando mecanismo...\n");
                *estado_actual = ESTADO_BLOQUEADO; // Transición de estado
            } else if (evento == EVENTO_MONEDA) {
                printf("Acción: Moneda rechazada. Ya está desbloqueado.\n");
                // Se mantiene en ESTADO_DESBLOQUEADO
            }
            break;
            
        default:
            printf("Error: Estado desconocido.\n");
            break;
    }
}

```

### 3) El bucle principal (`main`)

Finalmente, inicializamos nuestro estado y simulamos cómo el sistema reacciona a los eventos en el tiempo.

```c
int main() {
    // El sistema inicia bloqueado
    EstadoMolinete estado = ESTADO_BLOQUEADO;

    printf("--- Iniciando Simulación del Molinete ---\n");
    
    printf("\nIntento empujar el molinete:\n");
    procesar_evento(&estado, EVENTO_EMPUJE); 

    printf("\nInserto una moneda:\n");
    procesar_evento(&estado, EVENTO_MONEDA); 

    printf("\nPaso por el molinete:\n");
    procesar_evento(&estado, EVENTO_EMPUJE); 

    return 0;
}

```
