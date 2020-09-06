# I2C sólo escritura.

*La intención de este tutorial es enseñarte a diseñar y manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizando como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).* Vamos a sobre-entender que estamos dentro de frecuencias de entre 100 y 400 KHz, y el ancho de la dirección es de 7 bits.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

### Tensión de alimentación.

Hoy en día existen dos niveles de voltaje que puede ser de 5v o de 3,3v. Cuando el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tension_33_5.PNG)

Los hay de 4 y 8 bits. El I2C sólo tiene 2 líneas (SDA y SCL) entonces usaríamos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3,3v y que el de 3,3v tolere líneas de comunicaciónes de 5v. Si ese fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico I2C que vas a utilizar, y si no menciona nada sobre ello, has de mirar a partir de qué voltaje se considera que un cero es 0 y que un uno es 1. La Alhambra-II (FPGA) da señales de salida de 3,3v. Si el periférico (el esclavo) se alimenta con 5v hemos de tener esto presente.

### Las resistencias pull-up (innecesarias para nosotros).

El valor de las dos resistencias pull-up (Rp) no son críticas. De forma estandar se suele poner un valor de 4k7, pero puedes utilizar valores un poco mayores o menores y funcionará igual de bien. Sin embargo a nosotros **no nos hará falta esas resistencias** porque vamos a utilizar el I2C siempre como escritura, nunca como lectura, entonces las señales de SDA y SCL siempre-siempre van del maestro al/los esclavo(s), y el maestro (FPGA) mantendrá los niveles de tensión (ya sean 0s ó 1s) en las líneas sin entrar nunca en estado flotante o triestado. Si el periférico ya las lleva incluidas, no pasa nada, seguirá funcionando igual de bien que si no las tuviera.

# Protocolo I2C.

El protocolo I2C se basa en dos líneas de comunicación, el funcionamiento es síncrono, es decir, el dato (SDA) se valida con un ciclo de reloj (SCL). Pese a que existen muchas imágenes de ejemplo donde se aprecia sus ondas, las más intuitivas son las que pondré en este tutorial.

Recuerda que vamos a utilizar periféricos I2C que sólo necesitan leer (el maestro I2C es el que escribe), es decir, que **no se puede conectar periféricos que envían datos al maestro I2C.** Por ejemplo, no se puede conectar periféricos I2C del tipo: ADC, memorias externas, sensores, relojes de tiempo real, etc. Sí se puede conectar DACs, puertos de salida, resistencias variables digitales, OLEDs monocromáticas, displays series, etc.

Nosotros vamos a tratar el I2C como si fuese un registro de desplazamiento con algunas particularidades. Quédate con esta idea y así lo verás todo más sencillo.

### Señal "start" y "stop".

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/start__stop.png)

La parte más complicada a la hora de diseñar un maestro I2C es crear la señal **start** y **stop** porque tienen una forma particular y han de englobar toda la cadena de datos (dirección + datos). Es decir, que no funcionan con la misma filosofía que cuando se transmiten los datos en sí.

**Start:** Si observamos la imagen de arriba vemos que inicialmente tanto la línea de datos (SDA) y el reloj (SCL) se mantienen en alto, así se mantendrá mientras no haya transferencia de datos. Cuando vamos a transmitir datos ha de comenzar de la siguiente manera: estando las dos líneas en alto, primero ha de ponerse a 0 la línea SDA, y después ha de hacer lo mismo la línea SCL. Ese es el aviso de que se van a transmitir datos y el periférico I2C se preparará para recibirlos. En la imagen se aprecia como área roja izquierda.

**Repeat start:** Esta señal de color rosácea no la vamos a tener en cuenta, pero es bueno que sepas que podría existir. Algunos de los periféricos I2C con los que he experimentado la contemplan, pero sigue funcionando si no la tiene. Se utiliza para volver a hacer un **start** cada vez que termina un paquete de datos o de dirección, avisando de que viene otro.

**Stop:** Para saber que ha finalizado la transmisión de datos hay que hacer lo mismo que sucede con la señal *start* pero de forma inversa, es decir, estando las dos líneas en estado bajo, primero a de subir la línea SCL y después SDA. Y como es el final de envío de información ambas líneas han de mantenerse en alto.

* La señal **start** siempre-siempre antecederá el paquete de dirección.
* La señal **stop**  siempre-siempre estará al final del paquete de dato(s). El paquete de datos puede ser uno o muchos, no importa. Esta señal ha de ocurrir al finalizar el envío de un paquete de información.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/scl_sda_data.PNG)

