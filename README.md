# Diseño con FPGA de Maestro I2C de sólo escritura.

*La intención de este tutorial es aprender a diseñar y manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizo como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).* Vamos a sobre-entender que estamos dentro de frecuencias de entre 100 y 400 KHz, y el ancho de la dirección es de 7 bits.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

### Tensión de alimentación.

Hoy en día existe dos niveles de voltaje que puede ser 5V o 3,3V. Cuando el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tension_33_5.png)

Los hay de 4 y 8 bits. El I2C sólo tiene 2 líneas (SDA y SCL) entonces usaríamos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3,3V y que el de 3,3V tolere líneas de comunicaciónes de 5V. Si este fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico I2C que vas a utilizar. La Alhambra-II (FPGA) da señales de salida de 3,3V. Si el periférico (el esclavo) se alimenta con 5V hemos de tener esto presente. Más información sobre niveles de tensión haz [clic aquí.](https://www.youtube.com/watch?v=6SiGlechlNM)

### Las resistencias pull-up (innecesarias para nosotros).

El valor de las dos resistencias pull-up (Rp) no son críticas pueden rondar entre 50K y 1K, cuanto mayor es la velocidad menor ha de ser la resistencia. De forma estandar se suele poner un valor de 4k7. Sin embargo **no nos hará falta esas resistencias** porque vamos a utilizar el I2C siempre como escritura, nunca como lectura, entonces las señales de SDA y SCL siempre van del maestro al/los esclavo(s), y el maestro (FPGA) mantendrá los niveles de tensión (ya sean 0s ó 1s) en las líneas sin entrar nunca en estado flotante o triestado. Si el periférico ya las lleva incluidas, no pasa nada, seguirá funcionando igual de bien que si no las tuviera.

# Protocolo I2C.

El protocolo I2C se basa en dos líneas de comunicación, el funcionamiento es síncrono, es decir, el dato (SDA) se valida con un ciclo de reloj (SCL).

Recuerda que esta versión de maestro I2C sólo puede escribir, entonces **no se puede conectar periféricos que envían datos al maestro I2C.** Por ejemplo, no se puede conectar periféricos I2C del tipo: ADC, memorias externas, sensores, relojes de tiempo real, etc. Sí se puede conectar DACs, puertos de salida, resistencias variables digitales, OLEDs monocromáticas, displays series, etc.

### Señal "start" y "stop".

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/start__stop.png)

Lo complicado a la hora de diseñar un maestro I2C es crear las señales **start** y **stop** porque tienen una forma particular y no funcionan con la misma filosofía que cuando se transmiten los datos en sí.

**Start:** Si observamos la imagen de arriba vemos que inicialmente tanto la línea de datos (SDA) y el reloj (SCL) se mantienen en alto, así se mantendrá mientras no haya transferencia de datos (reposo o sin actividad). Cuando vamos a transmitir datos ha de comenzar de la siguiente manera: estando las dos líneas en alto, primero ha de ponerse a 0 la línea SDA, y después ha de hacer lo mismo la línea SCL. Ese es el aviso de que se van a transmitir datos y el periférico I2C se preparará para recibirlos. En la imagen se aprecia como área roja izquierda.

**Repeat start:** Esta señal de color rosácea no la vamos a tener en cuenta, pero es bueno que sepas que podría existir. Algunos de los periféricos I2C con los que he experimentado la contemplan, pero sigue funcionando si no la tiene. Se utiliza para volver a hacer un **start** cada vez que termina un paquete de datos o de dirección, avisando de que viene otro.

**Stop:** Para saber que ha finalizado la transmisión de datos (área roja derecha) hay que hacer lo mismo que sucede con la señal *start* pero de forma inversa, es decir, estando las dos líneas en estado bajo, primero a de subir la línea SCL y después SDA, y como es el final de envío de información ambas líneas han de mantenerse en alto.

* La señal **start** siempre antecederá el paquete de información (información = dirección + datos).
* La señal **stop**  siempre precederá el paquete de información.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/scl_sda_data.PNG)

