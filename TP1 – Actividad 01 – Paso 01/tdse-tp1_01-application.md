# Paso 6

Este es un proyecto clásico de sistemas embebidos utilizando un microcontrolador STM32 (específicamente la línea STM32F103) junto con el sistema operativo en tiempo real FreeRTOS y las librerías HAL (Hardware Abstraction Layer) de ST.

A continuación, presento el análisis detallado de cada uno de los puntos solicitados.

### 1. Análisis y funcionamiento de los archivos

* **`startup_stm32f103rbtx.s`**: Es el archivo de inicio (escrito en lenguaje ensamblador) que se ejecuta apenas el microcontrolador recibe energía o un reinicio. Su función es establecer el entorno base para que el código C pueda ejecutarse. Configura el puntero de pila (Stack Pointer), llama a la función `SystemInit` (para una configuración muy básica del reloj), inicializa la memoria RAM (copiando los valores iniciales de las variables globales/estáticas a la sección `.data` y llenando con ceros la sección `.bss`), y finalmente invoca la función `main()`. También contiene la tabla de vectores de interrupción.
* **`main.c`**: Es el núcleo de la aplicación de usuario. Aquí se inicializa la HAL (`HAL_Init`), se configura la señal de reloj principal a su frecuencia de trabajo (`SystemClock_Config`) y se inicializan los periféricos (GPIO, UART, TIM2). Además, arranca el Timer 2, inicializa la lógica de la aplicación (`app_init()`), crea una tarea por defecto de FreeRTOS (`defaultTask`) y, finalmente, cede el control del procesador a FreeRTOS mediante `osKernelStart()`.
* **`stm32f1xx_it.c`**: Contiene las Rutinas de Servicio de Interrupción (ISR). Sirve como intermediario; cuando ocurre una interrupción por hardware (por ejemplo, el desbordamiento de un Timer o una interrupción externa EXTI), el procesador salta a estas funciones. En este código, las rutinas como `TIM2_IRQHandler`, `TIM4_IRQHandler` y `EXTI15_10_IRQHandler` simplemente delegan el trabajo a los manejadores de la HAL correspondientes (`HAL_TIM_IRQHandler` y `HAL_GPIO_EXTI_IRQHandler`).
* **`FreeRTOSConfig.h`**: Es el archivo de configuración principal de FreeRTOS. Define cómo se comportará el RTOS: la frecuencia del "tick" del sistema (1000 Hz), el tamaño de la memoria dinámica (Heap), el uso de Mutexes, si soporta localización de memoria estática y dinámica, etc. También es vital porque "mapea" las interrupciones del hardware ARM Cortex-M (`SVC_Handler`, `PendSV_Handler`, `SysTick_Handler`) hacia el kernel del RTOS, y activa la recolección de estadísticas de tiempo de ejecución (`configGENERATE_RUN_TIME_STATS = 1`).
* **`freertos.c`**: Provee código de soporte específico para integrar FreeRTOS. Dado que en `FreeRTOSConfig.h` se habilitó la asignación estática (`configSUPPORT_STATIC_ALLOCATION = 1`), este archivo implementa la función `vApplicationGetIdleTaskMemory`, la cual reserva memoria en tiempo de compilación para la "Idle Task" (la tarea que se ejecuta cuando ninguna otra tarea tiene nada que hacer).

---

### 2. Evolución de `SysTick` y `SystemCoreClock`

* **`SystemCoreClock`**:
    1.  En el **Reinicio (`Reset_Handler`)**, el microcontrolador arranca usando su oscilador interno (HSI) por defecto, que normalmente es de 8 MHz.
    2.  Dentro de **`main()`**, al llamar a `SystemClock_Config()`, el código reconfigura el reloj usando el PLL. Toma el HSI dividido por 2 (4 MHz) y lo multiplica por 16, logrando una frecuencia de **64 MHz**.
    3.  A partir de ese momento, la variable global `SystemCoreClock` (que la HAL actualiza internamente) pasa a valer **64,000,000**.
* **`SysTick`** (El temporizador del sistema del núcleo Cortex-M):
    1.  En el **Reinicio**, el SysTick está apagado.
    2.  Generalmente, `HAL_Init()` enciende un temporizador para llevar la cuenta de los milisegundos. Sin embargo, en proyectos de CubeMX con FreeRTOS, se suele desplazar a la HAL hacia otro Timer (en este caso el TIM4) para dejar el SysTick libre.
    3.  Cuando se llama a **`osKernelStart()`**, FreeRTOS configura e inicia el SysTick para que se desborde exactamente a la frecuencia dictada por `configTICK_RATE_HZ` (1000 veces por segundo). A partir de ese momento, el SysTick empieza a evolucionar libremente como el "marcapasos" del RTOS.

---

### 3. Comportamiento del programa desde `Reset_Handler` hasta antes del `while(1)`

El flujo de ejecución es estrictamente lineal antes de que el RTOS tome el control:

