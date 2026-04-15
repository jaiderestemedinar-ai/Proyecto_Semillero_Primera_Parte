# 🧠 Subsistema PID: Análisis, Pruebas y Lecciones Aprendidas

Este documento profundiza en el diseño, sintonización y la bitácora de depuración del Controlador Proporcional-Integral (PI) implementado en el PIC18F45K22 para el control de tracción del chasis Mecanum. 

Se documenta explícitamente la transición desde las pruebas a lazo abierto hasta la estabilidad final, haciendo especial énfasis en los fallos técnicos, errores de implementación y las soluciones de arquitectura de software aplicadas.

## Terminos a tener en cuenta

KP, KI y KD son las ganancias (multiplicadores) del controlador PID (Proporcional, Integral, Derivativo). Su trabajo es observar el Error (la diferencia entre la velocidad que quieres y la velocidad real que lee el encoder) y decidir cuánta energía (PWM) enviarle al motor para corregirlo.
La ley de control matemática formal se expresa así:

# 1. KP (Constante Proporcional): El MúsculoEl término proporcional reacciona al presente. Analiza el error exacto que hay en este instante y lo multiplica por la constante Kp.

**¿Cómo actúa?** Si le pides al chasis ir a 150 RPM y está detenido en 0 RPM, el error es enorme (150). El Kp inyecta un "golpe" masivo de PWM para intentar cerrar esa brecha rápidamente. A medida que el motor acelera y llega a 140 RPM, el error se reduce a 10, por lo que el Kp empuja con mucha menos fuerza.

**¿Qué pasa si es muy bajo?** El carro será muy lento para reaccionar y se sentirá "perezoso".

**¿Qué pasa si es muy alto?** Ocurre el fallo de los "tics" mecánicos que experimentaste. El motor acelera tan rápido que se pasa de la meta (Overshoot), luego frena de golpe, se vuelve a pasar hacia abajo, y se queda oscilando y golpeando la caja reductora.

# 2. KI (Constante Integral): La Memoria
El término integral reacciona al pasado. Su trabajo es sumar (acumular) el error a lo largo del tiempo.

**¿Cómo actúa?**Imagina que el motor llega a 145 RPM, pero el peso del brazo robótico y la fricción del suelo son tan fuertes que el $K_p$ por sí solo no tiene la fuerza para empujar esos 5 RPM que faltan. Ese error se llama Error en Estado Estacionario. Aquí entra el $K_i$: empieza a sumar ese pequeño error de 5 RPM ciclo tras ciclo (5, 10, 15, 20...). Al acumularse, la orden de PWM crece lentamente hasta que "rompe" la fricción y el motor alcanza exactamente las 150 RPM.

**¿Qué pasa si es muy alto?** El sistema acumula error demasiado rápido y causa inestabilidad.

El peligro (Windup): Si chocas contra una pared, el error nunca llegará a cero, y el Ki sumará hasta el infinito. Por eso le pusimos un límite por software (de -700 a +700) para protegerlo.

# 3. KD (Constante Derivativa): El Oráculo (El Freno)
El término derivativo reacciona al futuro intentando predecir qué va a pasar analizando la tasa de cambio del error (qué tan rápido nos acercamos a la meta).

**¿Cómo actúa?** Si el motor está acelerando como un cohete hacia las 150 RPM, el $K_d$ nota que la velocidad a la que se reduce el error es altísima. Su trabajo es actuar como un amortiguador o "paracaídas", restando un poco de potencia antes de llegar a la meta para que el motor frene suavemente justo en el objetivo sin pasarse.

¿Por qué la eliminamos de tu código (PI vs PID)? Porque la derivada es extremadamente sensible al ruido. Tus encoders magnéticos de 1080 PPR leídos cada 50ms pueden tener micro-fluctuations (leer 40 pulsos un ciclo, 41 el siguiente, 39 el siguiente). Si calculas la derivada de ese ruido, la matemática genera "picos" violentos de energía. El Kd volvería locos a los motores haciéndolos vibrar de forma agresiva. Para un chasis pesado, un control PI bien sintonizado (solo Proporcional e Integral) es mucho más estable.

## 1. Topología del Controlador (PI vs PID)

El objetivo del sistema es mantener una velocidad de giro constante en cada una de las 4 llantas, independientemente de la carga, la fricción del suelo o las variaciones de voltaje de la batería. 

