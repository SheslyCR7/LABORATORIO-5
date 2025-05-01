# LABORATORIO-5

**Luz Marina Valderrama-5600741**

**Shesly Nicole Colorado - 5600756**

**Samuel Esteban Fonseca Luna - 5600808**

## OBJETIVO 

Este laboratorio tiene la finalidad de mostrar de manera gráfica la representación de la activación del sistema simpático y parasimpático en un sujeto de 19 años sin ninguna patología cardiaca al que se le aplicó una prueba, la cual consistía en el aumento de pulso cardíaco debido a la respuesta simpática del cuerpo al sentirse en la sensación de peligro la cual fue inducida al aguantar la respiración hasta llegar a la sensación de ahogo que genera un instinto básico de supervivencia el cual sería conseguir oxígeno y lograr respirar, de esta manera entra en juego el sistema parasimpático al reconocer ninguna sensación de peligro y regular el ritmo cardíaco bajando su frecuencia

### Materiales y métodos

- DAQ (Dispositivo de Adquisición de Datos)

- Sensor AD8232

- Python 3.11

- Librerías: NumPy, SciPy, Matplotlib, pyserial

# Adquisición

La señal ECG fue registrada durante un experimento que incluyó una fase de reposo, una fase de apnea voluntaria (retención de la respiración) y una fase de recuperación. Se utilizó un sistema de adquisición basado en NI DAQ con una frecuencia de muestreo de 500 Hz, lo cual proporciona una resolución adecuada para detectar los picos R característicos del ECG. Los datos se recolectaron durante un total de 5 minutos, asegurando así la captura de suficientes ciclos cardíacos para realizar un análisis significativo del HRV.

La adquisición se realizó mediante un script de Python que utilizó la librería nidaqmx, donde se leyeron bloques de 1000 muestras y se almacenaron en tiempo real en un archivo .txt. Simultáneamente, la señal se graficó en pantalla para tener una referencia visual inmediata de la calidad del ECG.

  
    import nidaqmx
    import numpy as np
    import matplotlib.pyplot as plt
    import time
    from nidaqmx.constants import AcquisitionType

## Parámetros de adquisición

    tasa_muestreo = 500  # Frecuencia de muestreo en Hz (10 kS/s)
    duracion = 300  # Duración total de la adquisición en segundos
    num_muestras = tasa_muestreo * duracion  # Número total de muestras
    nombre_archivo = r"C:\Users\juane\OneDrive\Documentos\jeje\datos_ecg.txt"  # Ruta personalizada para el archivo .txt

## Inicializar listas para almacenar datos

     tiempos = []
     datos = []

## Configurar Matplotlib para graficar en tiempo real

     plt.ion()  # Activar modo interactivo
     fig, ax = plt.subplots()
     line, = ax.plot([], [], label="Señal AI1")  # Línea vacía para actualizar datos
     ax.set_xlim(0, duracion)  # Eje X: tiempo en segundos
     ax.set_ylim(-10, 10)  # Ajuste del voltaje (modifica según tu DAQ)
     ax.set_xlabel("Tiempo (s)")
    ax.set_ylabel("Voltaje (V)")
    ax.set_title("Señal en Tiempo Real")
    ax.legend()
    ax.grid()

## Crear tarea de adquisición

    with nidaqmx.Task() as task:
       task.ai_channels.add_ai_voltage_chan("Dev3/ai0")  # Canal de entrada AI1 (ajustar según DAQ)
       task.timing.cfg_samp_clk_timing(rate=tasa_muestreo, sample_mode=AcquisitionType.CONTINUOUS, 
     samps_per_chan=1000)

        print("Iniciando adquisición en tiempo real... (Presiona Ctrl+C para detener)")

        tiempo_inicio = time.time()  # Marcar tiempo de inicio
 
      try:
          while time.time() - tiempo_inicio < duracion:  # Ejecutar por el tiempo especificado
              data = task.read(number_of_samples_per_channel=1000)  # Leer 1000 muestras por iteración
              datos.extend(data)  # Guardar en la lista de datos
              tiempos.extend(np.linspace(time.time() - tiempo_inicio, time.time() - tiempo_inicio + len(data) / tasa_muestreo, 
    len(data)))

 ## Actualizar gráfico en tiempo real

            line.set_xdata(tiempos)
            line.set_ydata(datos)
            ax.relim()
            ax.autoscale_view()
            plt.pause(0.01)

     except KeyboardInterrupt:
         print("\nAdquisición detenida por el usuario.")

    print("Adquisición completada. Guardando datos en TXT...")

## Guardar datos en un archivo .txt

    with open(nombre_archivo, 'w') as f:
       f.write("Tiempo (s)\tVoltaje (V)\n")  # Encabezados
       for t, v in zip(tiempos, datos):
         f.write(f"{t:.6f}\t{v:.6f}\n")  # Escribir cada par de tiempo y voltaje con 6 decimales

    print(f" Datos guardados en '{nombre_archivo}'")

