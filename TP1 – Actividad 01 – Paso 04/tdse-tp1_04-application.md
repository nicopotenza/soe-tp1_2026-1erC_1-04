# Paso 2

## ¿Cómo implementar el procesamiento periódico mediante una Tarea?

El procesamiento periódico se realiza mediante el uso de las función vTaskDelayUntil(), ya que es la que permite una ejecución periódica de tiempo fijo.

## ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?

La tarea IDLE se ejecuta cuando todas las demás tareas están bloqueadas. Se puede utilizar haciendo que las demás tareas sedan el tiempo de ejecución restante.



# Paso 3

Se declaró al inicio de task\_led():

&#x09;TickType\_t xLastWakeTime;

&#x09;const TickType\_t xPeriodo = pdMS\_TO\_TICKS(50);

Luego, dentro antes del for(;;), se realizó:

&#x09;xLastWakeTime = xTaskGetTickCount();

Dentro del for(;;), luego de invocar a task\_led\_status() se invoca a:

&#x09;vTaskDelayUntil(\&xLastWakeTime, xPeriodo);

Que se encarga de bloquear la ejecución del programa durante 50ms. Este tiempo está elegido para no impactar en el tiempo de cambio del blink, seteado en 500ms.



Para que funcione, hubo que habilitar el uso de la función vTaskDelayUntil(). Eso de realizó desde el .ioc, en el apartado

&#x09;Categories

&#x09;-> Middleware and Sofware Packs

&#x09;--> FREERTOS

&#x09;---> Include parameters

&#x09;----> Include definitions

&#x09;-----> vTaskDelayUntil -> Enabled



Luego se compiló y ejecutó. Se verificó que el funcionamiento operativo no se vió afectado. En cuanto al tiempo de ejecución, paso de utilizar el 50% del tiempo de procesador, a utilizar <1%, dejando >99% para el Task BTN.



# Paso 4

Se realizó lo mismo para el task\_BTN. En este caso, como el tiempo de delay de la tarea era de 50ms, se configuró un periodo de 5ms.



Luego se compiló y ejecutó. Se verificó que el funcionamiento operativo no se vió afectado. En cuanto al tiempo de ejecución, paso de utilizar el 50% del tiempo de procesador en el caso original, pasar a utilizar >99% en el paso 3, y ahora los tiempos de ejecución son:



Name		Run Time (%)

IDLE		   98%

Task BTN	    2%

Task LED	   <1%

