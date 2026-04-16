# Paso 2

## ¿Cómo usar el parámetro de Tarea?

El parámetro de tarea debe compartirse en el momento de crearla, desde la función xTaskCreate().

Dentro de la tarea, debe procesarse ese void \* para capturar la información necesaria de la parametrización.

## ¿Cómo cambiar la prioridad de una Tarea ya creada?

Para cambiar la prioridad de una tarea, se utiliza (desde la tarea a modificar o desde otra tarea) 



vTaskPrioritySet(xTaskHandle, nuevaPrioridad);



Como parámetros se envía el Handle de la tarea y el valor nuevo de prioridad que debe tener.



# Paso 3

Se crearon las tareas para ambos botones, cada una recibe como parámetro un puntero al task\_btn\_dta que debe utilizar.



Lo que se observó es que si se usa uno u otro botón, el estado del LED cambia correctamente.

En el caso de presionar un botón y luego otro, el led recibe nuevamente el estado de BLINK pero lo ignora por estar en ese estado.

Cuando uno de los dos botones se libera, el LED recibe el estado de OFF y se apaga, aún con el otro botón aún presionado.



# Paso 4

SE modificó la prioridad de task\_led a (tskIDLE\_PRIORITY + 2ul) al momento de la creación. De esa forma se ejecuta primero.

Una vez inicializada la función, se invoca a vTaskPrioritySet(NULL, (tskIDLE\_PRIORITY + 1ul)); para equiparar la prioridad con las demás tareas y que puedan ejecutarse.

De esa forma, la secuencia de ejecución fué:



\[info]  

\[info] app\_init is running - Tick \[mS] =   0

\[info]  RTOS - Event-Triggered Systems (ETS)

\[info]  soe-tp0\_03-application: Demo Code

\[info]  

\[info] Task LED is running - Tick \[mS] =   0

\[info]  

\[info] Task BTN 1 is running - Tick \[mS] =   0

\[info]  

\[info] Task BTN 2 is running - Tick \[mS] =   1

