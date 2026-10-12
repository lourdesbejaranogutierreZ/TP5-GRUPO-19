# TP4 — Cronómetro Digital y Comunicacion en serie

Comisión / Docente: Blasco Marcos - Comisión 4.2
Grupo N°: 19
Scrum Master: Valentin

👥Desarrolladores:
Bejarano Lourdes
Leyria Guillermo
Tablada Rocio

---

## Consigna

Diseñar e implementar, sobre el microcontrolador **PIC16F887**, un cronómetro digital que cuenta en el rango **00.00 a 99.99 segundos** (formato SS.CC — segundos y centésimas), con una resolución de **10 ms** en el dígito menos significativo.


Ademas, el usuario deberá poder ingresar un valor de cuatro dígitos, ya sea mediante el teclado matricial o desde la computadora con un programa terminal, como Hercules. El microcontrolador recibirá el valor por UART, lo mostrará en el display con el formato SS.CC y permitirá iniciar o pausar el cronómetro con los controles existentes.

### Hardware

- **4 displays de 7 segmentos**, multiplexados, que muestran el valor SS.CC.
- **2 pulsadores** de control:
  - **Pulsador de control (RB0):** conectado al pin RB0, único pin del PIC16F887 habilitado para la interrupción externa **INT**. Mediante esta interrupción se gestionan las acciones de **iniciar**, **detener** y **resetear** la cuenta.
  - **Pulsador secundario:** función a definir en la etapa de ajuste del TP (por ejemplo, separar el reset de la función de inicio/detención).

### Temporización y refresco de displays

- **Cristal:** 4 MHz.
- **Timer0:** genera una interrupción cada **5 ms**. En cada interrupción se enciende un display distinto (multiplexado secuencial), de modo que el ciclo completo de refresco de los 4 displays se completa cada **20 ms** (4 × 5 ms).
- **Conteo del cronómetro:** cada 2 interrupciones de Timer0 (10 ms) se incrementa en 1 el dígito de centésimas. El desborde de centésimas incrementa el dígito de unidades de segundo, y así sucesivamente hasta el máximo de 99.99 s, momento en el cual la cuenta se reinicia (o se detiene, a definir).
- **Interrupción externa INT (RB0):** gestiona los pulsos del pulsador de control para alternar entre los estados de la cuenta (iniciar / detener / resetear).

### Requisitos funcionales resumidos

1. El cronómetro inicia en 00.00 y permanece detenido hasta la primera pulsación de inicio.
2. Al iniciar, cuenta ascendente con resolución de 10 ms, refrescando los 4 displays sin parpadeo perceptible (ciclo de refresco de 20 ms).
3. Puede detenerse (pausa) y reanudarse sin perder el valor acumulado.
4. Puede resetearse a 00.00 cuando está detenido.

---

## 📐 Documentación Técnica

### 1. Diagrama de Bloques
https://drive.google.com/drive/u/0/folders/1ou7aJKP65mgS5SlwTYi8-WA4XuNQNcI5 

##  Simulacion sin UART
https://drive.google.com/file/d/1dMAR9z8IlqX5DGDOdg7hepHAr-EBhGcw/view?usp=sharing

##  Código TP5
LIST    P=16F887
        #include <p16f887.inc>
	
        __CONFIG _CONFIG1, _FOSC_XT & _WDTE_OFF & _PWRTE_OFF & _MCLRE_ON & _CP_OFF & _CPD_OFF & _BOREN_ON & _IESO_ON & _FCMEN_ON & _LVP_OFF
        __CONFIG _CONFIG2, _BOR4V_BOR40V & _WRT_OFF
	
; Variables que guardo en el banco 0
	
W_TEMP          EQU     0x70
STATUS_TEMP     EQU     0x71
INDEX           EQU     0x20
NUM0            EQU     0x21    ; decenas de segundo
NUM1            EQU     0x22    ; unidades de segundo
NUM2            EQU     0x23    ; decimas
NUM3            EQU     0x24    ; centesimas
ESTADO          EQU     0x25    ; bit 0 = indica si esta contando
TICK5           EQU     0x26 ; variable  que se utiliza para contar las interrupciones del Timer0.
INT_LOCK        EQU     0x27
INT_DB          EQU     0x28 ;  Contador usado durante el antirrebote de RB0.  
RB1_LOCK        EQU     0x29
KEY_RAW         EQU     0x2A
KEY_LAST        EQU     0x2B
KEY_COUNT       EQU     0x2C
KEY_LOCK        EQU     0x2D
ENTRY_COUNT     EQU     0x2E  ; variable para indicar cuantos digitos ingreso el usuario (hasta 4). 
STARTED         EQU     0x2F ; bit 0: el cronometro ya se inicio al menos una vez
TEMP            EQU     0x34
TEMP2           EQU     0x35