1.  **Arranque (Ensamblador):** Entra a `Reset_Handler`. Copia las variables globales iniciadas de la memoria Flash a la RAM y limpia a cero el resto de variables globales. Llama a inicializadores de librerías (`__libc_init_array`).
2.  **Entrada a C:** Salta a `main()`.
3.  **Configuración de Hardware:**
    * `HAL_Init()` resetea los periféricos.
    * `SystemClock_Config()` acelera el procesador a 64 MHz.
    * Se configuran los pines GPIO (LD2, B1), la consola UART2 y el Timer 2.
4.  **Inicio de Periféricos:** Se encienden las interrupciones base de TIM2 (`HAL_TIM_Base_Start_IT`).
5.  **Entorno de Usuario:** Se invoca `app_init()` (donde presumiblemente el usuario pone configuraciones iniciales propias).
6.  **Creación de Objetos del RTOS:** Se reserva memoria e inicializa la estructura para `defaultTask`.
7.  **Arranque del Kernel:** Se llama a `osKernelStart()`.
8.  **PUNTO DE NO RETORNO:** La función `osKernelStart()` toma el control de los registros del procesador, activa el SysTick y salta a ejecutar la primera tarea (`defaultTask`).
    * *Nota:* El programa **nunca llega al `while(1)`** de `main.c`. Esa porción de código actúa únicamente como un "sumidero" a prueba de fallos. Si la ejecución llega ahí, significa que `osKernelStart()` falló gravemente (casi siempre por falta de memoria Heap para crear la tarea Idle).

---

### 4. Interacción de SysTick y Timer 2 (TIM2) con FreeRTOS

Ambos actúan como fuentes de tiempo para el Sistema Operativo, pero con propósitos totalmente distintos:

* **SysTick:** Es el **Reloj principal (Heartbeat)** de FreeRTOS. Interrumpe al procesador 1000 veces por segundo (1 ms). En cada interrupción (`SysTick_Handler`), FreeRTOS evalúa si una tarea que estaba en pausa ya debe despertar (por ejemplo, después de un `osDelay(1)`), o si debe forzar un cambio de contexto (Preemption) para darle tiempo de CPU a otra tarea de igual o mayor prioridad.
* **Timer 2 (TIM2):** Es un temporizador de **Alta Frecuencia para Estadísticas (Run-Time Stats)**. En `FreeRTOSConfig.h` se definió `configGENERATE_RUN_TIME_STATS 1`. FreeRTOS requiere un temporizador que sea *mucho más rápido* que el SysTick para medir con precisión (a nivel de microsegundos) cuánto tiempo de CPU consume exactamente cada tarea. `TIM2` está configurado para interrumpir frecuentemente, incrementando la variable `ulHighFrequencyTimerTicks`, la cual FreeRTOS lee usando la macro `portGET_RUN_TIME_COUNTER_VALUE`.

---

### 5. Interacción del Timer 4 (TIM4) con la HAL del proyecto STM32

El **Timer 4 (TIM4)** funciona como la **Base de Tiempo de la HAL (Hardware Abstraction Layer)**.

En un proyecto sin RTOS, la HAL usa el SysTick para llevar la cuenta del tiempo (usada por ejemplo en `HAL_Delay()` o en los Timeouts de funciones de hardware como `HAL_UART_Receive`). Sin embargo, dado que **FreeRTOS monopoliza el SysTick** para su planificador de tareas, si la HAL y FreeRTOS compartieran el mismo temporizador, ocurrirían colisiones críticas de interrupciones o interbloqueos, especialmente antes de que el RTOS inicie o si el planificador se suspende.

Para solucionar esto de manera segura:
1.  Se independiza a la HAL asignándole el **TIM4**.
2.  Cada vez que el TIM4 se desborda, lanza su interrupción (`TIM4_IRQHandler`).
3.  Esta interrupción llama a `HAL_TIM_PeriodElapsedCallback()`, que podemos ver en `main.c`.
4.  Dentro de ese callback, el código verifica si el Timer es el TIM4 e invoca **`HAL_IncTick()`**. Esto incrementa la variable interna `uwTick` de la HAL, permitiendo que todas las funciones de ST funcionen independientemente de si FreeRTOS está ejecutándose o no.

# Paso 8

Este conjunto de archivos implementa una aplicación embebida basada en **FreeRTOS** (usualmente sobre un microcontrolador STM32, a juzgar por el uso de la HAL de ST). La arquitectura del software utiliza **Máquinas de Estados Finitos (FSM - Statecharts)** ejecutándose dentro de tareas (threads) del sistema operativo en tiempo real. 

A continuación, se presenta un análisis detallado del funcionamiento de cada archivo:

### 1. `app.c` - Inicialización de la Aplicación
Este archivo es el punto de entrada de la lógica de usuario antes de ceder el control al planificador (scheduler) de FreeRTOS.
* **Función `app_init()`:** * Inicializa variables globales de diagnóstico (`g_app_tick_cnt`, `g_task_idle_cnt`, etc.) que sirven para perfilar el rendimiento.
  * Utiliza la API de FreeRTOS `xTaskCreate` para instanciar dinámicamente dos tareas: **`Task BTN`** (monitor del botón) y **`Task LED`** (control del LED).
  * Ambas tareas se crean con la misma prioridad (`tskIDLE_PRIORITY + 1ul`), lo que significa que el RTOS alternará el tiempo de CPU entre ambas (Time Slicing) si ambas están listas para ejecutarse.

