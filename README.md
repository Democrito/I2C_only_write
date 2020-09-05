# I2C sólo escritura.

*La intención de este tutorial es enseñarte a ser capaz de diseñar y manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizando como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).*

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

### Tensión de alimentación.

El I2C de **sólo lectura** tiene unas peculiariedades y entre ellas está que no se usa resistencias pull-up, eso significa que se puede conectar como cualquier otro circuito convencional.

Hoy en día existen dos niveles de voltaje que puede ser de 5v o de 3,3v. Cuando sucede que el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.

Nota: Uso la palabra *tensión* y *voltaje* indistintamente, pero es lo mismo.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tensi%C3%B3n_33_5.PNG)

Los hay de 8 y de 4 bits. El I2C sólo tiene 2 líneas (SDA y SCL) entonces usaríamos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3,3v y que el de 3,3v tolere líneas de comunicaciónes de 5v. Si ese fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico que vas a utilizar, y si no menciona nada sobre ello, has de mirar a partir de qué voltaje se considera que un 0 es cero y que un 1 es uno.

La Alhambra-II (FPGA) da señales de salida de 3,3v. Si el periférico se alimenta con 5v hemos de saber si es capaz de interpretar como 1 lógico un voltaje que no esté por debajo de los 2,8v (lea como 1 lógico a partir de los 2,8v y no menos que eso). De lo contrario sería necesario un adaptador de tensión.

### Las resistencias pull-up (innecesarias para nosotros).

El valor de las dos resistencias pull-up (Rp) no son críticas. De forma estandar se suele poner un valor de 4k7, pero puedes utilizar valores un poco mayores o menores y funcionará igual de bien. Sin embargo a nosotros **no nos hará falta esas resistencias** porque vamos a utilizar el I2C siempre como escritura, nunca como lectura, entonces las señales de SDA y SCL siempre-siempre van del maestro al/los esclavo(s), y el maestro (FPGA) mantendrá los niveles de tensión (ya sean 0s ó 1s) en la línea sin entrar nunca en estado flotante o triestado. Si el periférico ya las lleva incluidas, no pasa nada, seguirá funcionando igual de bien que si no las tuviera.

# Protocolo I2C.

El protocolo I2C se basa en dos líneas de comunicación, el funcionamiento es síncrono, es decir, el dato (SDA) se valida con un ciclo de reloj (SCL). Pese a que existen muchas imágenes de ejemplo donde se aprecia sus ondas, las más intuitivas son las que pondré en este tutorial.

Recuerda que siempre vamos a utilizar periféricos que sólo necesitan leer (el maestro I2C es el que escribe), es decir, que **no se puede conectar periféricos que envían datos al maestro I2C.** Por ejemplo, no se puede conectar periféricos I2C del tipo: ADC, memorias externas, sensores, relojes de tiempo real, etc. Sí se puede conectar DACs, puertos de salida, OLEDs monocromáticas, displays series, etc.

Nosotros vamos a tratar el I2C como si fuese un registro de desplazamiento con alguna peculiariedades. Quédate con esta idea y así lo verás todo más fácil.

### Señal "start" y "stop".

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/start_stop.PNG)

La parte más complicada a la hora de diseñar un maestro I2C es crear la señal **start** y **stop**, no porque sean difíciles de crear esas señales, sino que tienen una forma particular y han de englobar toda la cadena de datos (dirección + datos). Es decir, que no funcionan con la misma filosofía que cuando se transmiten los datos en sí.

**Start:** Si observamos la imagen de arriba vemos que inicialmente tanto la línea de datos (SDA) y el reloj (SCL) se mantienen en alto, así se mantendrá mientras no haya transferencia de datos. Cuando vamos a transmitir datos ha de comenzar de la siguiente manera: primero ha de ponerse a 0 la línea SDA, y después ha de hacer lo mismo la línea SCL. Ese es el aviso de que se van a transmitir datos y el periférico se preparará para recibirlos. En la imagen se aprecia como área roja izquierda.

**Repeat start:** Esta señal de color rosácea no la vamos a tener en cuenta, pero es bueno que sepas que podría existir. Algunos de los periféricos I2C con los que he experimentado la contemplan, pero sigue funcionando si no la tiene. Se utiliza para volver a hacer un **start** cada vez que termina un paquete de datos o de dirección, avisando de que viene otro.

**Stop:** Para saber que ha finalizado la transmisión de datos hay que hacer lo mismo que sucede con la señal *start* pero de forma inversa, es decir, estando las dos líneas en estado bajo, primero a de subir la línea SCL y después SDA. Y como es el final de envío de información ambas líneas han de mantenerse en alto.

* La señal **start** siempre-siempre antecederá el paquete de dirección.
* La señal **stop**  siempre-siempre estará al final del paquete de dato(s). El paquete de datos puede ser uno o muchos, no importa. Esta señal ha de ocurrir al finalizar el envío de un paquete de información.

### Señal de validación de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/validacionDelDato.PNG)

Si queremos diseñar un maestro I2C de sólo escritura fiable (que sea compatible con todos los periféricos) hemos de cumplir esta regla de oro y es que **la señal de dato ha de englobar por completo la señal del reloj** es decir, que el flanco de subida y de bajada de SCL ha de estar dentro del periodo que dura el dato. En la imagen lo podemos ver como el relleno de color verde. Esto es importante porque hay periféricos que validan con flanco de subida y otros requieren de un periodo mínimo de duración de estado alto del reloj y por seguridad ese ciclo alto de reloj ha de estar dentro del periodo que dura el dato.

Una forma sencilla de ver esto es imaginar un registro de desplazamiento, en el que no sabemos si funciona por flanco de subida o de bajada, entonces lo que haríamos es englobar ambos casos. Primero sacamos el dato y luego creamos un pulso de validación (un estado alto de periodo concreto) y cuando el pulso de validación vuelve al estado bajo es entonces cuando podemos sacar el siguiente dato.

### Paquete de dirección.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/send_address.PNG)

El paquete de dirección se compone de 9 bits: dirección del periférico que son 7 bits (en rojo), bit de lectura/escritura (en verde) y bit de ACK (reconocimiento, en azul).
El paquete que contiene la dirección del periférico ha de estar precedida con la secuencia **start** (área amarilla), esta secuencia es lo primero que ha de salir antes de comenzar a enviar 0s y 1s.

Como vamos a diseñar un maestro I2C de sólo escritura entonces el bit **R/W** siempre será 0. El bit **ACK** es un bit de lectura para comprobar si algún periférico lo ha recibido, esperando el maestro I2C un 0 para confirmarlo. Como no puede hacer lectura y estamos seguros de que el periférico está conectado, este bit también lo fijamos a 0.
Es decir, que en **el paquete de dirección los bits R/W y ACK se fijan a 0**. No debemos preocuparnos por verificar los datos. La única preocupación que has de tener para asegurarte de que siempre va a funcionar correctamente es asegurarte de que la distancia entre el maestro y el periférico sea lo más corto posible; yo uso cables de unos 20cm. He probado con periféricos I2C del tipo resistencia variable, DAC, puerto de salida y OLED por largas horas y sin problemas. El I2C fue diseñado para comunicar información dentro de una PCB o dentro de un módulo de shields, sin embargo acepta distancias muchísimo mayores.

