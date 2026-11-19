# PRACTICA 4: Navegación Global con Taxi

## Introducción

Esta práctica se basa en programar un taxi en **Unibotics** para que llegue de forma autónoma al destino que selecciono en el mapa. La idea es que se desplace por las calles de la ciudad, evitando los edificios y manteniendo cierta separación de las paredes.

Para hacerlo he utilizado **GPP (Gradient Path Planning)**. Primero calculo un campo de costes a partir del destino y después el coche va mirando las casillas que tiene alrededor, eligiendo una con menor coste para continuar avanzando.

## Objetivos

- Principal: Implementar la **navegación global mediante un campo de costes**.
- Secundarios: Distinguir las calles de los obstáculos, trabajar correctamente con las coordenadas del mapa y conseguir que el taxi llegue al destino seleccionado.
- Penalizar las zonas cercanas a las paredes para que el coche tienda a circular más separado de ellas.
- Ajustar la velocidad y el giro para avanzar de forma eficiente y evitar acercarse demasiado a las esquinas.
- Elegir en cada iteración entre las **ocho casillas vecinas**, siguiendo las indicaciones del profesor.

## Resultado

He hecho distintas pruebas seleccionando destinos sobre las calles y modificando la velocidad y las penalizaciones. Al principio el coche se movía muy lento y pasaba bastante cerca de las paredes. Después fui aumentando la velocidad y ampliando la zona penalizada de tres a cinco casillas.

Con estos cambios he conseguido que el taxi se dirija al destino y se detenga al entrar en su casilla, con una mejor separación de las paredes en los recorridos probados. La última velocidad de avance que he utilizado es **2.5**.

Al probar un destino lejano que requiere varios giros, he observado que en algunos tramos alterna giros a izquierda y derecha y avanza a pequeños pasos. Ese comportamiento es el ajuste que queda pendiente de comprobar.

**Imagen del mapa y del destino seleccionado**

<img width="353" height="296" alt="Screenshot From 2026-11-19 11-38-16" src="https://github.com/user-attachments/assets/a5e6d061-5405-4e9d-a4f5-96e45cc4a5ba" />


**Vídeo corto del funcionamiento**



https://github.com/user-attachments/assets/b962fe51-c1d3-4b69-a627-8d7e52f48361



## Detalles

He desarrollado la práctica poco a poco. Primero cargué el mapa con el taxi parado, después comprobé las coordenadas y los costes, y por último añadí el control del movimiento.

He utilizado **HAL** para obtener la posición y orientación del taxi y enviar las velocidades, **WebGUI** para cargar el mapa y obtener el destino, y **NumPy** para trabajar con las matrices. El bucle de control utiliza **Frequency.tick(20)**, con una frecuencia objetivo de 20 Hz.

### 1. Mapa y coordenadas

El mapa es una imagen de **400 × 400 píxeles**. Cada píxel representa una casilla de **1.25 × 1.25 metros** en la simulación. Las calles aparecen en blanco y los obstáculos en negro.

Con esta imagen creo una matriz llamada **libre**, donde guardo si cada casilla es transitable. Las celdas con un valor mayor que 127 se consideran libres.

Para localizar al taxi y al destino tengo que relacionar dos sistemas de coordenadas. La simulación proporciona posiciones **X e Y en metros**, mientras que la matriz se consulta mediante **fila y columna**.

Utilizo `WebGUI.worldToGrid()` para convertir las posiciones. En la versión que estoy usando, su primer componente corresponde a la columna y el segundo a la fila. Al consultar la matriz coloco primero la fila y después la columna. También compruebo que los índices estén dentro del mapa antes de acceder a una casilla.

### 2. Cálculo del campo de costes

Al seleccionar un destino válido, creo una matriz del mismo tamaño que el mapa. Inicialmente sus valores son **infinito**, usando `np.inf`, porque todavía no he calculado el coste de llegar desde esas casillas hasta el destino.