*Más en detalle*: La señal **start** y **stop** se producen cuando **SCL está en estado alto y SDA cambia**.

Sólo se interpretará y validará como bit de dato cuando en el transcurso de producirse la señal SCL comenzó y terminó con el mismo valor en SDA. En el siguiente punto se explica mejor esto.

### Señal de validación de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/validacionDelDato.PNG)

Si queremos diseñar un maestro I2C de sólo escritura fiable hemos de cumplir esta regla de oro: **la señal de dato (el 1 o el 0) ha de englobar por completo la señal del reloj** es decir, que el flanco de subida y de bajada de SCL ha de estar dentro del periodo que dura el dato. En la imagen lo podemos ver como el relleno de color verde. Esto es muy importante, ya que si la señal SCL antedeciera o precediera al dato (SDA) se podría interpretar como una señal de start o de stop, según el caso.

En el gráfico está exagerado todo esto para que se aprecie mejor. Basta con unos micro-segundos o nano-segundos para considerar que el nivel alto de SCL queda dentro del bit (sea 0 ó 1) de SDA.

Una forma sencilla de ver esto es imaginar un registro de desplazamiento, en el que no sabemos si funciona por flanco de subida o de bajada, entonces lo que haríamos es englobar ambos casos. Primero sacamos el dato y luego creamos un pulso de validación (un estado alto de periodo concreto) y cuando el pulso de validación vuelve al estado bajo es entonces cuando podemos sacar el siguiente dato.

### Paquete de dirección.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/send_address.PNG)

El paquete de dirección se compone de 9 bits: dirección del periférico I2C que son 7 bits (en rojo), bit de lectura/escritura o R/W (en verde) y bit de ACK o reconocimiento (en azul).
El paquete que contiene la dirección del periférico ha de estar precedida con la secuencia **start** (área amarilla), esta secuencia (*start*) es lo primero que ha de salir antes de comenzar a enviar los 0s y 1s.

Como vamos a diseñar un maestro I2C de sólo escritura entonces el bit **R/W** siempre será 0. El bit **ACK** es un bit de lectura para comprobar si algún periférico lo ha recibido, esperando el maestro I2C un 0 para confirmarlo. Como no puede hacer lectura y estamos seguros de que el periférico está conectado, este bit también lo fijamos a 0.
Es decir, que **en el paquete de dirección los bits R/W y ACK se fijan a 0**. No debemos preocuparnos por verificar los datos. La única preocupación que has de tener es asegurarte de que la distancia entre el maestro y el periférico sea lo más corto posible; yo suelo usar cables de unos 20cm. El I2C fue diseñado para comunicar información dentro de una PCB o dentro de módulos tipo shields, sin embargo acepta distancias muchísimo mayores.

### Paquete de datos.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/data01.PNG)

El paquete de dato también se compone de 9 bits (y esto nos facilita enormemente el diseño), que son los 8 bits de datos (un byte) más otro bit de ACK. Todos los bits de ACK siempre lo haremos valer 0. Un paquete de datos puede estar compuesto por uno o muchos "bytes" (con su ACK correspondiente) y se envían de forma consecutiva, lo único importante aquí es saber que el último byte de dato ha de terminar con la señal stop, que es la forma de indicar que hemos finalizado el envío. En la imagen se pone como ejemplo el envío de dos "bytes" (con su correspondiente ACK) y al último se le añade la señal de stop.

La gran mayoría de las veces los periféricos (esclavos I2C) el número de bytes a enviar son fijos, pero existen periféricos que ese tamaño podría ser variable, porque antes de ponerlo en marcha necesita ser configurados (por ejemplo: cierto tipo de displays series, pantallas OLEDs, etc).

Por esta razón en el diseño de un maestro I2C genérico (que contemple la mayoría de casos) hemos de diseñarlo de tal forma que podamos enviar desde un sólo byte hasta miles de ellos de una sola tacada.

# Diseño electrónico.

Voy a poner la versión que hice. Si quieres hacer tu propia versión todo lo que leerás a continuación te puede ahorrar mucho tiempo.