## Desactivar modo interactivo y mostrar la gráfica final
    plt.ioff()
    plt.figure(figsize=(10, 5))
    plt.plot(tiempos, datos, label="Señal adquirida", color="b")

# Preprocesamiento de la señal
Antes del análisis, se aplicó un preprocesamiento esencial para limpiar la señal de artefactos eléctricos y ruidos de alta frecuencia. Primero, se centró la señal eliminando su componente DC (promedio), lo que permite un análisis más preciso de las variaciones rápidas del ECG. Posteriormente, se aplicó un filtro pasa bajos de segundo orden con una frecuencia de corte de 60 Hz, suficiente para preservar las componentes útiles del ECG mientras se eliminan interferencias como el ruido de red o la actividad muscular.

Este filtrado permitió obtener una señal más clara, con picos R bien definidos, condición indispensable para el análisis de la variabilidad del ritmo cardíaco.


# Codigo ECG

    import numpy as np
    import pandas as pd
    import matplotlib.pyplot as plt
    from scipy.signal import butter, filtfilt, find_peaks
    import scipy.stats as stats
    import pywt

## Cargar datos ECG desde el archivo CSV

     file_path = r"C:\Users\shesl\Downloads\lab 5\datos_ecg.txt"

## Cargar datos con pandas usando 'sep' en lugar de 'delim_whitespace'

     data = pd.read_csv(file_path, sep='\s+')
     print(data.columns)

## Extraer las columnas necesarias

    time = data['Tiempo(s)'].values
    voltage = data['Voltaje(V)'].values

## Asegurémonos de que la señal está centrada (restando la media)

    voltage = voltage - np.mean(voltage)

## Filtro pasa-bajas para las ondas R (0.5 Hz a 50 Hz)

    def butter_lowpass(cutoff, fs, order=5):
      nyquist = 0.5 * fs
      normal_cutoff = cutoff / nyquist
      b, a = butter(order, normal_cutoff, btype='low', analog=False)
      return b, a

    def butter_lowpass_filter(data, cutoff, fs, order=5):
       b, a = butter_lowpass(cutoff, fs, order)
       y = filtfilt(b, a, data)
       return y

## Parámetros del filtro pasabajas

    fs = 1 / np.mean(np.diff(time))  # Frecuencia de muestreo (ajustar según los datos)
    cutoff = 60  # Frecuencia de corte del filtro (ajustada a 60 Hz)
    filtered_data = butter_lowpass_filter(voltage, cutoff, fs)

## Detección de picos R con umbral más bajo para evitar picos de ruido

    def detect_r_peaks(signal, threshold=0.5):  # Ajuste del umbral para evitar ruido
      peaks, _ = find_peaks(signal, height=threshold)
      return peaks

    r_peaks = detect_r_peaks(filtered_data)

## Filtrar los picos R para que no excedan el rango de los puntos definidos

    r_peaks_filtered = r_peaks[r_peaks < len(time)]  # Asegurarnos de que los picos estén dentro del rango

## Verificar si se han detectado picos R

    print(f"Número de picos R detectados: {len(r_peaks_filtered)}")

# Visualizar la señal original y filtrada en un solo gráfico

    plt.figure(figsize=(10, 6))

# Visualizar la señal original centrada

    plt.figure(figsize=(10, 6))
    plt.plot(time[:300000], voltage[:300000], label="Señal ECG Original Centrada")
    plt.xlabel('Tiempo (s)')
    plt.ylabel('Voltaje (V)')
    plt.title('Señal ECG Original Centrada ')
    plt.legend()
    plt.show()

# Graficar la señal original

    plt.subplot(1, 2, 1)  # 1 fila, 2 columnas, primer gráfico
    plt.plot(time[:3000], voltage[:3000], label="Señal ECG Original")
    plt.xlabel('Tiempo (s)')
    plt.ylabel('Voltaje (V)')
    plt.title('Señal ECG Original')
    plt.legend()

