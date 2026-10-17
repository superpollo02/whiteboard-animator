# 🎬 Whiteboard Animator & Infographic Animation Pipeline

Convierte diagramas, infografías, esquemas y bocetos estáticos estilo pizarra blanca en animaciones de revelado paso a paso dibujadas a mano (Whiteboard Animation videos en formato `.mp4`).

Este repositorio documenta el entorno, casos prácticos y automatizaciones para animar infografías y diagramas de procesos utilizando `whiteboard-animator`.

---

## 📌 Tabla de Contenidos
- [Características Principales](#-características-principales)
- [¿Cómo Funciona el Motor por Detrás?](#-cómo-funciona-el-motor-por-detrás)
- [Requisitos Previos](#-requisitos-previos)
  - [1. FFmpeg (Indispensable)](#1-ffmpeg-indispensable)
  - [2. Python y Paquete](#2-python-y-paquete)
- [Uso y Comandos Rápidos](#-uso-y-comandos-rápidos)
  - [Ejemplo 1: Duración fija (Sin audio)](#ejemplo-1-duración-fija-sin-audio)
  - [Ejemplo 2: Con audio / narración](#ejemplo-2-con-audio--narración)
  - [Ejemplo 3: Múltiples escenas encadenadas](#ejemplo-3-múltiples-escenas-encadenadas)
  - [Opciones de Calidad](#opciones-de-calidad)
- [Paginado y Regiones Avanzadas (Con IA / Gemini)](#-paginado-y-regiones-avanzadas-con-ia--gemini)
- [Uso mediante API de Python](#-uso-mediante-api-de-python)
- [Buenas Prácticas para Imágenes de Entrada](#-buenas-prácticas-para-imágenes-de-entrada)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Licencia y Créditos](#-licencia-y-créditos)

---

## 🚀 Características Principales

- **Detección inteligente de texto y trazos:** Identifica palabras completas (usando un modelo ligero de detección ONNX CRAFT) para trazarlas palabra por palabra en lugar de píxeles inconexos.
- **Rellenos orgánicos:** Descompone formas y áreas coloreadas simulando trazos de pincel/marcador con barridos naturales.
- **Trazado de esquemas y ramificaciones:** Identifica gráficos lineales, flechas y conexiones para dibujarlos de forma continua.
- **Sincronización con narración de voz:** Ajusta la velocidad de dibujo para que termine dentro del tiempo de la locución y mantenga el fotograma final visible.
- **100% CPU, sin necesidad de GPU:** No requiere CUDA ni tarjetas gráficas dedicadas.

---

## 🧠 ¿Cómo Funciona el Motor por Detrás?

El proceso de renderizado transforma la imagen en fotogramas a través de 5 etapas deterministas:

1. **Umbralización y Limpieza de Fondo:**  
   Identifica los píxeles de tinta sobre fondo claro (los píxeles con brillo $> 240$ se consideran fondo blanco).
2. **Detección de Texto (CRAFT ONNX):**  
   Localiza las cajas y polígonos donde hay texto para planificar su escritura secuencial (de izquierda a derecha / arriba a abajo).
3. **Esqueletización y Análisis Topológico:**  
   Calcula el esqueleto de las líneas y curvas para simular el trazo con rotulador paso a paso.
4. **Planificación de Trazos y Relleno:**  
   Divide el contenido en curvas, líneas y regiones rellenas. Asigna ventanas de tiempo por elemento o región visual.
5. **Composición y Streaming directo a FFmpeg:**  
   Genera los fotogramas en memoria y los envía directamente mediante una tubería (pipe) al proceso `ffmpeg` para codificar el video H.264/AAC sin generar gigabytes de imágenes intermedias en disco.

---

## 🛠 Requisitos Previos

### 1. FFmpeg (Indispensable)

`whiteboard-animator` necesita `ffmpeg` y `ffprobe` disponibles en la variable de entorno `PATH` para codificar los videos `.mp4`.

- **Windows (vía winget):**
  ```powershell
  winget install Gyan.FFmpeg --accept-package-agreements --accept-source-agreements
  ```
- **Windows (Manual):** Descargar desde [ffmpeg.org](https://ffmpeg.org/download.html) y agregar la carpeta `bin` a las variables de entorno.
- **macOS:**
  ```bash
  brew install ffmpeg
  ```
- **Ubuntu/Debian:**
  ```bash
  sudo apt update && sudo apt install -y ffmpeg
  ```

Verifica la instalación ejecutando:
```bash
ffmpeg -version
ffprobe -version
```

### 2. Python y Paquete

Se recomienda Python 3.10 o superior:

```bash
pip install whiteboard-animator
```

*(Opcional: Si deseas detección automática de regiones con Google Gemini)*:
```bash
pip install "whiteboard-animator[gemini]"
```

---

## 💻 Uso y Comandos Rápidos

### Ejemplo 1: Duración fija (Sin audio)
Animar una imagen especificando los segundos de duración:

```powershell
whiteboard-animate "Infografia Proceso.jpg" --duration 8 -o sketch.mp4
```

O para diagramas más detallados con mayor duración:
```powershell
whiteboard-animate "Circuito_de_gestión_de_turnos.consultorios externos.png" --duration 16 -o "Circuito_Turnos.mp4"
```

> 💡 **Nota en Windows:** Si tu ejecutable no se encuentra en el PATH global inmediatamente tras instalarlo con pip, puedes invocarlo directamente con el módulo de Python:
> ```powershell
> python -m whiteboard_animator.cli "Infografia Proceso.jpg" --duration 8 -o sketch.mp4
> ```

### Ejemplo 2: Con audio / narración
Si tienes una locución grabada (`.wav` o `.mp3`), el dibujo sincronizará automáticamente su velocidad:

```bash
whiteboard-animate diagrama.png --audio narracion.wav -o presentacion.mp4
```

### Ejemplo 3: Múltiples escenas encadenadas
Puedes pasar varias imágenes y sus audios correspondientes en orden para producir un video continuo:

```bash
whiteboard-animate escena1.png escena2.png --duration 10 -o video_completo.mp4
```

### Opciones de Calidad
Puedes configurar el preset de renderizado con `--quality`:
- `low`: 20 fps, 500 kbps (ideal para previsualizaciones rápidas).
- `medium`: 24 fps, 1500 kbps (por defecto, balance óptimo).
- `high`: 24 fps, 3000 kbps (máxima nitidez para presentaciones).

```bash
whiteboard-animate "Infografia Proceso.jpg" --quality high --duration 12 -o salida_hd.mp4
```

---

## 🎯 Paginado y Regiones Avanzadas (Con IA / Gemini)

Por defecto, la animación dibuja los elementos visuales en un orden heurístico geométrico. Para guiar exactamente el orden de revelado y asociar partes de la imagen con la narración:

```bash
whiteboard-animate esquema.png --audio audio.mp3 --regions regiones.json -o salida.mp4
```

### Formato de `regiones.json`
Las coordenadas `ymin`, `xmin`, `ymax`, `xmax` se definen normalizadas de 0 a 1000:

```json
{
  "idea": "Proceso de Atención al Paciente",
  "regions": [
    {
      "label": "recepcion",
      "reveal_order": 1,
      "box": {"ymin": 50, "xmin": 50, "ymax": 450, "xmax": 450},
      "annotation": "El paciente solicita el turno en ventanilla",
      "reveal": "fill"
    },
    {
      "label": "consultorio",
      "reveal_order": 2,
      "box": {"ymin": 50, "xmin": 500, "ymax": 450, "xmax": 950},
      "annotation": "El médico confirma la atención en sistema",
      "reveal": "fill"
    }
  ]
}
```

---

## 🐍 Uso mediante API de Python

También es posible integrar el pipeline directamente en scripts de Python:

```python
from whiteboard_animator import Scene, render_video

scenes = [
    Scene("Infografia Proceso.jpg", duration=8.0),
    Scene("Circuito_de_gestión_de_turnos.consultorios externos.png", duration=16.0)
]

render_video(
    scenes,
    "video_consolidado.mp4",
    quality="high"
)
print("¡Video exportado exitosamente!")
```

---

## 🎨 Buenas Prácticas para Imágenes de Entrada

Para obtener los mejores resultados de animación:
1. **Fondo Blanco o Muy Claro:** Colores de fondo oscuros o degradados complejos no son compatibles con el efecto de pizarra.
2. **Líneas y Textos Definidos:** Ilustraciones vectoriales rasterizadas, diagramas de flujo (Miro, Lucidchart, Canva, Draw.io) o infografías digitales funcionan de manera sobresaliente.
3. **Colores Sólidos:** Los colores planos estilo marcador producen un efecto de relleno muy realista.

---

## 📂 Estructura del Proyecto

```text
├── Infografia Proceso.jpg                             # Ejemplo de infografía de proceso
├── Circuito_de_gestión_de_turnos.consultorios...png  # Diagrama de flujo / circuito de atención
├── Guía__Trivias_Interactivas_con_IA.png              # Ejemplo de infografía interactiva
├── estructura seccion moodle.jpg                      # Diagrama de estructura educativa
├── placaredes educai.jpg                              # Pieza gráfica informativa
├── .gitignore                                         # Reglas para excluir binarios y videos
└── README.md                                          # Documentación completa del proyecto
```

---

## 📄 Licencia y Créditos

- Motor de animación: Basado en [`whiteboard-animator`](https://pypi.org/project/whiteboard-animator/) (Licencia MIT).
- Detección de texto: Basado en modelo ONNX CRAFT (*Character Region Awareness for Text Detection*).