El diseño general es este e iremos por partes como Jack el destripador.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/general_scheme_master_i2c.PNG)

Comenzamos con la creación de las señales start y stop, y entre esas dos señales ha de ir todo el paquete de información (dirección + datos). Para ello se me ocurrió utilizar dos multiplexores, el de arriba se encarga de la señal SDA y el de abajo de la señal SCL.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/mux_data.PNG)

Observa las constantes (0s y 1s) que están conectadas a ambos multiplexores, los bits de entrada i0 e i1 se encargan de crear la secuencia de inicio (start), el de arriba corresponde a la secuencia en SDA y el de abajo el de la secuencia en SCL. Cuando termine la secuencia "start" los datos pasarán por i2 de ambos, en SDA todos los datos (incluido la dirección) y en SCL el pulso correspondiente con duración determinada que valida dicho dato. El contador emarcado en verde es el que dirije esta orquesta. Como puedes intuir no dura el mismo tiempo la multiplexación de cada una de las entradas, de eso se encarga otros circuitos. Finalmente, al finalizar los datos pasa a la creación de la secuencia de "stop".

Ahora viene una parte interesante del circuito, se trata de tener una entrada donde podamos decirle cuántos bytes de datos van a ser enviados; en ese paquete de datos ha de estar incluido el de dirección, que será el primero que se envíe. Esto significa que si ponemos por ejemplo un valor de 3 en la entrada **nbytes**, se enviará esa cantidad de datos en serie, uno de dirección y dos de datos en este ejemplo (la dirección siempre se incluye como otro dato más porque se cuenta todo lo que sale como información, es decir, los 0s y 1s).

El bloque "DATA" se encargará de que cada byte a enviar lleve su ACK correspondiente. Los bytes de cada dato a enviar han de ir entrando desde el exterior (la dirección incluida). Más adelante veremos este procedimiento.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/data_transfer.PNG)

Vemos que la cantidad de datos a enviar (nbytes) se multiplica por 9. Se trata de pasar de bytes a bits y conocer la cantidad exacta de bits que han de salir. Como son 8 bits de datos más un bit de ACK el total es 9. Eso irá a un contador que junto a un comparador sabrá cuándo ha de terminar de transferir datos en serie y pasar a la siguiente secuencia, que sería finalmente la secuencia que forma el "stop".

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/synchro.PNG)

**Nota:** recomiendo tener el circuito abierto en Icestudio para poder hacer zoom, ya que las imágenes de aquí están "a vista de pájaro".

El módulo encuadrado en rojo se encarga de sincronizar la frecuencia I2C con la entrada "start" del circuito. Poner en marcha el circuito justo cuando la frecuencia I2C está en un flanco de bajada. Esto obliga a que la señal de la secuencia "start" del I2C dure siempre el mismo periodo.

El módulo encuadrado en azul tomar el primer ciclo de la señal de la frecuencia del I2C y la descompone en dos pulsos por la salida "Tics2", el primero cuando hace el flanco de subida y el segundo cuando hace flanco de bajada, estos dos pulsos crean la señal **start** y a la vez activa la comparación de número máximo de bits a enviar en otro módulo. Terminado estos dos pulsos, que se corresponde con la secuencia **start**, deja de actuar esa salida y los siguientes pulsos se conmuta por la salida "shift" de dicho módulo, que irá desplazando cada uno de los bits en SDA con su correspondiente pulso de validación en SCL.

El módulo encuadrado en verde se encarga de avisar al exterior de que se ha completado el envío de 9 bits (un byte + ACK), para que le pase el siguiente dato a enviar.

El módulo encuadrado en amarillo tiene la función de detectar el fin de envío de datos. En su interior hay un comparador que compara el número de bits total a enviar con el número de bits enviado hasta el momento. Cuando la comparación se cumple (es igual) da un tic por "end". La entra 'p' y la salida 'o' es transparente, es decir, un cable que envía los dos pulsos iniciales al contador para formar la señal **start**. Este "cable" también se encarga de permitir que el comparador pueda estar activo; de esto se encarga un flip-flop tipo D que hay en el interior.