### 2. `task_btn.c` - Tarea de Gestión del Botón
Contiene la lógica para leer el estado de un pulsador (botón de usuario) de manera robusta.
* **El Bucle Principal (`task_btn`):** Es un bucle infinito que invoca continuamente a la función `task_btn_statechart()`.
* **La Máquina de Estados (`task_btn_statechart`):** Implementa una lógica "Anti-Rebote" (Debounce) no bloqueante utilizando 4 estados:
  1. **`ST_BTN_XX_UP`**: Estado de reposo (botón sin presionar). Si detecta un nivel bajo, guarda el instante de tiempo (`xTaskGetTickCount()`) y pasa a FALLING.
  2. **`ST_BTN_XX_FALLING`**: Espera que transcurra un tiempo de validación (`DEL_BTN_XX_MAX` = 50ms). Si después de 50ms el botón sigue presionado, lo considera un "Toque Válido". Imprime un mensaje en el log, envía el comando `EV_LED_XX_BLINK` a la tarea del LED y pasa al estado DOWN.
  3. **`ST_BTN_XX_DOWN`**: Espera a que el botón sea soltado. Al soltarse, guarda el tiempo y pasa a RISING.
  4. **`ST_BTN_XX_RISING`**: Espera otros 50ms para filtrar el rebote mecánico de soltar el botón. Si sigue suelto, manda la orden `EV_LED_XX_OFF` a la tarea del LED y vuelve al estado UP.

### 3. `task_led_interface.c` - Interfaz de Comunicación
Este archivo actúa como un puente o API para la comunicación entre tareas, evitando que la tarea del botón modifique directamente las variables privadas de la tarea del LED.
* **Función `put_event_task_led()`:** Recibe un evento (ej. encender o apagar el parpadeo) y lo guarda en la estructura global compartida `task_led_dta`, levantando una bandera (`flag = true`). Es el mecanismo de comunicación (Inter-Task Communication) que avisa a la máquina de estados del LED que hay un nuevo comando por procesar.

### 4. `task_led.c` - Tarea de Control del LED
Controla el encendido y apagado físico del LED basándose en las órdenes que recibe del botón.
* **El Bucle Principal (`task_led`):** Apaga el LED al arrancar e ingresa a un bucle infinito evaluando `task_led_statechart()`.
* **La Máquina de Estados (`task_led_statechart`):** Tiene 2 estados principales:
  1. **`ST_LED_XX_OFF`**: El LED está apagado. Constantemente evalúa si la interfaz levantó la bandera (`flag == true`) y si el evento es `EV_LED_XX_BLINK`. Si es así, baja la bandera, enciende el LED, registra el tiempo actual y salta al estado BLINK.
  2. **`ST_LED_XX_BLINK`**: Primero evalúa si el botón mandó una orden de apagado (`EV_LED_XX_OFF`). Si la recibe, apaga el LED y vuelve a OFF. Si *no* hay orden de apagado, calcula si ya pasaron 500ms (`DEL_LED_XX_MAX`) desde el último cambio. Si el tiempo se cumplió, utiliza `HAL_GPIO_TogglePin` para invertir el estado físico del LED, logrando así el efecto de parpadeo (Blink) constante.

### 5. `freertos.c` - Callbacks (Hooks) del Sistema
Aquí se implementan funciones "Hook" que FreeRTOS llama automáticamente bajo ciertas condiciones del sistema (útiles para telemetría y debugging):
* **`vApplicationIdleHook()`:** Se ejecuta automáticamente solo cuando ninguna de las tareas (`BTN` o `LED`) tiene nada que hacer. En este código, incrementa un contador (`g_task_idle_cnt`). En sistemas reales, aquí se suele poner al microcontrolador a dormir para ahorrar batería.
* **`vApplicationTickHook()`:** Se ejecuta con cada interrupción del "Tick" del RTOS (típicamente cada 1 milisegundo). Incrementa el contador global `g_app_tick_cnt`.
* **`vApplicationStackOverflowHook()`:** Es una rutina crítica de seguridad. Si FreeRTOS detecta que alguna de las tareas (Botón o LED) consumió más memoria RAM (Stack) de la que se le asignó al crearla, salta a esta función. Aquí, el sistema deshabilita las interrupciones (`taskENTER_CRITICAL`) y se queda colgado intencionalmente (`configASSERT(0)`) para que el desarrollador pueda depurar el fallo catastrófico.

### Resumen del flujo general:
1. El sistema arranca y crea las tareas `BTN` y `LED`.
2. Las tareas entran en sus bucles infinitos, compartiendo la CPU.
3. El usuario presiona el botón físico.
4. `task_btn` valida el tiempo (50ms) y avisa mediante `put_event_task_led()`.
5. `task_led` detecta el aviso y empieza a conmutar (hacer titilar) el pin del LED cada 500ms de forma asíncrona.
6. Al soltar el botón de nuevo, `task_btn` manda la señal de apagado, deteniendo la animación de parpadeo.