; Direcciones de inicio e interrupcion    
        ORG     0x0000
        GOTO    INICIO

        ORG     0x0004
        GOTO    ISR
	
; Dejo estas tablas en esta parte de la memoria para que ADDWF PCL
; pueda usarlas sin tener que cambiar PCLATH.
	
        ORG     0x0020
	
TABLA_DISPLAY:
        ADDWF   PCL, F
        RETLW   B'00111111'       ; 0
        RETLW   B'00000110'      ; 1
        RETLW   B'01011011'      ; 2
        RETLW   B'01001111'      ; 3
        RETLW   B'01100110'      ; 4
        RETLW   B'01101101'      ; 5
        RETLW   B'01111101'       ; 6
        RETLW   B'00000111'      ; 7
        RETLW   B'01111111'       ; 8
        RETLW   B'01101111'       ; 9

HABILITACION_DISPLAY:
        ADDWF   PCL, F
        RETLW   B'00000001'
        RETLW   B'00000010'
        RETLW   B'00000100'
        RETLW   B'00001000'
	
; MAPA TECLADO
        ORG     0x0040

MAPA_TECLAS:
        ADDWF   PCL, F
	
        RETLW   D'1' ; fila A: 1, 2, 3 y resta
        RETLW   D'2'
        RETLW   D'3'
        RETLW   0xFF
	
        RETLW   D'4' ; ; fila B: 4, 5, 6 y multiplicacion
        RETLW   D'5'
        RETLW   D'6'
        RETLW   0xFF
	
        RETLW   D'7'  ; fila A: 7, 8, 9 y division
        RETLW   D'8'
        RETLW   D'9'
        RETLW   0xFF
	
        RETLW   0xFF  ; fila D: ON/C, 0, igual y suma
        RETLW   D'0'
        RETLW   0xFF
        RETLW   0xFF

; INICIALIZACION
        ORG     0x0060
	
INICIO:
        CLRF    INDEX ; ponemos todas las variables en 00
        CLRF    NUM0
        CLRF    NUM1
        CLRF    NUM2
        CLRF    NUM3
        CLRF    ESTADO  ; cronometro detenido
        CLRF    TICK5
        CLRF    INT_LOCK
        CLRF    INT_DB
        CLRF    RB1_LOCK
        CLRF    KEY_COUNT
        CLRF    KEY_LOCK
        CLRF    ENTRY_COUNT
        CLRF    STARTED ; cronometro sin cargar 
	
        MOVLW   0xFF
        MOVWF   KEY_RAW
        MOVWF   KEY_LAST
	
; Desactivo las entradas analogicas para usar RA0-RA3 como entradas digitales para el teclado
        BSF     STATUS, RP0
        BSF     STATUS, RP1
	
        CLRF    ANSEL
        CLRF    ANSELH
	
; Vuelvo al banco 1 para configurar los puertos y el Timer0.
	
        BCF     STATUS, RP1
        MOVLW   B'00001111' ; RA0-RA3 como entradas del teclado
	
        MOVWF   TRISA
        MOVLW   B'00000011' ; RB0/RB1 como entradas y RB4-RB7 como salidas, son filas
	
        MOVWF   TRISB
        CLRF    TRISD ;como salidas
	
        MOVLW   B'10110000'    ;RC0-RC3 como salidas para multiplexar
	
; RC4-RC5 entradas, RC6=TX como salida
;RC7=RX como entrada
       MOVWF TRISC
;TX: Salida de transmision del modulo UART. Envia datos bit por bit desde el ;microcontrolador hacia otro dispositivo
;RX: Entrada de recepcion del modulo UART. Recibe datos serie enviados por un dispositivo ;externo.
       
        MOVLW   B'00000011' ; pull-ups internos para RB0 y RB1
        MOVWF   WPUB
	
        MOVLW   B'00000100' ;prescalar 1:32 para el Timer0
        MOVWF   OPTION_REG 
	
;habilitamos pull-up
;interrupcion en flanco de bajada
;fuente de reloj interno
;asignamos el divisor de frecuencia al timer0 (bit3)
;prescalar 1:32 para 5ms
;Configuracion de UART
	
        MOVLW   D'25' ;velocidad estandar de 9600 baudios , de Datasheet
        MOVWF   SPBRG ;divisor de frecuencia
	
        MOVLW   B'00100100'
        MOVWF   TXSTA ;registro de estados y control de transmision
	
