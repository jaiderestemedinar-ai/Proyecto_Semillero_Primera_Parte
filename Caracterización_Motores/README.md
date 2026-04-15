### ⚙️ VOLUMEN 1: Caracterización de Motores GM37 y Encoders Magnéticos

Este documento detalla el análisis físico, la corrección de especificaciones de fábrica y la formulación matemática de los actuadores principales utilizados en el chasis omnidireccional del Proyecto Semillero . Comprender la verdadera naturaleza de estos motores fue el primer paso crítico para diseñar el sistema de control PID del que se hablará en otro apartado.

# 1. El Conflicto de la Etiqueta: Especificaciones y como venian en realidad
El chasis está impulsado por 4 motores reductores DC modelo GM37-520. Según la etiqueta del fabricante pegada en la carcasa, las especificaciones dictaban:

![Descripción de la foto para el sistema](Motor_Mecanum.jpg)

Voltaje Nominal: 12V

Velocidad Impresa: 110 RPM

Caja Reductora (Gear Ratio): 1:90

Encoder: 12 PPR (Pulsos Por Revolución) / AB Phase (Cuadratura).


# 1.1. La Prueba Empírica: Descubriendo los 300 RPM
Durante las fases iniciales de pruebas a lazo abierto (inyectando 12V directos mediante PWM al 100%), observé que el chasis se movía a una velocidad inusualmente alta para ser de "110 RPM".

Al medir la velocidad real con el microcontrolador, descubrí que los motores alcanzaban ~300 RPM en vacío, casi el triple de lo que indicaba la etiqueta.

Conclusión : Los motores fueron ensamblados o etiquetados incorrectamente en fábrica (posiblemente equipados con un motor interno de mayor velocidad base). Este descubrimiento fue crucial, ya que si hubiéra diseñado el controlador PID esperando un máximo de 110 RPM, el sistema se habría vuelto inestable al saturarse físicamente. El límite oficial de diseño del sistema se fijó empíricamente en 300 RPM.

![Prueba en serial monitor](Prueba-RPM_motores.png)

Aquí tambien queda en evidencia el codigo empleado para saber los RPM por individual de cada motor:

/*
 * =======================================================================
 * PRUEBA EMPÍRICA: CARACTERIZACIÓN DE VELOCIDAD TERMINAL (LAZO ABIERTO)
 * Microcontrolador: PIC18F45K22 a 16MHz
 * Objetivo: Medir RPM máximas reales inyectando PWM al 100% (255)
 * =======================================================================
 */

#include <xc.h>
#include <stdint.h>
#include <stdio.h>

// Configuración de Fusibles: Oscilador Interno, Watchdog OFF, LVP OFF
#pragma config FOSC = INTIO67, WDTEN = OFF, LVP = OFF, MCLRE = INTMCLR  
#define _XTAL_FREQ 16000000     

// --- VARIABLES GLOBALES (ODOMETRÍA) ---
volatile long contador_encoder = 0; 
long posicion_anterior = 0;
volatile float rpm_actual = 0.0;
volatile char flag_muestreo = 0;
volatile char ultimo_estado_A = 0;

// ==========================================
// CONFIGURACIÓN DE MÓDULOS DE HARDWARE
// ==========================================
void UART1_Init(void) {
    SPBRG1 = 25; // 9600 Baudios
    TRISCbits.TRISC6 = 0; 
    TRISCbits.TRISC7 = 1; 
    TXSTA1bits.SYNC = 0; 
    TXSTA1bits.BRGH = 0; 
    RCSTA1bits.SPEN = 1; 
    TXSTA1bits.TXEN = 1; 
    RCSTA1bits.CREN = 1; 
}

void UART1_Print(const char* text) { 
    for(int i=0; text[i] != '\0'; i++) { 
        while(!TXSTA1bits.TRMT); 
        TXREG1 = text[i]; 
    } 
}

