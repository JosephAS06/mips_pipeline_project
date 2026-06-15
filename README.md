## Implementación funcional

La implementación del coprocesador se validó en dos niveles. Primero se probó el
núcleo aritmético `fir_core.sv` de forma aislada, con el fin de comprobar que la
convolución FIR se estuviera calculando correctamente en punto fijo. Después se probó
`fir_wb_wrapper.sv`, que encapsula el core y permite accederlo como un periférico
mapeado en memoria a través del bus Wishbone.

Esta separación fue importante porque permitió depurar primero la parte matemática
del filtro y luego validar el comportamiento del sistema desde el punto de vista de
la CPU. Es decir, no solo se verificó que el filtro produjera la salida correcta,
sino también que los registros de control, estado, entrada, coeficientes y resultado
funcionaran como se espera dentro del mapa de memoria del SoC.

Para correr estas pruebas se utilizaron los testbenches ubicados en `tb/`, junto con
vectores de referencia generados en Python. Las señales utilizadas en las
demostraciones también son sintéticas y fueron generadas en Python, de manera que se
pudiera comparar la salida del hardware contra una referencia determinista en
software.

---

### Validación de `fir_core.sv`

El testbench de `fir_core.sv` se diseñó para verificar tanto el comportamiento
nominal del filtro como sus casos frontera. La primera prueba fue la respuesta al
impulso, aplicando `x[0] = 32767` y `x[k] = 0` para las muestras siguientes. En este
caso, la salida esperada en cada ciclo corresponde a `coeffs[k]·32767`, por lo que
esta prueba permite comprobar directamente que los coeficientes se estén aplicando
en el orden correcto.

Luego se probó la respuesta a escalón con una entrada constante de 0.5 en formato
Q1.15. Esta prueba permite observar cómo la salida converge según la suma acumulada
de los coeficientes del filtro. También se aplicó una señal de audio sintética
compuesta por una componente de 1 kHz y otra de 3.8 kHz, con el fin de comprobar el
comportamiento del filtro pasa-bajas en una señal representativa.

Además de las pruebas anteriores, se incluyeron casos frontera para verificar la
aritmética con signo y el comportamiento del reset:

| Prueba | Entrada | Resultado esperado |
|--------|---------|--------------------|
| Máximo positivo | `32767` | Salida consistente con la referencia en software |
| Máximo negativo | `-32768` | Conservación correcta del signo en punto fijo |
| Todos ceros | `0` en cada ciclo | Salida cero en cada ciclo |
| Reset a mitad del cálculo | Reset activo durante el procesamiento | `p_chain` queda en cero |

Estas pruebas permitieron validar que el núcleo aritmético funciona correctamente
para señales típicas, condiciones extremas y eventos de reinicio. En particular, el
reset a mitad del cálculo fue útil para comprobar que la cadena interna de
acumulación no conservara datos inválidos después de limpiar el core.

---

### Validación de `fir_wb_wrapper.sv`

Una vez validado el core, se probó el wrapper Wishbone. Este módulo es el que permite
que el filtro se comporte como un periférico del SoC, por lo que la validación se
centró en el acceso a registros, la máquina de estados y el handshake del bus.

Las pruebas realizadas fueron las siguientes:

| Prueba | Descripción |
|--------|-------------|
| Reset | Verificar el estado inicial de todos los registros |
| Registro de acceso | Comprobar `CTRL` R/W, `STATUS` R/O, `DATA_IN` W/O y `RESULT` R/O |
| Auto-limpieza | Verificar que el bit `start` se limpie después de 1 ciclo |
| Flujo nominal LP | Cargar coeficientes, escribir una muestra, activar `start` y leer `RESULT` |
| Reconfiguración | Cambiar coeficientes entre muestras sin aplicar reset global |
| `core_srst` | Limpiar la línea de retardo interna usando `CTRL[1]` |
| Consecutivos | Enviar N muestras back-to-back sin polling entre ellas |
| Wishbone | Verificar `ACK` de 1 ciclo y ausencia de dobles confirmaciones |

Con estas pruebas se verificó que el periférico respeta el mapa de registros y que
el flujo de operación es el esperado: primero se cargan los coeficientes, luego se
escribe la muestra de entrada en `FIR_DATA_IN`, se activa `start` desde `FIR_CTRL`,
se espera el bit `done` en `FIR_STATUS` y finalmente se lee el valor filtrado desde
`FIR_RESULT`.