Se optó por una topología **Proporcional-Integral (PI)**, descartando el componente Derivativo (Kd). 
* **¿Por qué sin Derivativo?** La derivada reacciona a la tasa de cambio del error. Al usar encoders magnéticos de alta resolución (1080 PPR) leídos cada 50ms, la señal tiene un ruido inherente (pequeñas fluctuaciones de 1 o 2 pulsos). Un término derivativo amplificaría este ruido a alta frecuencia, causando que los motores "vibren" intentando compensar micro-errores inexistentes.

La ley de control base es:
`Salida PWM = (Kp * Error) + (Ki * Integral)`

## 2. Bitácora de Pruebas y Fallos de Implementación

El camino hacia la estabilidad requirió resolver múltiples desafíos que mezclaron problemas de lógica, limitaciones de hardware y errores de enrutamiento eléctrico. A continuación, se detallan los fallos más críticos documentados durante las pruebas:

### ❌ Fallo 1: Motores "Desbocados" y Feedback Positivo
**El Problema:** Durante las primeras pruebas de lazo cerrado, al darle una orden de avanzar, el motor Frontal Izquierdo (FL) aceleraba sin control hasta el 100% de PWM, ignorando el Setpoint, mientras los demás se comportaban normal.
**El Diagnóstico:** La polaridad del encoder de ese motor estaba invertida respecto a su puente H. En lugar de restar el error, el sistema lo sumaba. El PID, al ver que estaba "lejos" de la meta, inyectaba más energía, lo que aumentaba el error aún más (creando un lazo de realimentación positiva).
**La Solución:** Se implementó una corrección de fase por software (Software-based mirroring), invirtiendo la lógica de suma/resta en la rutina de interrupción del encoder específicamente para esa llanta, alineando su vector de dirección lógico con el físico.

### ❌ Fallo 2: Oscilaciones Mecánicas Acústicas ("Tics")
**El Problema:** Al enviar comandos de velocidad media, el chasis avanzaba pero producía un sonido de "tic" mecánico repetitivo, como si la caja reductora se estuviera golpeando internamente.
**El Diagnóstico:** Sintonización agresiva. La ganancia Proporcional (Kp) era demasiado alta. El PIC sobre-corregía la velocidad en cada ciclo de 50ms, pasando de tener "muy poca" a "demasiada" energía constantemente, golpeando los engranajes de la relación 1:90.
**La Solución:** Se redujo el Kp drásticamente y se suavizó la curva de corrección introduciendo el Kickstart de 80ms (explicado en la documentación cinemática) para evitar que el PID tuviera que pelear contra la inercia estática inicial.

### ❌ Fallo 3: Motor Muerto por Asignación de Pines
**El Problema:** El motor Trasero Derecho (RR) no respondía bajo ninguna circunstancia, lo que inicialmente se atribuyó a un fallo en el algoritmo PID o un puente H quemado.
**El Diagnóstico:** Tras realizar pruebas de continuidad eléctrica exhaustivas sin encontrar fallas, la revisión de registros reveló un conflicto en la asignación de hardware. Los pines designados chocaban con otras funciones del microcontrolador o no estaban correctamente configurados como salidas en el registro `TRIS`.
**La Solución:** Se reasignaron los pines `DIR_RR_IN3` y `DIR_RR_IN4` a `LATD6` y `LATD7`, y se limpiaron exhaustivamente los registros analógicos (`ANSELD = 0`).

### ❌ Fallo 4: Corrupción de Datos (Acceso No Atómico a Variables)
**El Problema:** Esporádicamente, el chasis daba tirones violentos sin explicación, como si el encoder hubiera leído 10,000 pulsos de golpe en un solo ciclo.
**El Diagnóstico:** Un error clásico pero avanzado de sistemas embebidos de 8 bits. La variable `contador_encoder` es un `long` (32 bits). El PIC18F45K22 (8 bits) necesita 4 ciclos de reloj para leerla completa. Si el PIC estaba leyendo la variable en el `main` y, justo en medio de esa lectura, una interrupción del encoder disparaba y modificaba la variable, los datos leídos quedaban corruptos.
**La Solución:** Se implementó protección de memoria desactivando momentáneamente las interrupciones globales (`GIE = 0`) solo durante la fracción de microsegundo en que se copia la variable al entorno seguro del cálculo PID.

```c
// Fragmento de Protección de Memoria (Acceso Atómico)
INTCONbits.GIE = 0; // Apagar interrupciones (Bloquear la puerta)
long c[4] = {contador[0], contador[1], contador[2], contador[3]}; // Copia rápida
INTCONbits.GIE = 1; // Encender interrupciones (Reabrir la puerta)

// Ahora el cálculo se hace usando la copia segura (c[i])
long delta = c[i] - pos_anterior[i];