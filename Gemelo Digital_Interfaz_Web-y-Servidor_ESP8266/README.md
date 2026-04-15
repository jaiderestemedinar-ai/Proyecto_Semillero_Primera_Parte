## 🌐 VOLUMEN 5: Gemelo Digital, Interfaz Web y Servidor ESP8266
Este documento describe la arquitectura de software de alto nivel del Proyecto Semillero. Detalla cómo se diseñó una interfaz gráfica HMI (Human-Machine Interface) capaz de renderizar modelos 3D y enviar comandos en tiempo real, utilizando un módulo ESP8266 como servidor local.

## 1. Arquitectura de Red (El Servidor ESP8266)
Para que el proyecto fuera 100% autónomo y no dependiera de redes de internet externas, el microcontrolador ESP8266 NodeMCU fue configurado como un Access Point (Punto de Acceso SoftAP).

El ESP8266 crea su propia red Wi-Fi llamada Robot_Mecanum. Al conectarse a esta red, el usuario interactúa con la página web almacenada y servida a través del protocolo WebSockets (Puerto 81).

## 1.1. El Filtro Anti-Spam (Código ESP8266)
Para asegurar que los comandos lleguen intactos al PIC1 sin saturar la red (colisión de paquetes), se implementó un filtro de "Ráfagas" en el ESP8266.

```
// Fragmento del filtro en ESP8266
if (c != ultimoComando) {
    Serial.write(c); // Envia la orden nueva
    ultimoComando = c;
    contadorRafa = 1;
} 
else if (contadorRafa < 5) {
    Serial.write(c); // Deja pasar hasta 5 letras idénticas (Ráfaga de seguridad)
    contadorRafa++;
}
// Las letras subsecuentes iguales se descartan para evitar saturación UART
```
Adicionalmente, el ESP8266 se encarga de escuchar la telemetría del PIC (como la velocidad o el radar) esperando el cierre de un formato JSON (}) para enviarlo a la Interfaz Web mediante la función broadcastTXT().

## 2. La Interfaz Web (HMI)
El archivo index.html es una aplicación web de una sola página (SPA) que consolida los controles físicos y la visualización virtual. Su diseño se inspira fuertemente en interfaces de control industrial (ej. Epson RC+).

## 2.1. Escudo de Controles (Pointer Events)
Para garantizar una experiencia de teleoperación industrial de tipo "Hombre Muerto" (el robot solo se mueve mientras el botón está activado físicamente), se descartó el uso de los eventos tradicionales mousedown o touchstart.

En su lugar, se implementó la API moderna Pointer Events, la cual unifica clics de ratón y toques de pantalla táctil, capturando el "puntero" para asegurar que, si el dedo resbala fuera del botón, el sistema mande inmediatamente una ráfaga de frenado (SSSSS).

```
// Captura de eventos para máxima fiabilidad
const onEnd = (e) => { 
    e.preventDefault(); 
    btn.releasePointerCapture(e.pointerId); 
    if(activeButtons[id]) { 
        activeButtons[id] = false; 
        sendCommand(releaseCmd.repeat(5)); // Ráfaga de seguridad de PARADA
    } 
}; 
btn.addEventListener('pointerup', onEnd); 
btn.addEventListener('pointercancel', onEnd); 
btn.addEventListener('pointerleave', onEnd);
```
## 3. El Gemelo Digital (URDF + Three.js)
La característica más avanzada de la Interfaz Web es la representación virtual del chasis sincronizada en tiempo real. Esto se logró importando una librería gráfica y un parseador de modelos robóticos.

![Prueba en serial monitor](urdf_en_la_interfaz.png)

## 3.1. ¿Cómo se renderiza el URDF?
Se importó Three.js (un motor 3D de WebGL) junto con urdf-loader.
El archivo carrosemillero2.urdf (Unified Robot Description Format) contiene la cinemática, los pesos y las mallas visuales de los enlaces (chasis, brazos, rodillos) diseñadas originalmente en SolidWorks.

```
// Carga del modelo en la escena de Three.js
const loader = new URDFLoader(); 
loader.load('carrosemillero2.urdf', result => { 
    robot = result; 
    scene.add(robot); 
    robot.rotation.x = -Math.PI / 2; // Ajuste de ejes Z-up a Y-up
});
```

## 3.2. Sincronización Físico-Virtual
Cuando el usuario oprime un botón, el PIC1 envía un JSON al ESP8266 confirmando que las llantas están girando (ej. {"v":0.1}).
La función animate() de Three.js intercepta este valor y lo aplica a las articulaciones (joints) definidas en el URDF, haciendo que el modelo 3D gire a la par del chasis físico.

```
// Bucle maestro de renderizado 
function animate() { 
    requestAnimationFrame(animate); 
    if (robot) { 
        // Identifica los enlaces de las 4 llantas y les suma la velocidad recibida
        const joints = { fl: 'rdi', fr: 'rdd', rl: 'rti', rr: 'rtd' }; 
        for (const key in joints) {
            if (robot.joints[joints[key]]) {
                robot.joints[joints[key]].setJointValue(robot.joints[joints[key]].angle + wheelVelocity); 
            }
        }
    } 
    renderer.render(scene, camera); 
}
```
## 3.3. Escalabilidad: Cambiar el URDF a futuro
El sistema es modular. Si en el futuro se rediseña el chasis o se cambia la pinza del brazo robótico, no es necesario reescribir la página web. Solo se deben seguir estos pasos:

Exportar el nuevo diseño desde SolidWorks usando el plugin URDF Exporter.

Guardar el nuevo archivo como nuevo_modelo.urdf en la raíz del servidor.

Cambiar la línea loader.load('carrosemillero2.urdf', ...) por el nuevo nombre del archivo.

(Opcional): Si los nombres de los motores/articulaciones en SolidWorks cambiaron (ej. de rdi a llanta_frontal), actualizar el diccionario const joints en la función de animación.

## 4. Visualización Sensorial (Radar Canvas)
Finalmente, se integró un lienzo HTML5 (<canvas>) superpuesto a la vista 3D para renderizar el radar ultrasónico.
La función drawRadar() procesa los paquetes JSON {"a":40,"d":25} convirtiendo coordenadas polares (Ángulo y Distancia) en coordenadas cartesianas (X, Y) mediante trigonometría (Math.cos y Math.sin), dibujando la posición de los obstáculos en tiempo real y cambiando su color a rojo si infringen el umbral de seguridad de 7cm.

![radar](deteccion_de_objeto.png)