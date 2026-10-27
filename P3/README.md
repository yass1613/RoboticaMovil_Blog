# PRACTICA 3: Navegación Local con Campos de Fuerzas Virtuales

## Introducción

Esta práctica se basa en programar un coche en **Unibotics** para que siga los puntos de un circuito y evite los obstáculos que se encuentra por el camino. Para ello he utilizado el algoritmo **VFF (Virtual Force Field)**, que combina una fuerza atractiva hacia el destino y una fuerza repulsiva que aleja al coche de los obstáculos.

La idea es que el coche vaya decidiendo hacia dónde avanzar según el siguiente objetivo y lo que detecta con el láser, intentando seguir el recorrido sin chocarse ni dar vueltas innecesarias.

## Objetivos

- Principal: Implementar la navegación mediante **fuerzas atractivas y repulsivas**.
- Convertir el objetivo del mapa al sistema de coordenadas del coche.
- Utilizar las distancias del láser para evitar los obstáculos, dando más importancia a los cercanos.
- Ajustar el giro y la velocidad para tomar las curvas y continuar hacia los siguientes puntos.
- Evitar comportamientos como dar una vuelta completa, ir en sentido contrario o acercarse demasiado a un obstáculo.

## Resultado

He hecho distintas pruebas cambiando la intensidad de las fuerzas y observando las flechas de la interfaz. Al principio el coche daba vueltas en algunos obstáculos y, en otras pruebas, se chocaba en las curvas.

Después de ajustar la fuerza atractiva para que mantuviese una intensidad constante, en la prueba final el coche consiguió realizar el recorrido correctamente, evitando los obstáculos y sin los cambios de sentido que tenía antes.

**Vídeo corto:**

https://youtube.com/shorts/FDb9y5L2iyI?feature=share

## Detalles

El funcionamiento se repite continuamente: primero obtengo el siguiente objetivo y la posición del coche, después calculo la atracción y la repulsión, y por último utilizo su suma para decidir el movimiento. He usado **HAL** para leer la posición y el láser y mandar las velocidades, **WebGUI** para obtener los objetivos y visualizar las fuerzas, y **Frequency** para regular el bucle a 50 Hz.

### 1. Objetivo y sistema de coordenadas del coche

El destino lo proporciona la práctica mediante `WebGUI.getNextTarget()`. Sus coordenadas son **absolutas**, es decir, están expresadas respecto al mapa.

Para calcular la fuerza atractiva tengo que pasar ese destino a coordenadas **relativas al coche**, como indicó el profesor. En este sistema el coche está en el origen: X positiva es hacia delante e Y positiva es hacia la izquierda.

Primero resto la posición del coche a la del objetivo:

$$
dx = x_{objetivo} - x_{coche}
$$

$$
dy = y_{objetivo} - y_{coche}
$$

Después utilizo la orientación del coche, llamada **yaw**, para girar ese desplazamiento:

$$
x_{rel} = dx\cos(yaw) + dy\sin(yaw)
$$

$$
y_{rel} = -dx\sin(yaw) + dy\cos(yaw)
$$

Así puedo saber dónde está el destino desde el punto de vista del coche. Por ejemplo, un destino relativo `(5, -2)` está 5 metros hacia delante y 2 hacia la derecha.

Para comprobar esta transformación comparé la distancia antes y después del cambio de coordenadas. Las dos coincidían, y también comprobé que la flecha apuntase hacia el objetivo.

### 2. Fuerza atractiva

Esta fuerza apunta desde el coche hacia el destino. Primero utilicé directamente las coordenadas relativas como fuerza, pero después cambié su intensidad para que no dependiese de lo lejos que estuviese el objetivo.

Calculo la distancia:

$$
d = \sqrt{x_{rel}^{2} + y_{rel}^{2}}
$$

Y, cuando la distancia es mayor que cero, divido las dos componentes entre ella y las multiplico por **K_ATRACCION**, que he probado con valor 3:

$$
\vec F_{atractiva} = K_{atraccion}\left(\frac{x_{rel}}{d},\frac{y_{rel}}{d}\right)
$$

Dividir entre la distancia hace que el vector tenga longitud 1, y multiplicar por la constante le da la intensidad que quiero. **Su dirección no cambia**, sigue apuntando hacia el destino.

Guardo aparte las coordenadas relativas originales para mostrar el objetivo en su posición real. La fuerza ajustada representa hacia dónde tirar, pero ya no representa la distancia al destino.

### 3. Fuerza repulsiva

Para detectar los obstáculos utilizo las lecturas del láser, que proporcionan las distancias en metros. Según la distribución de esta práctica, el índice 90 corresponde al frente, el 0 a la derecha y el 179 casi completamente a la izquierda.

Para cada lectura convierto su índice a un ángulo respecto al frente:

$$
\alpha_i = (i-90)\frac{\pi}{180}
$$

Solo tengo en cuenta las distancias **positivas, finitas y menores que la distancia de influencia**, que he usado inicialmente de 3 metros. Así descarto lecturas inválidas y obstáculos demasiado lejanos.

Cada lectura válida aporta una fuerza en sentido contrario al obstáculo:

$$
F_{x,i} = -\frac{\cos(\alpha_i)}{d_i}, \qquad F_{y,i} = -\frac{\sin(\alpha_i)}{d_i}
$$

El signo negativo hace que la fuerza se aleje del obstáculo. Al dividir entre la distancia, **cuanto más cerca está, más empuja**: a 0,5 metros la intensidad es 2, mientras que a 2 metros es 0,5, antes de aplicar el peso.

