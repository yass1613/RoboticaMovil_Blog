# PRACTICA 2: Sigue Líneas con Control PID

## Introducción

Esta práctica se basa en programar un coche de Fórmula 1 para que siga la **línea roja de un circuito utilizando la cámara**. La idea es que consiga dar vueltas de manera autónoma, intentando completar el recorrido en el menor tiempo posible, pero sin perder la línea ni salirse en las curvas.

## Objetivos

- Principal: Crear un **controlador PID** que permita seguir la línea roja.
- Secundarios: Detectar correctamente la línea, reducir las oscilaciones y ajustar la velocidad para completar el circuito lo más rápido posible sin perder el control.

## Resultado

He hecho distintas pruebas, modificando las constantes del controlador y aumentando poco a poco la velocidad. Al principio el coche conseguía seguir la línea, pero tardaba algo en corregir. Al aumentar la corrección y la velocidad aparecieron oscilaciones y, en algunas pruebas, se salía por fuera de una curva.

Después de los ajustes, he conseguido que dé **varias vueltas seguidas sin salirse a una velocidad constante de 10.2**. Las constantes utilizadas finalmente son:

- **Kp = 0.0075**
- **Ki = 0.00001**
- **Kd = 0.007**
- **Límite de la integral = 1000**

También probé con una velocidad de **10.3**, pero perdió la línea, por lo que volví a **10.2** y comprobé que completaba varias vueltas correctamente.

**Imagen del circuito:**

<img width="332" height="186" alt="image" src="https://github.com/user-attachments/assets/dd5933b9-acda-4cab-9619-ac03f60e5963" />


**Detección de la línea:**

<img width="961" height="275" alt="image" src="https://github.com/user-attachments/assets/9718f372-d559-4d1d-9c0c-6a3b28f70b28" />


**Vídeo corto:**



https://github.com/user-attachments/assets/6601948b-3e60-4b4e-b860-44d91b69aa86



## Detalles

He desarrollado la práctica poco a poco, comprobando primero la detección de la línea con el coche parado y añadiendo después el controlador.

### Detección de la línea

Primero obtengo la imagen de la cámara y la convierto de **BGR a HSV** con la librería **OpenCV**, para poder filtrar el color rojo.

He utilizado **dos rangos de rojo**, porque este color aparece en los dos extremos del tono en HSV. Después uno los resultados para crear una **máscara**, donde los píxeles detectados aparecen en blanco y el resto en negro. Así pude comprobar visualmente si el filtro estaba funcionando antes de mover el coche.

Para calcular la posición de la línea, analizo solamente una **franja situada entre el 60% y el 85% de la altura de la imagen**. Dentro de esa franja calculo el centro de los píxeles blancos mediante los momentos de la imagen.

### Cálculo del error

Una vez localizada la línea, calculo la diferencia entre el centro horizontal de la imagen y el centro de la línea detectada:

**Error = centro de la imagen − centro de la línea**

Si el error es positivo, la línea está a la izquierda; si es negativo, está a la derecha. Cuando está cerca de cero, la línea está aproximadamente centrada en la franja.

Este error es el que utilizo para calcular la velocidad angular del coche.

### Controlador PID

Primero implementé solamente el **controlador P** y después añadí la parte derivativa y la integral.

**1. Parte proporcional (P)**

Multiplica el error actual por **Kp**. Cuanto más lejos esté la línea del centro de la imagen, mayor será la corrección. Durante las pruebas vi que aumentar este valor ayudaba a corregir antes, pero también podía producir más oscilaciones.

**2. Parte derivativa (D)**

Calcula la diferencia entre el error actual y el de la iteración anterior. Siguiendo la explicación del profesor, lo he hecho **por iteraciones, sin dividir entre un tiempo `dt`**.

Esta parte tiene en cuenta cómo está cambiando el error y ayuda a amortiguar las oscilaciones cuando está bien ajustada. En la primera iteración vale cero, porque todavía no hay un error anterior.

**3. Parte integral (I)**

Acumula los errores de las distintas iteraciones **con su signo**, para compensar desviaciones que se mantienen durante el recorrido.

He limitado esta suma entre **−1000 y 1000**, evitando que crezca indefinidamente. Con el valor de Ki que utilizo, su aportación es pequeña.

Finalmente, sumo las tres partes para obtener el giro:

**Giro = Kp × error + Ki × integral + Kd × derivada**

El coche avanza a una **velocidad constante**, mientras que el controlador va modificando la velocidad angular para seguir la línea.

### Ajuste y pruebas

Para ajustar el comportamiento fui haciendo pruebas con distintas ganancias y velocidades. A velocidad **7** seguía la línea, pero la corrección parecía algo lenta. Después fui aumentando la parte proporcional y probando velocidades mayores.

Una vez encontré unas ganancias que funcionaban bien, las mantuve y aumenté la velocidad poco a poco. Con cada cambio comprobaba si mejoraba el tiempo y si conseguía completar varias vueltas sin perder la línea.

Finalmente elegí **10.2**, ya que fue la mayor velocidad probada que completó varias vueltas correctamente con esta configuración. No significa que sea el máximo posible con otros ajustes, pero sí fue el mejor resultado de mis pruebas. T

### Pérdida de la línea

Si no se detecta la línea dentro de la franja o no llega una imagen válida, el coche **se detiene**. Además, reinicio la integral y el error anterior para no reutilizar valores antiguos cuando vuelva a detectar la línea.

El bucle utiliza **Frequency.tick(50)** para regular la ejecución a una frecuencia objetivo de 50 Hz.