void Timers_Init(void) {
    T1CON = 0x31;               // 16-bit, Prescaler 1:8, Timer1 ON
    TMR1H = 0x9E; TMR1L = 0x58; // Carga para desbordamiento exacto a 50ms (16MHz)
    PIE1bits.TMR1IE = 1;    
}

void PWM_Init(void) {
    TRISCbits.TRISC2 = 0;   // Pin CCP1 como salida 
    PR2 = 255;              // Periodo PWM
    CCP1CON = 0x0C;         // Modo PWM
    T2CON = 0x05;           // Timer2 ON, Prescaler 1:4
    CCPR1L = 0;             // Duty Cycle inicial (0%)
}

// ==========================================
// RUTINA DE INTERRUPCIONES (NÚCLEO CRÍTICO)
// ==========================================
void __interrupt() ISR(void) {
    
    // 1. LECTURA DEL ENCODER (Interrupt On Change)
    if(INTCONbits.RBIF) {
        char estado_A = PORTBbits.RB4; 
        char estado_B = PORTBbits.RB5; 
        
        // Detección de flanco de subida en Canal A
        if(estado_A == 1 && ultimo_estado_A == 0) {
            // Lectura de cuadratura (Dirección)
            if(estado_B == 0) {
                contador_encoder--; 
            } else {
                contador_encoder++; 
            }
        }
        ultimo_estado_A = estado_A;
        
        char dummy = PORTB; // Lectura para limpiar mismatch
        INTCONbits.RBIF = 0;  
    }

    // 2. CÁLCULO DE VELOCIDAD (Timer1: Cada 50ms)
    if(PIR1bits.TMR1IF) {
        long delta_clicks = contador_encoder - posicion_anterior;
        posicion_anterior = contador_encoder;
        
        // Multiplicador 1.11 validado matemáticamente (1080 PPR @ 50ms)
        rpm_actual = (float)delta_clicks * 1.11; 
        
        // ---------------------------------------------------------
        // PRUEBA DE VELOCIDAD TERMINAL: INYECCIÓN DIRECTA DE ENERGÍA
        // ---------------------------------------------------------
        CCPR1L = 255; // Forzar PWM al 100% (Ignorando PID)
        
        flag_muestreo = 1;  
        TMR1H = 0x9E; TMR1L = 0x58; // Recargar Timer para próximos 50ms
        PIR1bits.TMR1IF = 0; 
    }
}

// ==========================================
// BUCLE PRINCIPAL
// ==========================================
void main(void) {
    // Configuración de Reloj y Pines
    OSCCON = 0x72; // 16 MHz
    ANSELB = 0; 
    
    // Pines de Dirección (IN1, IN2)
    TRISBbits.TRISB0 = 0; 
    TRISBbits.TRISB1 = 0; 
    
    // Pines del Encoder (Canal A, Canal B)
    TRISBbits.TRISB4 = 1; 
    TRISBbits.TRISB5 = 1; 
    
    // Forzar dirección: Adelante
    LATBbits.LATB0 = 1; 
    LATBbits.LATB1 = 0; 
    
    // Inicializar Periféricos
    UART1_Init(); 
    PWM_Init(); 
    Timers_Init();
    
    // Configurar Interrupciones
    IOCBbits.IOCB4 = 1; // Habilitar IOC en RB4 (Canal A)
    char dummy = PORTB; 
    INTCONbits.RBIF = 0;  
    INTCONbits.RBIE = 1; // Globales de Periféricos
    INTCONbits.PEIE = 1; 
    INTCONbits.GIE = 1;   
    
    char mensaje[60];
    
    // Retardo inicial de seguridad
    __delay_ms(1000);
    UART1_Print("\n--- PRUEBA DE VELOCIDAD TERMINAL (MAX RPM) ---\r\n");
    UART1_Print("Advertencia: Inyectando PWM 255 constante...\r\n\n");

    while(1) {
        // Imprimir telemetría solo cuando hay nuevos datos (Cada 50ms)
        if(flag_muestreo) {
            sprintf(mensaje, "Real: %.1f RPM | PWM: 255 (AL MAXIMO)\r\n", (double)rpm_actual);
            UART1_Print(mensaje);
            flag_muestreo = 0; 
        }
    }
}


