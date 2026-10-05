# PRACTICA 1: Robot Aspiradora Básica

## Introducción
Esta práctica se basa en programar un robot aspiradora básica de **gama baja (no tiene autolocalización)** , intentando hacer un recorrido eficiente y ocupando el mayor área de limpieza posible de la habitación, de manera que no se choque con obstáculos y evitar tardar mucho.

## Objetivos 
- Principal: Crear un modelo **reactivo** con una maquina de estados
- Secundarios: Hacerlo de manera eficiente, intentando evitar sucesos como: que vuelva por sitios donde ya haya limpiado, que se quede atasco en un mismo lugar, que vaya muy rápido y/o se choque, etc ...

## Resultado
He hecho distintas pruebas y observado el resultado a distintos tiempos, calculando la media de porcentaje limpiado de la habitación 
- 5 Minutos: Aprox. 16%-18%
<img width="1913" height="1007" alt="Screenshot from 2026-10-04 12-35-43" src="https://github.com/user-attachments/assets/268ac382-8101-43da-9b07-a1208f7743ff" />
- 15 Minutos: Aprox. 40%-44%
<img width="1913" height="1007" alt="Screenshot from 2026-10-04 12-35-05" src="https://github.com/user-attachments/assets/e7f51a16-7707-4fa9-a48a-c5f287c8980d" />
- Video Corto:




## Detalles 
Es una máquina de estados, donde comienza en **AVANZAR** el cual va en linea recta, hasta que el láser detecta una distancia menor a la distancia de seguridad, cuando eso ocurre entra al estado **GIRAR** donde he usado la librería **random** donde he randomizado dos parámetros: 

**1. La velocidad angular 
2. El tiempo de giro (en forma de contador)**

dando así lugar a muchas posibilidades para evitar que se quede atascado y recorre caminos distintos. Además de filtrar(evitando las lecturas negativas y NaN) y guardar las distancias del láser.