![Image](https://github.com/user-attachments/assets/fafd633f-473d-47b0-b5ae-27db621c530c)

# Graficar la señal filtrada

     plt.subplot(1, 2, 2)  # 1 fila, 2 columnas, segundo gráfico
     plt.plot(time[:3000], filtered_data[:3000], label="Señal ECG Filtrada")
     plt.xlabel('Tiempo (s)')
     plt.ylabel('Voltaje (V)')
     plt.title('Señal ECG Filtrada')
     plt.legend()

     plt.tight_layout()  # Ajustar espacio entre los subgráficos
     plt.show()


![Image](https://github.com/user-attachments/assets/47295a4d-8e28-40b2-8fb7-c98789e86aa9)

# Visualizar la señal con los picos R detectados

     plt.figure(figsize=(10, 6))
     plt.plot(time[:3000000], filtered_data[:3000000], label="Señal ECG Filtrada")
     plt.plot(time[:3000000][r_peaks_filtered], filtered_data[:3000000][r_peaks_filtered], 'ro', label="Picos R")
     plt.xlabel('Tiempo (s)')
     plt.ylabel('Voltaje (V)')
     plt.title('Señal ECG Filtrada con Picos R Detectados')
     plt.legend()
     plt.show()

![Image](https://github.com/user-attachments/assets/927c4875-163f-42bd-b266-ba03f1e7c9ba)

## Cálculo de los intervalos R-R


Para calcular el HRV, se requiere identificar con precisión los picos R de la señal ECG, ya que estos corresponden al momento exacto en que ocurre la contracción ventricular. Se utilizó la función find_peaks de la librería scipy.signal, estableciendo un umbral de 0.5 V y una distancia mínima entre picos adecuada al ritmo cardíaco esperado. Esta configuración permitió obtener una secuencia confiable de marcas temporales correspondientes a cada latido.

A partir de estos picos se calcularon los intervalos R-R, que representan el tiempo transcurrido entre un latido y el siguiente. Esta información constituye la base para analizar la HRV en el dominio del tiempo y en frecuencia. 




    if len(r_peaks_filtered) > 1:  # Asegurarnos de que haya más de un pico R
       rr_intervals = np.diff(r_peaks_filtered) / fs  # En segundos
     else:
       rr_intervals = []

# Análisis de HRV (Media y Desviación Estándar)

     if len(rr_intervals) > 0:
       hrv_mean = np.mean(rr_intervals)
       hrv_std = np.std(rr_intervals)
       print(f"Media de los intervalos R-R: {hrv_mean:.4f} segundos")
       print(f"Desviación estándar de los intervalos R-R: {hrv_std:.4f} segundos")
    else:
       print("No se pudieron calcular los intervalos R-R debido a una detección insuficiente de picos R.")



Con los intervalos R-R ya identificados, se procedió al análisis estadístico de la HRV. Se calcularon la media de los intervalos y su desviación estándar. La media brinda información sobre la frecuencia cardíaca general durante el periodo de análisis, mientras que la desviación estándar refleja la variabilidad del ritmo cardíaco, un indicador clave del balance simpático-parasimpático.

Durante la fase de apnea, observamos un acortamiento en los intervalos R-R, lo cual sugiere un aumento en la frecuencia cardíaca, indicador de una activación del sistema simpático. Posteriormente, durante la fase de recuperación, los intervalos se alargan progresivamente, lo cual es una señal clara de la activación parasimpática, encargada de restaurar la homeostasis.



# Análisis Wavelet (CWT)

     def wavelet_transform(signal, scales=np.linspace(1, 128, 256)):
         coefficients, freqs = pywt.cwt(signal, scales, 'cmor1.0-1.0')
         return coefficients, freqs

Para profundizar en el análisis de HRV, se utilizó una transformada wavelet continua (CWT), específicamente con la wavelet de Morlet compleja (cmor1.0-1.0). Esta técnica permite observar cómo evoluciona el contenido frecuencial de la señal HRV a lo largo del tiempo, lo que no es posible con métodos como la transformada de Fourier.

El espectrograma resultante muestra zonas de alta energía en frecuencias altas durante la fase de apnea, consistentes con una activación simpática. En contraste, las frecuencias más bajas se ven reforzadas durante la fase de recuperación, reflejando el predominio del sistema parasimpático. Esta herramienta permite distinguir de manera visual y precisa los cambios dinámicos en la regulación autonómica del corazón.

## Verificar si tenemos intervalos R-R válidos para el espectrograma

      if len(rr_intervals) > 0:
        coefficients, freqs = wavelet_transform(rr_intervals)

    # Visualización del espectrograma

       plt.figure(figsize=(10, 6))
       plt.imshow(np.abs(coefficients), aspect='auto', extent=[0, len(rr_intervals), freqs[-1], freqs[0]],
               cmap='jet', origin='lower')
       plt.colorbar(label='Magnitud de la Transformada Wavelet')
       plt.ylabel('Frecuencia (Hz)')
       plt.xlabel('Tiempo (s)')
       plt.title('Espectrograma HRV usando la Transformada Wavelet')
       plt.grid(True, which='both', linestyle='--', alpha=0.3)
       plt.show()
     else:
         print("No se pudo realizar el análisis wavelet debido a la falta de intervalos R-R válidos.")

![Image](https://github.com/user-attachments/assets/eba5ff20-0270-43e6-a5de-e9b565d597f0)

# Conclusiones


Este laboratorio permitió evidenciar, a través del análisis de la señal ECG, cómo el cuerpo humano responde a un estímulo de estrés agudo como lo es la apnea voluntaria. Durante la apnea, se observó un aumento en la frecuencia cardíaca y una disminución en la HRV, lo que indica una respuesta del sistema simpático. Posteriormente, la fase de recuperación evidenció una disminución en la frecuencia cardíaca y un aumento en la HRV, indicadores de activación del sistema parasimpático.

El uso combinado de técnicas de detección de picos, análisis temporal y espectral permitió caracterizar esta respuesta de forma robusta y detallada. Además, la transformada wavelet demostró ser una herramienta poderosa para visualizar la evolución temporal de las frecuencias asociadas al HRV, reforzando su utilidad en el análisis de señales biológicas no estacionarias.