## 2. Sistema de Odometría: Encoders y PPR
Para medir la velocidad y dirección real, cada motor cuenta con un Encoder Magnético de Efecto Hall en la cola del motor principal (antes de los engranajes reductores).

## 2.1. Entendiendo el "12 PPR" de la Etiqueta
La etiqueta menciona "12 PPR / AB Phase". Esto significa que el disco magnético pegado al motor interno genera 12 pulsos eléctricos por cada vuelta que da el motor pequeño.

Sin embargo, no me interesa cuántas vueltas da el motor interno; me interesa cuántas vueltas da la llanta (el eje de salida).

## 2.2. Cálculo del PPR Real (Eje de Salida)
Aquí es donde entra la relación de la caja reductora (1:90). Esto significa que el motor interno debe girar 90 veces para que el eje principal (la llanta) complete 1 sola vuelta.

Por lo tanto, la resolución real del sistema para la llanta se calcula multiplicando los pulsos internos por la reducción:

PPR del Sensor = 12

Relación de Engranajes = 90

PPR_Total (Llantas) = 12 * 90 = 1080 Pulsos Por Revolución

¿Qué significa esto? Que por cada vuelta completa de la llanta sobre el piso, el PIC18F45K22 recibe exactamente 1080 interrupciones eléctricas. Es una resolución altísima que permite un control muy fino a bajas velocidades.

## 3. Matemática del Microcontrolador: La Fórmula del 1.11
Para que el control PID funcione, debe comparar el "Setpoint" (la velocidad que deseamos) con la "Velocidad Real". Ambas deben estar en la misma unidad de ingeniería: Revoluciones Por Minuto (RPM).

## 3.1. El Problema del Tiempo
El PIC18F45K22 utiliza un Timer que interrumpe el programa exactamente cada 50 milisegundos (ms). En esos 50ms, el PIC cuenta cuántos pulsos generó la llanta. A esa cantidad de pulsos le llamamos Delta.

El reto es convertir rápidamente esa variable Delta (Pulsos en 50ms) a RPM sin ahogar al procesador de 8 bits con operaciones de punto flotante o divisiones complejas.

## 3.2. Desmenuzando la Fórmula de Conversión
Quiero llegar de: [Pulsos / 50ms] a [Vueltas / Minuto].

Paso 1: Convertir Pulsos a Vueltas (Revoluciones)
Se sabe que 1 vuelta son 1080 pulsos.

Vueltas = Delta / 1080

Paso 2: Convertir 50ms a Minutos
Se sabe que 1 minuto tiene 60,000 milisegundos.

¿Cuántas veces cabe el ciclo de 50ms en un minuto?

Ciclos por Minuto = 60,000 / 50 = 1200 ciclos.

Paso 3: Unir las Fórmulas
Si se multiplica las vueltas que dio en un ciclo, por la cantidad de ciclos que hay en un minuto, obtenemos las Vueltas Por Minuto (RPM).

RPM = (Delta / 1080) * 1200

Paso 4: Simplificación Matemática (La Constante necesaria)
Para ahorrarle trabajo al microcontrolador, resolvemos las constantes estáticas (1200 / 1080) por nuestra cuenta:

1200 / 1080 = 1.11111...

La Ecuación Final del Sistema:

C

int rpm_actual = (int)(delta * 1.11);

Resumen: Esta única  línea de código en C es el corazón sensorial del chasis. Multiplicando los pulsos leídos por 1.11, el PIC deduce instantáneamente a qué velocidad en RPM va el carro cada 50 milisegundos, permitiendo que el PID reaccione casi en tiempo real a la fricción o cambios en el terreno.