Voy sumando todas las componentes X por un lado y las Y por otro. Estas sumas empiezan en cero en cada vuelta del bucle para utilizar solo las lecturas actuales. Por último multiplico ambas por **K_REPULSION**, que he usado con valor 0,1, para ajustar cuánto influyen frente a la atracción.

### 4. Fuerza resultante y movimiento

Sumo las dos fuerzas componente a componente:

$$
\vec F_{total} = \vec F_{atractiva} + \vec F_{repulsiva}
$$

La interfaz permite ver la **atracción en verde**, la **repulsión en rojo** y la **resultante en negro**. Estas flechas me han servido para entender por qué el coche giraba hacia un lado u otro.

Para convertir la resultante en una dirección utilizo `atan2(Y, X)`. Esta función devuelve el ángulo del vector respecto al frente del coche: positivo hacia la izquierda y negativo hacia la derecha.

Multiplico ese ángulo por **K_GIRO** y limito la orden entre un máximo positivo y negativo. Para el avance utilizo:

$$
velocidad = \frac{VELOCIDAD}{1 + |angulo|}
$$

Así el coche avanza más rápido cuando la dirección es casi recta y reduce la velocidad cuando necesita girar más. Esta reducción depende del ángulo, no directamente de la distancia frontal al obstáculo.

Si la fuerza resultante tiene una longitud prácticamente nula, detengo el avance y el giro porque no hay una dirección clara. Esta comprobación evita mandar un movimiento sin una dirección definida, aunque por sí sola no resuelve un atasco.

### 5. Cambio de objetivo

Cuando la distancia al destino es menor que **MARGEN_OBJETIVO**, inicialmente de 1 metro, lo marco como alcanzado con `setReached(True)`. En esa iteración mando las velocidades a cero y en la siguiente obtengo el nuevo punto.

De esta forma no es necesario pasar exactamente por las coordenadas del objetivo, pero sí acercarse lo suficiente antes de continuar con el siguiente.

## Errores y soluciones

### 1. Marcar los objetivos antes de llegar

Al principio tenía `setReached(True)` directamente dentro del bucle, sin comprobar ninguna distancia. Eso hacía que fuese marcando los puntos aunque el coche todavía no hubiese llegado.

Lo corregí utilizando el margen de llegada, para marcar cada objetivo solo cuando el coche está suficientemente cerca.

### 2. Transformación incorrecta del objetivo

Inicialmente multiplicaba la diferencia de X por el coseno y la de Y por el seno, pero eso no hacía la transformación completa. Cada componente relativa necesita las dos diferencias, combinadas con seno y coseno.

Corregí las fórmulas y comprobé que la distancia se conservase y que la flecha apuntase hacia el destino.

### 3. El coche daba la vuelta y terminaba en sentido contrario

Este fue uno de los problemas que más pruebas me llevó. La flecha negra apuntaba hacia atrás y el coche intentaba seguirla, llegando a dar una vuelta completa en algunos obstáculos.

Con los mensajes de la consola comprobé un caso en el que la atracción X era aproximadamente **2,707** y la repulsión X era **−2,725**. Al sumarlas quedaba una componente negativa: aunque el objetivo seguía delante, la dirección resultante apuntaba hacia atrás.

El problema era que la atracción original se hacía más pequeña al acercarme al objetivo, mientras la repulsión crecía al acercarme al obstáculo. Además, cuando casi se cancelaban, pequeñas diferencias podían cambiar mucho la dirección.

Primero reduje el peso de la repulsión, pero no fue suficiente. El cambio que mejoró el resultado final fue **normalizar la atracción y darle una intensidad constante**. Así su intensidad deja de depender de la distancia al destino y el equilibrio entre las fuerzas es más consistente.

### 4. No giraba lo suficiente en algunas curvas

También probé a fijar la componente X de la resultante a un valor positivo constante para evitar que apuntase hacia atrás. Sin embargo, en esas pruebas el coche se chocaba en algunas curvas al aumentar la velocidad.

Cambiar solo X puede reducir el ángulo del vector y hacer que el giro sea demasiado suave. Además, con el mismo giro, cuanto más rápido avanza el coche, más abierta es la curva que realiza.

Volví a sumar las componentes originales de las fuerzas y después ajusté la atracción mediante la normalización. No llegué a medir por separado todas las causas de aquel choque, pero con la combinación final el comportamiento mejoró.

### 5. Velocidad demasiado baja y mensajes de depuración

Con una velocidad de 0,5 el coche iba muy lento, y cuando la resultante apuntaba hacia atrás la fórmula del avance la reducía todavía más. Esto me ayudó a ver que no bastaba con subir la velocidad: primero había que corregir la dirección que estaba siguiendo.

Durante las pruebas también imprimía demasiados datos y era difícil leer qué pasaba primero. Añadí una comprobación para mostrar solo el primer aviso, distinguiendo si el objetivo estaba detrás o si la repulsión estaba haciendo que la resultante apuntase hacia atrás.

Una vez comprobado el funcionamiento, esos mensajes se pueden quitar. Las flechas de la interfaz permiten seguir observando las fuerzas sin llenar la consola.

## Conclusión

Lo que más me ha costado ha sido ajustar la relación entre las dos fuerzas. Al principio parecía que el problema era solo de velocidad o de giro, pero al observar los vectores vi que el coche estaba siguiendo una dirección resultante que a veces apuntaba hacia atrás.

Con esta práctica he entendido mejor la importancia de trabajar en el mismo sistema de coordenadas y de controlar la intensidad de las fuerzas. La configuración final me ha funcionado en el recorrido probado, aunque si cambio mucho la velocidad o el escenario tendría que volver a comprobar los ajustes.