La casilla del destino recibe coste **0**. Desde ella comienzo a extender los costes por las calles.

Cada movimiento tiene un coste según su dirección.

- Arriba, abajo, izquierda y derecha añaden **1**.
- Los cuatro movimientos diagonales añaden **√2**, aproximadamente **1.414**, por Pitágoras.

Este coste se suma al que ya tenía la casilla desde la que se está expandiendo. Por ejemplo, si la casilla actual tiene coste 4, un paso recto da 5 y uno diagonal da aproximadamente 5.414, antes de añadir la penalización por paredes.

Para organizar la expansión utilizo una lista de **casillas pendientes**, donde guardo el coste, la fila y la columna. En cada vuelta busco la entrada de menor coste y reviso sus ocho vecinas. Si encuentro un coste mejor para una de ellas, lo actualizo y la añado a la lista. Esta forma de propagar los costes corresponde al algoritmo de **Dijkstra**.

La expansión continúa hasta agotar las casillas pendientes. Las zonas que no están conectadas con el destino conservan el coste infinito. También descarto los movimientos diagonales cuando alguno de los dos lados de la esquina está ocupado.

### 3. Penalización por cercanía a las paredes

Siguiendo la idea del profesor, las casillas libres cercanas a una pared reciben un coste adicional. Cuanto más cerca está el obstáculo, mayor es esa penalización.

Al principio solo añadía 2 si había un obstáculo en alguna de las ocho casillas de alrededor. Como el coche seguía acercándose mucho, fui ampliando esa zona hasta utilizar **cinco niveles**.

| Obstáculo más cercano, medido por anillos de casillas | Penalización |
|---|---:|
| A una casilla | 20 |
| A dos casillas | 12 |
| A tres casillas | 6 |
| A cuatro casillas | 3 |
| A cinco casillas | 1 |
| Más lejos | 0 |

Para calcularlo, reviso cuadrados cada vez más grandes alrededor de cada celda libre, incluyendo las diagonales. Cuando encuentro un obstáculo, guardo la penalización correspondiente y termino la búsqueda de esa celda. Así aplico la penalización del obstáculo más cercano una sola vez.

Guardo estos valores en otra matriz al cargar el mapa, para consultarlos durante la expansión. El cálculo queda de esta forma.

**Coste nuevo = coste actual + coste del movimiento + penalización de la casilla nueva**

Las penalizaciones se acumulan a lo largo de la expansión. Esto favorece los recorridos que pasan más separados de las paredes, aunque supongan avanzar alguna casilla más. Los obstáculos siguen siendo casillas por las que el algoritmo no puede pasar.

### 4. Elección de la siguiente casilla

Una vez calculado el campo, el coche toma una decisión en cada iteración a partir de su posición actual.

1. Obtengo la casilla en la que está el taxi.
2. Miro sus ocho casillas vecinas.
3. Descarto las que están fuera del mapa, tienen obstáculos o requieren atravesar una esquina ocupada.
4. Entre las restantes, elijo la de menor coste, siempre que sea inferior al de la casilla actual.
5. Utilizo esa casilla como objetivo local para el movimiento.

Este proceso se repite continuamente, de manera que **el coche decide el siguiente paso mientras se mueve**. El campo de costes se vuelve a calcular cuando selecciono un destino en otra casilla.

### 5. Giro y avance del taxi

Después de elegir la vecina, utilizo `WebGUI.gridToWorld()` para pasarla a coordenadas del mundo. Con la diferencia entre ese punto y la posición del taxi calculo hacia dónde debe orientarse.

Para obtener esa dirección utilizo `atan2(dy, dx)` y después resto la orientación actual del coche, llamada **yaw**. El resultado es el error de orientación. Lo ajusto al intervalo entre **−π y π** para utilizar el giro de menor magnitud.