;bit 7, 8, 3 inhabilitado por modo asincrono y uso 8 bits
;bit5 habilitado y hsbilita TX
;bit 4 para modo asincrono
;bit 0 sin bit de paridad
	
; INTERRUPCION DE RECEPCION UART
        BSF     PIE1, RCIE ; para habilitar periferico interno y su interrupcion

; Vuelvo al banco 0 y limpio los puertos.
        BCF     STATUS, RP0
        BCF     STATUS, RP1

        CLRF    PORTA
        CLRF    PORTB
        CLRF    PORTC
        CLRF    PORTD
	
; HABILITAR EUSART
        MOVLW   B'10010000'
        MOVWF   RCSTA
	
;registro para configurar el UART como receptor
;bit 7 habilitado para TX y TX como comunicacion
;bit 4 habilita receptor continuo en modo asincrono
        ; SPEN = 1
        ; CREN = 1
; Configuro Timer0 para que interrumpa cada 4,992 ms aprox. con cristal de 4 MHz.
	
        MOVLW   D'100'
        MOVWF   TMR0
	
        BCF     INTCON, TMR0IF ; limpia la bandera del timer0
        BCF     INTCON, INTF ; limpia bandera de RB0
        BSF     INTCON, TMR0IE ; habilita interrupcion Timer0 
        BSF     INTCON, INTE; habilita interrupcion externa RB0 
        BSF     INTCON, PEIE ;habilita interrupcion por periferico (UART)
        BSF     INTCON, GIE ; habilita interrupciones en general
LAZO:
        GOTO    LAZO ;bucle para espera por interrupcion
	
	
; RUTINA DE INTERRUPCIONES
ISR:
    
;guardar contexto
        MOVWF   W_TEMP
        SWAPF   STATUS, W
        MOVWF   STATUS_TEMP ; guardamos el estado del PIC
        BCF     STATUS, RP0
        BCF     STATUS, RP1; banco 0
	
;Verificamos quien provoco la interrupcion
; UART
        BTFSC   PIR1, RCIF ;en registro de interrupcion en perifericos
        CALL    ATIENDE_UART ;si esta en 1 es que llego un dato o byte
	
; RB0
        BTFSC   INTCON, INTF ; pregunta si fue RB0
        CALL    ATIENDE_INT
	
; TIMER0
        BTFSC   INTCON, TMR0IF; pregunta si fue Timer0
        CALL    ATIENDE_TMR0
	
;recupera el contexto y vuelve al programa principal
        SWAPF   STATUS_TEMP, W
        MOVWF   STATUS
        SWAPF   W_TEMP, F
        SWAPF   W_TEMP, W
        RETFIE 
	
;Interrupcion externa RB0
	
ATIENDE_INT:
        BCF     INTCON, INTF; limpia la bandera de interrupcion
        BCF     INTCON, INTE  ;deshabilita temporalmente RB0 para evitar rebotes
        BSF     INT_LOCK, 0 ;  significa estoy procesando una pulsacion de RB0
	
        MOVLW   D'4'
        MOVWF   INT_DB; antirrebote
        CALL    INICIAR_PAUSAR
        RETURN
	
; UART RECEPCION Y TRASMISION
ATIENDE_UART:
        BTFSC   RCSTA, OERR
;en el registro de control del UART analiza el bit oerr para saber si hay error
        GOTO    UART_RECUPERA ;en caso de error
	
        BTFSC   RCSTA, FERR
;bit 2 del registro para detectar error de framing
        GOTO    UART_DESCARTA
	
        MOVF    RCREG, W ;funciona como cola mientras cuando llega los bytes
        MOVWF   TEMP
;validamos si es un numero de 0 a 9
	
; ES MENOR QUE '0'?
        MOVLW   '0'
        SUBWF   TEMP, W
        BTFSS   STATUS, C
        RETURN ;descarto si es menor a 0
        MOVWF   TEMP2 ;guarda el numero

; ES MAYOR QUE '9'?
        MOVF    TEMP, W
        SUBLW   '9'
        BTFSS   STATUS, C
        RETURN ;descarto si es mayor a 9

; ASCII a NUMERO
        MOVF    TEMP2, W
        MOVWF   TEMP
        CALL    GUARDA_DIGITO

