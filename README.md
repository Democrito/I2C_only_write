# I2C sólo escritura.

*La intención de este tutorial es enseñarte a ser capaz de manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizando como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).*

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

### Tensión de alimentación.

Hoy en día existen dos niveles de voltaje que puede ser de 5v o de 3.3v. Cuando sucede que el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.

**Nota:** Uso la palabra *tensión* y *voltaje* indistintamente, pero es lo mismo.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tensi%C3%B3n_33_5.PNG)

Los hay de 8 bits y de 4 bits. Como el I2C sólo tiene 2 líneas (SDA y SCL) entonces usaremos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3.3v y que el de 3.3v tolere líneas de comunicaciónes de 5v. Si ese fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico que vas a utilizar, y si no menciona nada sobre esto, has de mirar a partir de qué voltaje se considera que un 0 es cero y que un 1 es uno. Si las líneas son [*trigger Schmitt*](https://es.wikipedia.org/wiki/Disparador_Schmitt) has de mirar esa parte.

### Resistencias pull-up (innecesarias para nosotros).

El valor de las dos resistencias pull-up (Rpp) no son críticas. De forma estandar se suele poner un valor de 4k7, pero puedes utilizar valores un poco mayores o menores y funcionará igual de bien. Sin embargo a nosotros **no nos hará falta esas resistencias** porque vamos a utilizar el I2C siempre como escritura, nunca como lectura, entonces las señales de SDA y SCL siempre-siempre van del maestro al esclavo, y el maestro (FPGA) mantendrá los niveles de tensión (ya sean 0s ó 1s) en la línea sin entrar nunca en estado flotante o triestado.

### Protocolo I2C.

El protocolo I2C se basa en dos líneas para la comunicación, el funcionamiento es síncrono, es decir, el dato (SDA) se valida con una señal de clock (SCL). Pese a que existen muchas imágenes de ejemplo donde se aprecia sus ondas, la más fiable es esta, más abajo explicaré por qué.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/validacion_del_dato.PNG)

Si queremos diseñar un maestro I2C de sólo lectura fiable (que sea compatible con todos los periféricos) hemos de cumplir siempre esta regla de oro y es que **la señal de dato ha de englobar por completo la señal del reloj** es decir, que el flanco de subida y de bajada ha de estar dentro del periodo que dura el dato. En la imagen lo podemos ver como el relleno de color verde. Esto es muy importante porque hay periféricos que validan con flanco de subida y otros requieren de un periodo mínimo de duración de estado alto del reloj y por seguridad ese ciclo de reloj ha de estar dentro del ciclo del dato.

Una forma sencilla de ver esto es imaginar un registro de desplazamiento, en el que no sabemos si funciona por flanco de subida o por flanco de bajada, entonces lo que haríamos es englobar ambos casos. Primero sacamos el dato y luego creamos un pulso (un estado alto de periodo concreto) y cuando el pulso vuelve al estado bajo es entonces cuando sacamos el siguiente dato.