Multiplico ese error por **K_GIRO** y limito la velocidad angular. Cuando el error es suficientemente pequeño, permito que avance a la velocidad fijada. Si todavía tiene que orientarse, el avance queda a cero mientras gira.

Los últimos valores probados son los siguientes.

| Parámetro | Valor |
|---|---:|
| Velocidad de avance | 2.5 |
| K_GIRO | 2.0 |
| Giro máximo | 1.5 |
| Margen de orientación | 0.3 radianes |
| Frecuencia objetivo del control | 20 Hz |

El margen de 0.3 radianes equivale aproximadamente a 17 grados. El ángulo permite convertir la elección de la casilla en órdenes de giro y avance para el taxi.

### 6. Llegada y cambio de destino

En cada vuelta del bucle inicializo las velocidades a cero y solo asigno movimiento si encuentro un objetivo local válido. Cuando el taxi entra en la casilla del destino, se detiene.

Si selecciono un nuevo destino, el taxi se para mientras calcula el nuevo campo de costes. También compruebo que el destino sea transitable y que exista conexión con la posición del coche antes de ordenar el movimiento.

## Errores y soluciones

### 1. Las calles aparecían como obstáculos

Al principio, al pulsar en algunas calles el programa decía que había un obstáculo, y en otros puntos ocurría lo contrario. Pensé que podía ser un desplazamiento de la imagen.

Al revisar las coordenadas vi que estaba usando la columna como fila y la fila como columna. Corregí el orden tanto para el taxi como para el destino, usando **fila y columna al acceder a las matrices**.

### 2. La expansión se detenía al empezar

Al sustituir las operaciones de la lista de pendientes, dejé repetida la condición que comprueba si un coste nuevo es menor que el guardado.

Primero actualizaba el coste y después volvía a compararlo con el mismo valor. Como ya eran iguales, la segunda condición nunca se cumplía y no añadía la casilla a la lista. Lo corregí dejando una sola comprobación y añadiendo la vecina justo después de actualizar su coste.

### 3. Errores al colocar los mensajes de comprobación

Durante las pruebas añadí un mensaje que utilizaba la fila del taxi antes de haberla calculado. También dejé un `if` que solo tenía un comentario dentro, lo que provocaba un error de indentación.

Lo solucioné ordenando primero las conversiones y después los mensajes, y eliminando el bloque vacío. Estas comprobaciones me sirvieron para ver si el taxi y el destino estaban en casillas libres.

### 4. El coche pasaba demasiado cerca de las paredes

La primera penalización solo afectaba a las casillas pegadas al obstáculo. Un poco más lejos ya valía cero, por lo que el coche podía elegir puntos que todavía estaban demasiado próximos a la pared.

Primero añadí tres niveles de penalización y después los amplié a cinco. También aumenté el coste de las casillas más cercanas. En las pruebas observé una mejor separación, aunque esta penalización es una preferencia y no establece una distancia mínima obligatoria.

### 5. El avance era demasiado lento

Comencé con una velocidad de 0.5 para comprobar el funcionamiento. Después fui subiéndola a 1.0, 2.0 y finalmente 2.5, ajustando también el giro.

Parte del tiempo se pierde cuando el coche se detiene para orientarse hacia la siguiente casilla. Por eso, además de aumentar la velocidad, he tenido que observar cómo cambia la dirección durante el recorrido.

### 6. Giros alternos en recorridos largos

En la última prueba con un destino lejano vi que el coche avanzaba un poco, giraba a un lado y después corregía hacia el otro. En la consola su fila alternaba entre 199 y 200 mientras se desplazaba por una recta.

Una posible causa es que el punto utilizado para dirigir el coche quede en el borde de la casilla. Cerca de ese límite, pequeños desplazamientos pueden cambiar la celda que detecta el programa y provocar correcciones continuas.

La mejora que queda por probar es **apuntar al centro de la casilla elegida** y comprobar si desaparece esa alternancia. Todavía tengo que validar ese ajuste en el mismo recorrido con varios giros.