;Devuelve por UART el digito
        MOVF    TEMP, W
        ADDLW   '0'
        CALL    ENVIA_UART
        RETURN

; ERROR UART
UART_DESCARTA:
        MOVF    RCREG, W ;descarta lectura 
        RETURN

UART_RECUPERA:
        BCF     RCSTA, CREN ;en caso de error resetea la recepcion apagando el receptor 
        BSF     RCSTA, CREN ;vueve a encender el receptor
        RETURN
	
; TRANSMITIR UN CARACTER POR UART. ENTRADA:
; W = caracter ASCII
	
ENVIA_UART:
        MOVWF   TEMP2
	
ESPERA_TX:
        BTFSS   PIR1, TXIF ;espera que el transmisor este libre
        GOTO    ESPERA_TX
        MOVF    TEMP2, W
        MOVWF   TXREG ;envia el dato
        RETURN
	

; Interrupcion del TIMER0
ATIENDE_TMR0:
        MOVLW   D'100' ; vuelve a pre cargar 100
        MOVWF   TMR0
        BCF     INTCON, TMR0IF ;limpia la bandera 
        CALL    REFRESCA_DISPLAY ; actualiza display 
        CALL    LEE_TECLADO
        CALL    ANTIREB_RB0 ;verifica esto del boton en RB0
        CALL    LEE_RB1
	
; Con dos interrupciones de Timer0 pasan unos 10 ms y cada 10ms avanza el cronometro
        BTFSC   TICK5, 0
        GOTO    TICK_10MS
        BSF     TICK5, 0
        RETURN
TICK_10MS:
        BCF     TICK5, 0
        BTFSS   ESTADO, 0
        RETURN ;en caso de no estar en estado contando no avaza
        CALL    CUENTA_ASCENDENTE
        RETURN

; ACTUALIZACION DE LOS CUATRO DIGITOS
	
REFRESCA_DISPLAY:
        CLRF    PORTC ;apaga las salidas para evitar fantasma
        MOVLW   NUM0
        ADDWF   INDEX, W; INDEX: indica que digito usamos 
        MOVWF   FSR
        MOVF    INDF, W
        CALL    TABLA_DISPLAY
        MOVWF   TEMP2 ;guardo en temp2 porque necesito W

; Prendo el punto decimal despues del segundo digito: SS.CC.
;solo si estamos en el segundo display, INDEX=1
        MOVF    INDEX, W
        XORLW   D'1'
        BTFSS   STATUS, Z
        GOTO    DP_APAGADO
        MOVF    TEMP2, W
        IORLW   B'10000000' ; agrega el bit del punto decimal por suma
        GOTO    DP_LISTO

DP_APAGADO:
        MOVF    TEMP2, W ;copia el numero sin el punto

DP_LISTO:
        MOVWF   PORTD

; SELECCION DISPLAY
        MOVF    INDEX, W
        CALL    HABILITACION_DISPLAY
        MOVWF   PORTC

;incrementa el indice del multiplexado
        INCF    INDEX, F
        MOVLW   D'4' ;solo hasta 4 porque son 4 displays
        SUBWF   INDEX, W
        BTFSS   STATUS, Z
        RETURN
        CLRF    INDEX
        RETURN

; LECTURA TECLADO
LEE_TECLADO:
        MOVLW   0xFF
        MOVWF   KEY_RAW ; en estado vacio

; FILA 0
        MOVLW   B'00010000'
        MOVWF   PORTB ;solo activo una fila
        BTFSC   PORTA, 0
        GOTO    K00 ;pongo un indice para cada tecla
        BTFSC   PORTA, 1
        GOTO    K01
        BTFSC   PORTA, 2
        GOTO    K02
        BTFSC   PORTA, 3
        GOTO    K03

; FILA 1
        MOVLW   B'00100000'
        MOVWF   PORTB
        BTFSC   PORTA, 0
        GOTO    K04
        BTFSC   PORTA, 1
        GOTO    K05
        BTFSC   PORTA, 2
        GOTO    K06
        BTFSC   PORTA, 3
        GOTO    K07

; FILA 2
        MOVLW   B'01000000'
        MOVWF   PORTB
        BTFSC   PORTA, 0
        GOTO    K08
        BTFSC   PORTA, 1
        GOTO    K09
        BTFSC   PORTA, 2
        GOTO    K10
        BTFSC   PORTA, 3
        GOTO    K11

