# I2C sólo escritura.

*La intención de este tutorial es enseñarte a ser capaz de diseñar y manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizando como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).*

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

### Tensión de alimentación.

Hoy en día existen dos niveles de voltaje que puede ser de 5v o de 3.3v. Cuando sucede que el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.

Nota: Uso la palabra *tensión* y *voltaje* indistintamente, pero es lo mismo.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tensi%C3%B3n_33_5.PNG)

Los hay de 8 bits y de 4 bits. Como el I2C sólo tiene 2 líneas (SDA y SCL) entonces usaremos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3.3v y que el de 3.3v tolere líneas de comunicaciónes de 5v. Si ese fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico que vas a utilizar, y si no menciona nada sobre esto, has de mirar a partir de qué voltaje se considera que un 0 es cero y que un 1 es uno. Si las líneas son [*trigger Schmitt*](https://es.wikipedia.org/wiki/Disparador_Schmitt) has de mirar esa parte.

La Alhambra-II (FPGA) da señales de salida de 3,3v. Si el periférico se alimenta con 5v hemos de saber si es capaz de interpretar como 1 lógico un voltaje que no esté por debajo de los 2,8v (que lea como 1 lógico a partir de los 2,8v y no menos que eso). De lo contrario sería necesario un adaptador de tensión.

### Las resistencias pull-up (innecesarias para nosotros).

El valor de las dos resistencias pull-up (Rp) no son críticas. De forma estandar se suele poner un valor de 4k7, pero puedes utilizar valores un poco mayores o menores y funcionará igual de bien. Sin embargo a nosotros **no nos hará falta esas resistencias** porque vamos a utilizar el I2C siempre como escritura, nunca como lectura, entonces las señales de SDA y SCL siempre-siempre van del maestro al esclavo, y el maestro (FPGA) mantendrá los niveles de tensión (ya sean 0s ó 1s) en la línea sin entrar nunca en estado flotante o triestado. Si el periférico ya las lleva incluidas, no pasa nada, seguirá funcionando igual de bien que si no las tuviera.

# Protocolo I2C.

El protocolo I2C se basa en dos líneas de comunicación, el funcionamiento es síncrono, es decir, el dato (SDA) se valida con un ciclo de reloj (SCL). Pese a que existen muchas imágenes de ejemplo donde se aprecia sus ondas, las más fiables son estas que verás a continuación.

### Señal "start" y "stop".

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/start_stop.PNG)

La parte más complicada a la hora de diseñar un maestro I2C es crear la señal **start** y **stop**, no porque sean difíciles de hacer, sino porque esas señales tienen una forma peculiar y han de englobar toda la cadena de datos (dirección + datos). Es decir, que no funcionan con la misma filosofía que cuando se transmiten los datos en sí.

Si observamos la imagen de arriba veremos que inicialmente tanto la línea de datos (SDA) y el reloj (SCL) se mantienen en alto, y así se mantendrá mientras no haya transferencia de datos.

**Start:** Cuando vamos a transmitir datos ha de comenzar de la siguiente manera: primero ha de ponerse a 0 la línea SDA, y después ha de hacer lo mismo la línea SCL. Ese es el aviso de que se van a transmitir datos y el periférico se preparará para recibirlos. En la imagen se aprecia como el área roja izquierda.

**Repeat start:** Esta señal de color rosácea no la vamos a tener en cuenta, pero es bueno que sepas que también podría existir. Algunos de los periféricos I2C con los que he experimentado la contemplan, pero sigue funcionando si no la tiene. Se utiliza para volver a hacer un **start** cada vez que termina un paquete de datos o de dirección, avisando de que viene otro.

**Stop:** Para saber que ha finalizado la transmisión de datos hay que hacer lo mismo que sucede con la señal *start* pero de forma inversa, es decir, estando las dos líneas en estado bajo, primero a de subir la línea SCL y después SDA. Y como es el final de envío de información ambas líneas han de mantenerse en alto. 

### Señal de validación de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/validacionDelDato.PNG)

Si queremos diseñar un maestro I2C de sólo escritura fiable (que sea compatible con todos los periféricos) hemos de cumplir siempre esta regla de oro y es que **la señal de dato ha de englobar por completo la señal del reloj** es decir, que el flanco de subida y de bajada de SCL ha de estar dentro del periodo que dura el dato. En la imagen lo podemos ver como el relleno de color verde. Esto es muy importante porque hay periféricos que validan con flanco de subida y otros requieren de un periodo mínimo de duración de estado alto del reloj y por seguridad ese ciclo de reloj ha de estar dentro del ciclo del dato.

Una forma sencilla de ver esto es imaginar un registro de desplazamiento, en el que no sabemos si funciona por flanco de subida o de bajada, entonces lo que haríamos es englobar ambos casos. Primero sacamos el dato y luego creamos un pulso de validación (un estado alto de periodo concreto) y cuando el pulso vuelve al estado bajo es entonces cuando podemos sacar el siguiente dato.

### Byte de dirección.

Cada vez que enviemos un paquete de datos hemos de precederlo siempre con la dirección donde se encuentra el periférico que queramos que lo reciba. El paquete que contiene la dirección de nuestro periférico siempre ha de estar precedida con la secuencia **start**, dicha secuencia es lo primero que ha de salir antes de comenzar a enviar 0s y 1s.

El paquete de dirección se compone de 9 bits: dirección del periférico (7 bits), bit de escritura/lectura y bit de ACK (reconocimiento).

