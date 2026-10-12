# TP4 — Cronómetro Digital y Comunicacion en serie

Comisión / Docente: Blasco Marcos - 
Comisión 4.2
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
