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