; FILA 3
        MOVLW   B'10000000'
        MOVWF   PORTB
        BTFSC   PORTA, 0
        GOTO    K12
        BTFSC   PORTA, 1
        GOTO    K13
        BTFSC   PORTA, 2
        GOTO    K14
        BTFSC   PORTA, 3
        GOTO    K15
        GOTO    TECLADO_MUESTREO
	
	
; CODIGOS DE TECLAS
;cda tecla tiene un identificador 
K00:
        MOVLW   D'0'
        GOTO    GUARDA_TECLA
K01:
        MOVLW   D'1'
        GOTO    GUARDA_TECLA
K02:
        MOVLW   D'2'
        GOTO    GUARDA_TECLA
K03:
        MOVLW   D'3'
        GOTO    GUARDA_TECLA
K04:
        MOVLW   D'4'
        GOTO    GUARDA_TECLA
K05:
        MOVLW   D'5'
        GOTO    GUARDA_TECLA
K06:
        MOVLW   D'6'
        GOTO    GUARDA_TECLA
K07:
        MOVLW   D'7'
        GOTO    GUARDA_TECLA
K08:
        MOVLW   D'8'
        GOTO    GUARDA_TECLA
K09:
        MOVLW   D'9'
        GOTO    GUARDA_TECLA
K10:
        MOVLW   D'10'
        GOTO    GUARDA_TECLA
K11:
        MOVLW   D'11'
        GOTO    GUARDA_TECLA
K12:
        MOVLW   D'12'
        GOTO    GUARDA_TECLA
K13:
        MOVLW   D'13'
        GOTO    GUARDA_TECLA
K14:
        MOVLW   D'14'
        GOTO    GUARDA_TECLA
K15:
        MOVLW   D'15'

GUARDA_TECLA:
        MOVWF   KEY_RAW
	
; antirebote y validacion
TECLADO_MUESTREO:
        CLRF    PORTB ; dejo las filas apagadas al terminar
        MOVF    KEY_RAW, W
        XORWF   KEY_LAST, W ; compara si la tecla que lee es la misma que la anterior
        BTFSS   STATUS, Z
        GOTO    TECLA_NUEVA

; Si se mantiene la misma tecla, espero varias lecturas.
; TECLA MANTENIDA
        MOVF    KEY_RAW, W
        XORLW   0xFF
        BTFSC   STATUS, Z
        GOTO    TECLA_SUELTA
        MOVF    KEY_COUNT, W
        XORLW   D'4'
        BTFSC   STATUS, Z
        GOTO    TECLA_ESTABLE
        INCF    KEY_COUNT, F
        RETURN

TECLA_NUEVA:
        MOVF    KEY_RAW, W
        MOVWF   KEY_LAST
        MOVLW   D'1'
        MOVWF   KEY_COUNT
        RETURN

TECLA_SUELTA:
        CLRF    KEY_COUNT
        BCF     KEY_LOCK, 0
        RETURN

TECLA_ESTABLE:
        BTFSC   KEY_LOCK, 0 ;se procesa todo una sola vez por pulsacion
        RETURN
        BSF     KEY_LOCK, 0
        CALL    PROCESA_TECLA
        RETURN

; La tecla + inicia, pausa o continua la cuenta.
PROCESA_TECLA:
        ; Solo acepta teclas numericas (0 a 9).
        ; Las teclas no numericas se ignoran.
        ; No cargar numeros mientras cuenta
	
        BTFSC   ESTADO, 0
        RETURN

        ; No modificar un tiempo ya iniciado
        BTFSC   STARTED, 0
        RETURN

        ; Convertir la posicion de la tecla en un numero
        MOVF    KEY_RAW, W
        CALL    MAPA_TECLAS
        MOVWF   TEMP

        ; Si devuelve 0xFF, no es una tecla numerica
        XORLW   0xFF
        BTFSC   STATUS, Z
        RETURN

        ; Guardar el digito
        CALL    GUARDA_DIGITO

        ; Transmitir el digito por UART
        MOVF    TEMP, W
        ADDLW   '0'
        CALL    ENVIA_UART
        RETURN

GUARDA_DIGITO:
        BTFSC   ESTADO, 0
        RETURN

        BTFSC   STARTED, 0
        RETURN

        MOVF    ENTRY_COUNT, W
        XORLW   D'4' ;hatsa 4 digitos
        BTFSC   STATUS, Z
        RETURN

; Desplaza los digitos ingresados a la izqueirda
        MOVF    NUM1, W
        MOVWF   NUM0
        MOVF    NUM2, W
        MOVWF   NUM1
        MOVF    NUM3, W
        MOVWF   NUM2
        MOVF    TEMP, W
        MOVWF   NUM3
        INCF    ENTRY_COUNT, F

        RETURN
	