El módulo encuadrado en rosa-chiclé tiene la función de construir la señal **stop**. Toma el último ciclo de la frecuencia I2C y la descompone en dos pulsos (esto también lo hicimos para fabricar la señal start en otro módulo) y se lo pasa al contador que arbitra los multiplexores. Una vez que se produce los dos pulsos se resetea ciertos contadores y módulos. 

Queda sin enmarcar un contador de 16 bits, que es el encargado de contar los bits que van saliendo. Este contaje va al módulo amarillo para compararlo con el número máximo de bits a enviar.

# El cerebro de la bestia!

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/brain_i2c_only_write.PNG)

Ahora convertimos todo lo anterior en un módulo, el resultado es como se aprecia en la imagen. Tenemos una entrada de 16 bits donde indicaremos la cantidad de bytes (**nbytes**) que vamos a enviar, en el ejemplo de la imagen son sólo 3 (dirección + dato + dato), pero **tiene capacidad de poder enviar hasta 7281 bytes (65536/9)** de una tacada incluyendo el byte de dirección. Tenemos una entrada de datos de 8 bits (**data**), que son los datos en sí a enviar, ahí iremos poniendo cada uno de los bytes que serán enviados cuando corresponda y para ello nos ayudaremos exteriormente de una máquina de contar (otro módulo), que a su vez irá validando cada uno de esos bytes a través de la patilla **exec** de la máquina de contar. Cuando finaliza el envío de un byte con su ACK correspondiente, emite una señal de **next** para avisar a la máquina de contar que puede contar una unidad más. Luego veremos cómo se monta todo esto y varios ejemplos. Cuando finaliza todo el envío (los 3 bytes del ejemplo) el maestro I2C produce un tic de *done* conectado al *stop* de la máquina de contar y finaliza el envío de datos.

### Ejemplo 1. (Con pulsador)

Voy a poner el ejemplo práctico más sencillo que existe, se trata de enviar un sólo byte de dato pero hay que tener presente que ese dato ha de ir acompañado de la dirección a la que tiene que ir. Es decir, que vamos a enviar primero la dirección y luego un sólo byte de dato.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/example_to_send_2_bytes_with_push_button_tick.PNG)

