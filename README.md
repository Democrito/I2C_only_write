# I2C_only_write.

*La intención de este tutorial es enseñarte a ser capaz de manejar cualquier periferico I2C de sólo escritura a través del programa de diseño electrónico [**Icestudio**](https://github.com/FPGAwars/icestudio) utilizando como FPGA la [**Alhambra II**](https://alhambrabits.com/alhambra/).*

# Un breve repaso al I2C.
### Conceptos básicos:

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/croquis_general_i2c.PNG)

Hoy en día existen dos niveles de voltaje que puede ser de 5v o de 3.3v. Cuando sucede que el maestro y el esclavo tienen distintos niveles de voltaje entonces es necesario adaptar las tensiones para que no haya problemas de comunicación, y se hace a través de adaptadores de tensión bidireccional.
**Nota:** Uso la palabra *tensión* y *voltaje* indistintamente, pero es lo mismo.

![](https://github.com/Democrito/I2C_only_write/blob/master/IMG/adaptador_de_niveles_de_tensi%C3%B3n_33_5.PNG)

Los hay de 8 bits y de 4 bits. Como el I2C sólo tiene 2 líneas (SDA y SCL) entonces usaremos (sólo si fuese necesario) dos bits del adaptador de tensión bidireccional. Es frecuente que a pesar de que maestro y esclavo se alimenten con tensiones diferentes se puedan tolerar entre ambos, es decir, que el de 5v acepte señales de 3.3v y que el de 3.3v tolere líneas de comunicaciónes de 5v. Si ese fuese el caso no necesitaríamos el adaptador y para asegurarte has de mirar el datasheet del periférico que vas a utilizar, y si no menciona nada sobre esto, has de mirar a partir de qué voltaje se considera que un 0 es cero y que un 1 es uno (si las líneas son [*trigger Schmitt*](https://es.wikipedia.org/wiki/Disparador_Schmitt) has de mirar en esos datos.

El valor de las dos resistencias pull-up (Rpp) no son críticas. De forma estandar se usa un valor de 4k7, pero puedes usar valores un poco mayores o menores y seguirá funcionando bien. La velocidad de comunicación más comunes son de 100KHz y de 400KHz, cuanto mayor es la velocidad se recomienda bajar el valor resistivo para que sea más inmune al ruido eléctrico ambiental. Nosotros vamos a utilizar una frecuencia más bien baja, como mucho 400KHz, entonces esta parte no es crítica y una resistencia de 4k7 o de 5k nos irá perfecto, y si fuese de 3k3 o de 1k, irá igual de bien. Existen protocolos de I2C que puede llegar a funcionar a 1MHz, pero nosotros nos ceñiremos a la gran mayoría de periféricos