INICIAR_PAUSAR:
        BTFSC   ESTADO, 0
        GOTO    PAUSA_TIMER ; si estaba contando, pausa
        BTFSC   STARTED, 0
        GOTO    REANUDA_TIMER ; si estaba en pausa, continua
	
; Para iniciar por primera vez, tienen que estar cargados 4 digitos.
        MOVF    ENTRY_COUNT, W
        XORLW   D'4' ; exige que ingresemos 4 digitos
        BTFSS   STATUS, Z
        RETURN
        BSF     STARTED, 0
        GOTO    REANUDA_TIMER
	
PAUSA_TIMER:
        BCF     ESTADO, 0
        RETURN
; Reanuda la cuenta desde el valor actual del display.
	
REANUDA_TIMER:
        BSF     ESTADO, 0
        RETURN
	
RESET_TEMPORIZADOR:
    
        ; RB1 detiene y reinicia siempre a 00.00
        BCF     ESTADO, 0
        BCF     STARTED, 0

        CLRF    NUM0
        CLRF    NUM1
        CLRF    NUM2
        CLRF    NUM3


        CLRF    ENTRY_COUNT
        RETURN
	
BORRA_ENTRADA:
; Limpia la entrada cuando el cronometro esta detenido 
        BTFSC   ESTADO, 0
        RETURN
        BCF     STARTED, 0
        CLRF    ENTRY_COUNT
        CLRF    NUM0
        CLRF    NUM1
        CLRF    NUM2
        CLRF    NUM3
	
        RETURN
	
CUENTA_ASCENDENTE:
; Si esta en 99.99, la siguiente cuenta vuelve a 00.00.
    
        MOVF    NUM0, W
        XORLW   D'9'
        BTFSS   STATUS, Z
        GOTO    INCREMENTA_TIEMPO
	
        MOVF    NUM1, W
        XORLW   D'9'
        BTFSS   STATUS, Z
        GOTO    INCREMENTA_TIEMPO
	
        MOVF    NUM2, W
        XORLW   D'9'
        BTFSS   STATUS, Z
        GOTO    INCREMENTA_TIEMPO
	
        MOVF    NUM3, W
        XORLW   D'9'
        BTFSS   STATUS, Z
        GOTO    INCREMENTA_TIEMPO
	
;si llego a 99.9 reinicia la cuenta
        CLRF    NUM0
        CLRF    NUM1
        CLRF    NUM2
        CLRF    NUM3
        RETURN
	
INCREMENTA_TIEMPO:
; Primero aumento las centesimas.
        INCF    NUM3, F
        MOVLW   D'10'
        SUBWF   NUM3, W
        BTFSS   STATUS, Z
        RETURN
        CLRF    NUM3
	
; Cuando las centesimas llegan a 10, aumento las decimas
        INCF    NUM2, F
        MOVLW   D'10'
        SUBWF   NUM2, W
        BTFSS   STATUS, Z
        RETURN
        CLRF    NUM2
	
; Despues aumento las unidades de segundo.
        INCF    NUM1, F
        MOVLW   D'10'
        SUBWF   NUM1, W
        BTFSS   STATUS, Z
        RETURN
        CLRF    NUM1
	
; Finalmente aumento las decenas de segundo.
        INCF    NUM0, F
        RETURN

ANTIREB_RB0:
        BTFSS   INT_LOCK, 0
        RETURN
        BTFSS   PORTB, 0
        GOTO    RB0_SIGUE_PULSADO
        DECFSZ  INT_DB, F ;decrementa si el boton esta suelto
        RETURN
        BCF     INT_LOCK, 0
        BCF     INTCON, INTF
        BSF     INTCON, INTE ;habilita la interrupcion externa
        RETURN
	
RB0_SIGUE_PULSADO:
    
        MOVLW   D'4' ;recarga el contador antirebote
        MOVWF   INT_DB
        RETURN

LEE_RB1:
        BTFSC   PORTB, 1
        GOTO    RB1_LIBERADO
        BTFSC   RB1_LOCK, 0 ;para evitar que se ejecute mas de una vez por pulsacion
        RETURN
        BSF     RB1_LOCK, 0
        CALL    RESET_TEMPORIZADOR
        RETURN

RB1_LIBERADO:
        BCF     RB1_LOCK, 0 ;habilita deteccion
        RETURN

        END