*Más en detalle*: La señal **start** y **stop** se producen cuando **SCL está en estado alto y SDA cambia**. Si SCL la mantenemos en estado alto y SDA se pone a 0, producimos la señal de start; y si SCL se mantiene en alto y en ese momento hacemos que SDA pase a 1, provocamos la señal de stop.

Sólo se interpretará y validará como bit de dato cuando en el transcurso de producirse la señal SCL comenzó y terminó con el mismo valor en SDA. En el siguiente punto se explica mejor esto.

### Señal de validación de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/validacionDelDato.PNG)

Si queremos diseñar un maestro I2C de sólo escritura fiable (que sea compatible con todos los periféricos) hemos de cumplir esta regla de oro y es que **la señal de dato ha de englobar por completo la señal del reloj** es decir, que el flanco de subida y de bajada de SCL ha de estar dentro del periodo que dura el dato. En la imagen lo podemos ver como el relleno de color verde. Esto es muy importante, ya que si la señal SCL antedeciera o precediera al dato (SDA) se podría interpretar como una señal de start o de stop, según el caso.

Una forma sencilla de ver esto es imaginar un registro de desplazamiento, en el que no sabemos si funciona por flanco de subida o de bajada, entonces lo que haríamos es englobar ambos casos. Primero sacamos el dato y luego creamos un pulso de validación (un estado alto de periodo concreto) y cuando el pulso de validación vuelve al estado bajo es entonces cuando podemos sacar el siguiente dato.

### Paquete de dirección.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/send_address.PNG)

El paquete de dirección se compone de 9 bits: dirección del periférico I2C que son 7 bits (en rojo), bit de lectura/escritura o R/W (en verde) y bit de ACK o reconocimiento (en azul).
El paquete que contiene la dirección del periférico ha de estar precedida con la secuencia **start** (área amarilla), esta secuencia es lo primero que ha de salir antes de comenzar a enviar los 0s y 1s del paquete de dirección.

Como vamos a diseñar un maestro I2C de sólo escritura entonces el bit **R/W** siempre será 0. El bit **ACK** es un bit de lectura para comprobar si algún periférico lo ha recibido, esperando el maestro I2C un 0 para confirmarlo. Como no puede hacer lectura y estamos seguros de que el periférico está conectado, este bit también lo fijamos a 0.
Es decir, que **en el paquete de dirección los bits R/W y ACK se fijan a 0**. No debemos preocuparnos por verificar los datos. La única preocupación que has de tener es asegurarte de que la distancia entre el maestro y el periférico sea lo más corto posible; yo suelo usar cables de unos 20cm. He probado con periféricos I2C del tipo resistencia variable digital, DAC, puerto de salida y OLED por largas horas y sin problemas. El I2C fue diseñado para comunicar información dentro de una PCB o dentro de un módulo de shields, sin embargo acepta distancias muchísimo mayores.

### Paquete de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/data01.PNG)

El paquete de dato también se compone de 9 bits (y esto nos facilita enormemente el diseño), que son los 8 bits de datos (un byte) más otro bit de ACK. Todos los bits de ACK siempre lo haremos valer 0. Un paquete de datos puede estar compuesto por uno o muchos "bytes" (con su ACK correspondiente) y se envían de forma consecutiva, lo único importante aquí es saber que el último "byte" de dato ha de terminar con la señal stop, que es la forma de indicar que hemos finalizado el envío. En la imagen se pone como ejemplo el envío de dos "bytes" (con su correspondiente ACK) y al último se le añade la señal de stop.

La gran mayoría de las veces los periféricos (esclavos I2C) el número de "bytes" a enviar son fijos. Pueden ser un sólo "byte", o de dos, o de tres, incluso de 4, pero existen periféricos que ese tamaño podría ser variable, y eso es lo que sucede si queremos manejar una pantalla OLED (por ejemplo).

Por esta razón en el diseño de un maestro I2C genérico (que contemple la mayoría de casos) hemos de diseñarlo de tal forma que podamos enviar desde 1 "byte" hasta miles de ellos de una sola tacada.

# Diseño electrónico.

Lo más probable es que si a 50 personas les pidiéramos que diseñaran un maestro I2C de sólo escritura, obtendríamos 50 diseños diferentes. Voy a poner la versión que hice, que seguramente se podría mejorar y/u optimizar. La cuestión es que funciona bien y hasta el momento no he tenido ningún problema. Si quieres hacer tu propia versión todo lo que explico te puede ahorrar mucho trabajo.

El diseño general es este e iremos por partes como Jack el destripador.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/general_scheme_master_i2c.PNG)