También se validó que los coeficientes puedan modificarse por firmware entre
muestras. Esto es importante porque el diseño permite reconfigurar la respuesta del
filtro sin cambiar la interfaz de acceso desde la CPU. No obstante, como `N_TAPS` es
un parámetro de síntesis, el número de taps sigue quedando definido por el bitstream
utilizado.

---

### Prueba funcional: filtro pasa-bajas

La primera demostración funcional se realizó con el filtro pasa-bajas de N = 33 taps.
La señal de entrada se generó en Python y corresponde a la suma de una componente de
1 kHz con otra de 3.8 kHz, usando una frecuencia de muestreo de 8 kHz. La intención
de esta prueba fue representar una señal de voz contaminada con una componente de
alta frecuencia.

<p align="center">
  <img src="fir_coprocessor/python/fig_lp_demo.png" alt="Demo filtro FIR pasa-bajas" width="85%">
  <br>
  <em>Figura 2. Demostración del filtro FIR pasa-bajas con N = 33 taps.</em>
</p>

En la figura se observa que la señal de entrada contiene ambas componentes, mientras
que la salida filtrada conserva principalmente la componente de 1 kHz. En el dominio
de la frecuencia también se aprecia que la componente de 3.8 kHz cae dentro de la
banda de rechazo del filtro.

La salida obtenida en hardware coincide con la referencia generada en software. El
error máximo medido fue de 0 LSB en formato Q2.30, lo que confirma que la
implementación en FPGA reproduce bit a bit el resultado esperado para esta prueba.

---

### Prueba funcional: filtro notch de 60 Hz

La segunda demostración funcional se realizó con el filtro notch de N = 129 taps. En
este caso, la señal de entrada corresponde a un pulso QRS sintético contaminado con
una interferencia sinusoidal de 60 Hz. La frecuencia de muestreo utilizada fue de
500 Hz, como en una aplicación básica de ECG.

<p align="center">
  <img src="fir_coprocessor/python/fig_notch_demo.png" alt="Demo filtro FIR notch de 60 Hz" width="85%">
  <br>
  <em>Figura 3. Demostración del filtro FIR notch de 60 Hz con N = 129 taps.</em>
</p>

El resultado muestra que la componente de 60 Hz se atenúa sin eliminar la forma
principal del pulso QRS. En el espectro se observa la banda notch alrededor de 60 Hz,
mientras que la salida del hardware se superpone con la referencia de software.

Al igual que en la prueba pasa-bajas, el error máximo entre hardware y software fue
de 0 LSB en Q2.30. Esta prueba también permitió validar la configuración de mayor
tamaño del coprocesador, ya que para N = 129 se utilizan dos cadenas de DSPs en lugar
de una única cadena.

---

### Benchmark de desempeño

Además de verificar la exactitud de la salida, se midió la cantidad de ciclos
necesarios para procesar 256 muestras con el filtro notch de N = 129 taps. La
comparación se hizo entre la implementación en hardware y una versión equivalente en
software ejecutada por el SweRV EH1 a 50 MHz.

<p align="center">
  <img src="fir_coprocessor/python/fig_benchmark.png" alt="Benchmark coprocesador FIR FPGA vs software" width="85%">
  <br>
  <em>Figura 4. Benchmark del coprocesador FIR en FPGA contra software.</em>
</p>

El coprocesador requirió 20 766 ciclos de reloj para procesar las 256 muestras,
mientras que la versión en software requirió 3 433 624 ciclos. Esto corresponde a un
speedup de 165.3×.

| Configuración | Muestras | Ciclos HW | Ciclos SW | Speedup |
|---------------|----------|-----------|-----------|---------|
| Notch N = 129 | 256 | 20 766 | 3 433 624 | 165.3× |

Este resultado confirma que el coprocesador no solo produce la misma salida que la
referencia en software, sino que además reduce de forma significativa la cantidad de
ciclos requeridos para procesar la señal. Por tanto, el uso de hardware dedicado
permite liberar a la CPU de la operación de convolución y habilita el procesamiento
de señales en tiempo real con una carga mucho menor para el procesador.