* Este ejemplo lo puedes descargar haciendo [clic aquí.](https://github.com/Democrito/I2C_only_write/blob/master/examples/Example_to_send_2_bytes_with_push_button_tick.ice) (Clic con el botón derecho del ratón al botón que pone "**Raw**" y después elegir la opción "Guardar enlace cómo", o lo más parecido a eso, depende del browser que uses.)

* Para este ejemplo he utilizado el chip **PCF8574**, más información haciendo [clic aquí.](http://www.aquihayapuntes.com/indice-practicas-pic-en-c/expansor-de-e-s-pcf8574.html)

La dirección de envío en decimal es **39** y también se puede escribir en hexadecimal como **'h27**, cada uno/a que tome el formato que más le guste. Por otro lado tenemos un dato a enviar que es constante con valor de **85** en decimal. Vamos a imaginar que tenemos ese chip conectado a nuestra FPGA, subimos el diseño y pulsamos "SW1" de nuestra Alhambra (FPGA). Verás que los leds se encienden de forma alterna. Si ahora cambias el dato de envío por **170** y lo subes y vuelves a pulsar sobre "SW1", comprobarás que los led que estaban apagados ahora están encedidos y vice-versa. El chip **PCF8574** tiene un pin que tiene salida con colector abierto, así que si ves un led que se comporta extraño, es ese pin (necesita una resistencia en configuración pull-up y funcionará de modo inverso a la lógica normal).

Podemos obsevar que la constante de la dirección (el 39) es multiplicado por 2. La dirección en realidad es de 7 bits, pero nos falta añadir el bit R/W, que es el bit más bajo (sin contar con ACK), y el bit R/W es siempre 0 porque siempre vamos a escribir, entonces para añadir ese 0 de forma automática se ha de multiplicar por 2 y ese 0 quedará añadido, y ahora sí que tenemos un byte verdadero de 8 bits. Se puede evitar esta multiplicación si ya tienes en cuenta que le falta el bit más bajo (R/W) y le pones directamente la dirección para escribir (la dirección multiplicada por 2 porque es para escribir).

Como vamos a enviar dos bytes (dirección + dato) podemos usar un multiplexor de 8 bits de dos entradas. La máquina de contar, una vez que recibe el tic en **start** (pulsando SW1) se encargará de poner el primer dato (q=0) dará un tic por "exec", y cuando se haya enviado, recibirá un tic por "next", pondrá q=1 y multiplexará esta vez el dato "85" a enviar validándolo otra vez a través del pin "exec". Como le hemos dicho al *cerebro* I2C que enviamos dos bytes (dirección + dato) le llegará a la máquina de contar un tic de *done* en el *stop*. De todas formas se habría parado porque sólo puede contar hasta 1 (0 y 1), ya que la máquina de contar es de sólo un bit.

### Ejemplo 2. (A través del serial)

Utilizaremos el ejemplo anterior, pero en vez de usar las engorrosas constantes fijas, vamos a utilizar el serial para poner el valor que queramos (0..255) en cualquier momento.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/Example_to_send_2_bytes_through_the_serial.PNG)

* Este ejemplo lo puedes descargar haciendo [clic aquí.](https://github.com/Democrito/I2C_only_write/blob/master/examples/Example_to_send_2_bytes_through_the_serial.ice)

Desde un terminal serie escribimos un número comprendido entre 0 y 255, le damos a *enviar* o *enter* y ese dato aparecerá en "i1" del multiplexor, acto seguido el serial produce un tic de validación (done) que llega al "start" de la máquina de contar, y al igual que en el ejemplo anterior, enviará a través del I2C el dato que hayamos puesto.

### Ejemplo 3. (Ahora enviaremos 3 bytes)

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/Example_to_send_3_bytes_through_the_serial.PNG)

* Este ejemplo lo puedes descargar haciendo [**clic aquí.**](https://github.com/Democrito/I2C_only_write/blob/master/examples/Example_to_send_3_bytes_through_the_serial.ice)

Dejo una imagen de las señales que da [**Pulse View**](https://sigrok.org/wiki/Downloads). Gracias a ese programa podemos verificar cualquier tipo de señal. Si no tienes un analizador lógico puedes usar el módulo [**ICROK**](https://github.com/FPGAwars/iceRok) que se encuentra dentro de Icestudio y hace esa función.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/i2c_signals.PNG)

Ahora enviaremos 3 bytes (dirección + dato + dato). En este ejemplo utilizo una resistencia variable digital que se controla a través del I2C, con el nombre de [**AD5280**](https://groups.google.com/d/msg/fpga-wars-explorando-el-lado-libre/QZqGqehCvuk/L9yCuXW_BwAJ). El byte de dirección es 44 en decimal ('h2C en hexadecimal), luego viene un byte de dato que es de control, pero esto no nos importa ahora (son cuestiones técnicas del chip), para nosotros es un dato que se ha de enviar y finalmente otro dato, que puede ser un valor de 0..255, dependiendo del valor resistivo que queramos poner en la resistencia variable digital, y este valor lo podremos introducir a través del serial.

# Conclusión.

Con este controlador tenemos la flexibilidad de manejar cualquier esclavo I2C de sólo escritura sin importar el número de bytes que se vaya a enviar. Por sencillez sólo he puesto ejemplos donde envían un número de datos fijos (que suele ser lo normal), pero tiene la versatilidad de poder configurarse en cualquier momento el número de bytes para ser enviados. Se puede incluir dentro del diseño memorias RAMs y ROMs para configurar ciertos periféricos complejos, como lo puede ser una pantalla OLED monocromática. Y para finalizar, también es posible manejar más de un esclavo, porque es posible cambiar el ancho de la información a enviar y adaptarse a varios a la vez.