Comenzamos con la creación de las señales start y stop, y entre esas dos señales ha de ir todo el paquete de datos. Para ello se me ocurrió utilizar dos multiplexores, el de arriba se encarga de la señal SDA y el de abajo de la señal SCL.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/mux_data.PNG)

Observa las constantes (0s y 1s) que hay en ambos multiplexores, los bits de entrada i0 e i1 se encargan de crear la secuencia de inicio (start), el de arriba corresponde a la secuencia en SDA y el de abajo el de la secuencia SCL. Cuando termine la secuencia start todos los datos pasarán por i2 de ambos, en SDA todos los que corresponda (incluido la dirección) y en SCL el pulso correspondiente con duración determinada que valida dicho dato. El contador emarcado en verde es el que dirije esta orquesta. Como puedes intuir no dura el mismo tiempo la multiplexación de cada una de las entradas, de eso se encarga otros circuitos. Finalmente, al finalizar los datos pasa a la creación de la secuencia de Stop.

Ahora viene una parte interesante del circuito, se trata de tener una entrada donde podamos decirle cuántos bytes de datos van a ser enviados, en ese paquete de datos ha de estar incluido el de dirección, que será el primero que se envíe. Esto significa que si ponemos por ejemplo un valor de 3 en la entrada "nbytes", se enviará esa cantidad de datos en serie, uno de dirección y dos de datos en este ejemplo. El bloque "DATA" se encargará de que cada byte a enviar lleve su ACK correspondiente. Los bytes de cada dato a enviar han de ir entrando desde el exterior (la dirección incluida). Más adelante veremos este procedimiento.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/data_transfer.PNG)

Vemos que la cantidad de datos a enviar (nbytes) se multiplica por 9. Se trata de pasar de bytes a bits y conocer la cantidad exacta de bits que han de salir por SDA. Como son 8 bits de datos más un bit de ACK el total es 9. Eso irá a un contador que junto a un comparador sabrá cuándo ha de terminar de transferir datos en serie y pasar a la siguiente secuencia, que sería finalmente la secuencia que forma el Stop.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/synchro.PNG)

El módulo encuadrado en rojo se encarga de sincronizar la frecuencia I2C con la entrada start del circuito. Lo que hace es obligar a poner en marcha el circuito justo cuando la frecuencia I2C está en un flanco de bajada. Esto obliga a que la señal de la secuencia Start del I2C dure siempre el mismo periodo.

El módulo encuadrado en azul lo que hace es tomar el primer ciclo de la señal de la frecuencia del I2C y la descompone en dos pulsos por la salida "Tics2", el primero cuando hace el flanco de subida y el segundo cuando hace flanco de bajada, estos dos pulsos activa en otro módulo la memorización de la cantidad de bits que se van a transmitir.
Terminado estos dos pulsos, que se corresponde con la secuencia Start, deja de actuar y ahora los pulsos saldrán por la salida "shift" de dicho módulo, que irá desplazando cada uno de los bits en SDA con su correspondiente pulso de validación en SCL.

El módulo encuadrado en verde se encarga de avisar al exterior de que se ha completado el envío de 9 bits (un byte + ACK), para que le pase el siguiente dato a enviar.

El resto de módulos son ajustes de sincronización para la señal Stop del I2C. Evité erróneamente utilizar un bit más del multiplexor que formaría la secuencia completa, sin embargo decidí finalizar la señal Stop de forma artificiosa. En el futuro arreglaré esta parte, que pese a que funciona bien, es optimizable a nivel de recursos.

# El cerebro de la béstia!

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/brain_i2c_only_write.PNG)

Ahora convertimos todo lo anterior en un módulo, el resultado es como se aprecia en la imagen. Tenemos una entrada de 16 bits donde indicaremos la cantidad de bytes (**nbytes**) que vamos a enviar, en el ejemplo de la imagen son sólo 3 (dirección + dos de datos), pero tiene capacidad de poder enviar hasta 7281 bytes (65536/9) de una tacada. Tenemos una entrada de datos de 8 bits (**data**), que son los datos en sí, ahí iremos poniendo cada uno de los bytes que serán enviados cuando corresponda y para ello nos ayudaremos exteriormente de una máquina de contar (otro módulo), que a su vez irá validando cada uno de esos bytes a través de la patilla **exec**. Cuando finaliza el envío de un byte con su ACK correspondiente, emite una señal de **next** para avisar a la máquina de contar que puede contar una unidad más. Luego veremos cómo se monta todo esto y varios ejemplos. Cuando finaliza todo el envío (los 3 bytes del ejemplo) y se haya producido la señal de Stop, saldrá un tic por la patilla **done**.

(continuará)
