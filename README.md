# PROYECTO DE TERCER CORTE CORREGIDO
RODIAN GARAY Y MARIANA LOMBANA 


# Descripción General

El objetivo principal del primer módulo del proyecto es desarrollar un sistema automatizado capaz de obtener al menos 200 imágenes de distintas herramientas usadas en los laboratorios de ingeniería electrónica, tales como:
- Raspberry Pi
- Generador de señales
- Osciloscopio
- Fuente dual
- Destornillador
- Pinzas
- Condensador
- Transistor
- Bombilla

Este conjunto de imágenes servirá como base de datos visual para las siguientes fases del proyecto (ETL, clasificación y despliegue).
Para asegurar un alto rendimiento, el sistema usa:
- Hilos (threads) para ejecutar múltiples búsquedas en paralelo
- Semáforo para controlar cuántos navegadores se abren al mismo tiempo
- Mutex (Lock) para evitar errores al escribir archivos en disco
- Selenium + WebDriver Manager para realizar búsquedas reales en Mercado Libre

# Arquitectura del Sistema de Scraping

El sistema está construido bajo un modelo de concurrencia que coordina:
- Un hilo principal que organiza las tareas
- Varios hilos trabajadores que realizan el scraping
- Un mecanismo bloqueante que protege las descargas
- Cada hilo procesa un producto, abre un navegador (si el semáforo lo permite), obtiene enlaces de imágenes y los envía al descargador. Este último guarda los archivos asegurando que no ocurran colisiones.

```mermaid
flowchart TD
    CORE((Motor de Scraping))

    CTRL[Hilo Controlador]
    EXEC[Hilos Ejecutores x10]
    SAFE[Modulo de Descarga Segura]

    A1[Genera listado de categorías]
    A2[Coordina y distribuye tareas]

    B1[Solicita acceso por semáforo]
    B2[Inicia navegador Selenium]
    B3[Recolecta enlaces de imágenes]
    B4[Entrega enlaces al módulo de descarga]

    C1[Realiza descarga HTTP]
    C2[Usa Lock para evitar conflictos]
    C3[Organiza y almacena archivos]

    CORE --> CTRL
    CORE --> EXEC
    CORE --> SAFE

    CTRL --> A1
    CTRL --> A2

    EXEC --> B1
    EXEC --> B2
    EXEC --> B3
    EXEC --> B4

    SAFE --> C1
    SAFE --> C2
    SAFE --> C3

    CTRL --> EXEC --> SAFE

```


##  Tecnologías Utilizadas


| Tecnología            | Función                                      |
| --------------------- | -------------------------------------------- |
| **Python 3**          | Base lógica del programa                     |
| **Selenium**          | Captura de imágenes mediante navegación real |
| **WebDriver Manager** | Administra el driver de Chrome               |
| **Requests**          | Descarga de archivos                         |
| **Threads**           | Procesamiento paralelo                       |
| **Semaphore**         | Control de navegadores abiertos              |
| **Lock/Mutex**        | Protección en escritura a disco              |


# Modelo de Concurrencia
## Hilos
Cada producto se procesa en un hilo independiente, lo que permite descargar imágenes simultáneamente.
## Semáforo
Para evitar abrir demasiados navegadores a la vez, solo se permiten 3 Chrome simultáneos para no afectar la ram:
```
browser_semaphore = threading.Semaphore(3)
```
## Mutex

Cuando varios archivos se descragar simultaneamente se puede generar archovos corrupos, colisiones o directorios bloqueados para eso, solo un hilo puede escribir en disco para evitar daños o conflictos:
```
with file_lock:
    with open(filename, "wb") as f:
        f.write(img.content)
```
## Estructura Final 

El scraping genera una carpeta:
```
scraping/images/
    raspberry/
    osciloscopio/
    generador_de_senales/
    transistor/
    bombilla/
    ...
```

# Código utilizado para el scraping

```
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
import threading
import os
import time
import requests

# ============================
# CONFIGURACIÓN GENERAL
# ============================
BASE_DIR = "scraping/images/"
os.makedirs(BASE_DIR, exist_ok=True)

productos = [
    "multimetro", "raspberry", "generador de señales", "osciloscopio",
    "fuente dual", "destornillador", "pinzas", "condensador",
    "transistor", "bombilla"
]

file_lock = threading.Lock()
browser_semaphore = threading.Semaphore(3)  # 3 navegadores máximo

# ============================
# DRIVER
# ============================
def iniciar_driver():
    opt = webdriver.ChromeOptions()
    opt.add_argument("--headless")
    opt.add_argument("--window-size=1920,1080")
    opt.add_argument("--disable-dev-shm-usage")
    opt.add_argument("--no-sandbox")
    return webdriver.Chrome(
        service=Service(ChromeDriverManager().install()),
        options=opt
    )

# ============================
# DESCARGA SEGURA
# ============================
def descargar_imagen(url, destino):
    try:
        contenido = requests.get(url, timeout=5).content

        with file_lock:
            with open(destino, "wb") as archivo:
                archivo.write(contenido)

    except:
        pass

# ============================
# FUNCIÓN PRINCIPAL DEL HILO
# ============================
def scrapear(producto):
    with browser_semaphore:

        driver = iniciar_driver()
        driver.get(f"https://listado.mercadolibre.com.co/{producto}")
        time.sleep(3)

        altura = driver.execute_script("return document.body.scrollHeight")
        for _ in range(6):
            driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
            time.sleep(1.8)
            nueva_altura = driver.execute_script("return document.body.scrollHeight")
            if nueva_altura == altura:
                break
            altura = nueva_altura

        carpeta_producto = os.path.join(BASE_DIR, producto.replace(" ", "_"))
        os.makedirs(carpeta_producto, exist_ok=True)

        imagenes = driver.find_elements(By.TAG_NAME, "img")
        contador = 0

        for imagen in imagenes:
            src = (
                imagen.get_attribute("src")
                or imagen.get_attribute("data-src")
                or imagen.get_attribute("srcset")
            )

            if src and "http" in src:
                if "srcset" in src:
                    src = src.split(" ")[0]

                destino = os.path.join(carpeta_producto, f"{producto}_{contador}.jpg")
                descargar_imagen(src, destino)
                contador += 1

            if contador >= 200:
                break

        driver.quit()
        print(f"✔ {producto} → {contador} imágenes descargadas.")

# ============================
# HILOS
# ============================
hilos = []
for prod in productos:
    t = threading.Thread(target=scrapear, args=(prod,))
    t.start()
    hilos.append(t)

for t in hilos:
    t.join()

print("\nFINALIZADO\n")

``` 
Este script paso a paso :

 1. Abre Mercado Libre
 2. Busca cada producto
 3. Descarga hasta 200 imágenes por categoría
 4. Crea carpetas automáticamente
 5. Usa Selenium + Hilos de forma profesional

# Estructura de Salida del Scraping

Una vez ejecutado, automáticamente se genera la carpeta y enumera cada imagen en orden de esta forma:
```
scraping/
│
└── images/
    ├── raspberry/
    │     ├── img_001.jpg
    │     ├── img_002.jpg
    │     └── ...
    ├── osciloscopio/
    ├── generador de señales/
    ├── transistor/
    ├── bombilla/
    └── ...
```

Cada carpeta contiene 200 imágenes limpias obtenidas desde la web.

# Ejecucion paso a paso 

## 1. Ingresar a powershell 
 <img width="955" height="502" alt="1" src="https://github.com/user-attachments/assets/8dcdff8c-97e0-44c8-8e2a-77dd1d24050e" />
## 2. Se crea un entorno virtual 
 <img width="959" height="502" alt="2" src="https://github.com/user-attachments/assets/c820dafe-d584-45d0-9752-a07e490be93a" />
## 3. Se instalan librerias necesarias 
 <img width="1895" height="804" alt="image" src="https://github.com/user-attachments/assets/6c150a27-0c93-4106-b4a4-3a1682168b20" />
## 4. Se ejecuta el archivo scraper
 <img width="817" height="545" alt="image" src="https://github.com/user-attachments/assets/489c66b9-811d-4cc4-bd5b-72e75f839f11" />
## 5. Se crea una carpeta llamada imagenes con todas las carpetas de imagemes
 <img width="1919" height="1010" alt="image" src="https://github.com/user-attachments/assets/97807015-3243-4adb-b489-c01b50cf05f2" />
## 6. Cada una cuenta con 200 imagenes como se evidencia en esta:
 <img width="1919" height="1005" alt="image" src="https://github.com/user-attachments/assets/61e71216-071f-4895-9b62-8579229b2a96" />

# Conclusión del Punto 1

El sistema desarrollado cumple todos los requerimientos establecidos:

- Web Scraping con Selenium
- Búsqueda de más de 10 elementos electrónicos
- Descarga masiva de más de 200 imágenes por categoría
- Uso explícito y correcto de:

Hilos

Sección crítica

Semáforo

Mutex

- Arquitectura profesional lista para ETL, clasificación e integración en Docker
- Documentación clara y técnica para evaluación académica

Este punto es la base del proyecto completo, permitiendo construir la base de imágenes que alimentará el modelo de clasificación (punto 2) y el sistema de detección en tiempo real (puntos 3 y 4).

# PUNTO 2 — Desarrollo Completo del ETL (Extracción, Transformación y Carga)

El objetivo del Punto 2 es crear una Base de Datos completamente funcional, diseñada para almacenar las 200 imágenes por clase obtenidas en el scraping, manteniendo orden, trazabilidad y soporte para el modelo de clasificación y demás puntos del proyecto.
## Este apartado documenta:
- Diseño del modelo relacional
- Creación de la base de datos
- Tablas y relaciones
- Carga masiva automatizada de las imágenes
- Evidencias del correcto funcionamiento
- Estructura final del sistema

# 2.1. Objetivo del Módulo de Base de Datos

La base de datos debe permitir:
- Registrar cada clase (raspberry, osciloscopio, etc.).
- Registrar las 200 imágenes procesadas por clase.
- Guardar metadatos útiles para el modelo (ruta, tamaño, hash, fecha).
- Evitar duplicados.
- Integrarse con el ETL y con el clasificador.
- Consultar fácilmente el dataset completo.

Para este proyecto se usó SQLite porque:

- No requiere servido
- Es portable
- Funciona en cualquier entorno, incluyendo Docker}
- Perfecto para datasets ligeros

# 2.2. Modelo Relacional de la Base de Datos

El sistema se basa en dos tablas principales:

## 1. Tabla clases

Contiene las categorías del dataset.

| Campo       | Tipo        | Descripción                          |
| ----------- | ----------- | ------------------------------------ |
| id          | INTEGER PK  | ID único                             |
| nombre      | TEXT UNIQUE | Nombre de la clase (ej: "raspberry") |
| descripcion | TEXT        | Descripción opcional                 |

## 2. Tabla imagenes

Registra cada imagen del scraping o del ETL.

| Campo          | Tipo       | Descripción                     |
| -------------- | ---------- | ------------------------------- |
| id             | INTEGER PK | ID único                        |
| clase_id       | INTEGER FK | Relación con clase              |
| ruta           | TEXT       | Ruta del archivo en el sistema  |
| hash           | TEXT       | Hash MD5 para evitar duplicados |
| ancho          | INTEGER    | Ancho en px                     |
| alto           | INTEGER    | Alto en px                      |
| fecha_registro | TEXT       | Fecha de inserción              |

## Diagrama relacional
```
clases (1)  -------------------  (N) imagenes
      id   <------------------   clase_id
```
# 2.3. Creación de la Base de Datos
## Se crea la carpeta para la pase de datos 
<img width="1105" height="67" alt="image" src="https://github.com/user-attachments/assets/cfa3d7db-8633-48ac-99cc-f7a204cc9744" />
## Se ingresa a la carpeta y se verifica que este vacia 
<img width="1237" height="287" alt="image" src="https://github.com/user-attachments/assets/cd96f58a-9bbf-425f-9527-6996b09239bb" />
## Se crea el archivo dataset para crear las tablas 
<img width="1082" height="130" alt="image" src="https://github.com/user-attachments/assets/f75713ff-2319-44ca-9c21-8275f26aa8ac" />
## Se crea base de datos
<img width="1305" height="78" alt="image" src="https://github.com/user-attachments/assets/a1afc74f-38b7-4cb7-a9bd-633d383ac86f" />
```
import sqlite3
conn = sqlite3.connect("dataset.db")
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS clases (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nombre TEXT UNIQUE NOT NULL,
    descripcion TEXT
);
""")

cur.execute("""
CREATE TABLE IF NOT EXISTS imagenes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    clase_id INTEGER NOT NULL,
    ruta TEXT NOT NULL,
    hash TEXT NOT NULL UNIQUE,
    ancho INTEGER,
    alto INTEGER,
    fecha_registro TEXT,
    FOREIGN KEY(clase_id) REFERENCES clases(id)
);
""")

conn.commit()
conn.close()
print("✔ Base de datos creada exitosamente")

```
## Se crea las clases 
<img width="1313" height="386" alt="image" src="https://github.com/user-attachments/assets/21fd71f9-4b8d-4b79-a429-af1408885406" />
```
import sqlite3

DB_PATH = "dataset.db"

clases = [
    ("bombilla", "Dispositivo de iluminación"),
    ("condensador", "Componente que almacena energía"),
    ("destornillador", "Herramienta manual para tornillos"),
    ("fuente_dual", "Fuente de alimentación dual"),
    ("generador_de_senales", "Generador de señales electrónica"),  # SIN Ñ
    ("multimetro", "Instrumento de medición eléctrica"),
    ("osciloscopio", "Instrumento para visualizar señales"),
    ("pinzas", "Herramienta para manipular componentes"),
    ("raspberry", "Computadora de placa reducida"),
    ("transistor", "Componente semiconductor de tres terminales")
]

conn = sqlite3.connect(DB_PATH)
cur = conn.cursor()

print("Insertando clases en la base de datos...\n")

for clase, desc in clases:
    try:
        cur.execute("INSERT INTO clases (nombre, descripcion) VALUES (?, ?)", (clase, desc))
        print(f"✔ Clase insertada: {clase}")
    except sqlite3.IntegrityError:
        print(f"⚠ Clase ya existía: {clase}")

conn.commit()
conn.close()

print("\n✔ Inserción completa")

```
## Se crea la carga de imagenes 
<img width="1918" height="723" alt="image" src="https://github.com/user-attachments/assets/3160e5a4-b3a6-4a9d-a69c-b474b6e8f00b" />
```
import sqlite3
import os
import cv2
import hashlib
from datetime import datetime

# ==============================
# CONFIGURACIÓN
# ==============================

DB_PATH = "dataset.db"

# Ruta con tus 10 carpetas de imágenes
IMAGES_ROOT = r"C:/Users/Lenovo/Desktop/Proyecto final 2/1. Primer punto/scraping/images/"

# ==============================
# FUNCIONES AUXILIARES
# ==============================

def calcular_hash(path):
    """Genera hash MD5 único por imagen."""
    try:
        with open(path, "rb") as f:
            return hashlib.md5(f.read()).hexdigest()
    except:
        return None


def get_image_size(path):
    """Obtiene dimensiones de la imagen."""
    try:
        img = cv2.imread(path)
        if img is None:
            return None, None
        h, w = img.shape[:2]
        return w, h
    except:
        return None, None

# ==============================
# CONEXIÓN A LA BD
# ==============================

conn = sqlite3.connect(DB_PATH)
cur = conn.cursor()

print("📌 Conectado a la base de datos")

cur.execute("SELECT id, nombre FROM clases")
clases_db = {nombre: cid for cid, nombre in cur.fetchall()}

print("📌 Clases detectadas:", clases_db)

# ==============================
# RECORRER TODAS LAS CARPETAS
# ==============================

insertados_total = 0

for clase_nombre, clase_id in clases_db.items():
    carpeta = os.path.join(IMAGES_ROOT, clase_nombre)

    if not os.path.exists(carpeta):
        print(f"⚠ La carpeta '{carpeta}' no existe. Saltando...")
        continue

    print(f"\n🔎 Procesando clase: {clase_nombre}")

    for archivo in os.listdir(carpeta):
        ruta_img = os.path.join(carpeta, archivo)

        if not ruta_img.lower().endswith((".jpg", ".jpeg", ".png")):
            continue

        hash_img = calcular_hash(ruta_img)

        if hash_img is None:
            print(f"❌ Error leyendo: {ruta_img}")
            continue

        w, h = get_image_size(ruta_img)
        fecha = datetime.now().isoformat()

        try:
            cur.execute("""
                INSERT INTO imagenes (clase_id, ruta, hash, ancho, alto, fecha_registro)
                VALUES (?, ?, ?, ?, ?, ?)
            """, (clase_id, ruta_img, hash_img, w, h, fecha))

            insertados_total += 1

        except sqlite3.IntegrityError:
            print(f"⚠ Imagen duplicada: {ruta_img}")
            continue

conn.commit()
conn.close()

print("\n✔ PROCESO FINALIZADO ✔")
print(f" Total de imágenes insertadas: {insertados_total}")

```

## 2.3.1. ¿Cómo detecta la aplicación si una imagen es buena o mala?
Tu aplicación considera que una imagen es buena cuando:
- La imagen puede abrirse
Usamos esta línea:
```
img = cv2.imread(path)
```

Si img es None, OpenCV no pudo leerla → imagen dañada o no válida.

- La imagen tiene ancho y alto

Si cv2.imread() sí logra leerla:
```
h, w = img.shape[:2]
```

Si shape no existe, la imagen es mala.

🔴 ¿Qué ocurre con una imagen mala o dañada?

cv2.imread() devuelve None

El programa muestra:

❌ Error leyendo: ruta_de_la_imagen


Esa imagen no se inserta en la base de datos

Por eso tu BD queda limpia: solo imágenes válidas entran.

✅ 2. ¿Cómo detecta la aplicación imágenes repetidas?

El sistema usa un hash MD5, que es una especie de “huella digital” única generada a partir del contenido de la imagen.

En el código:

hashlib.md5(f.read()).hexdigest()


Esto significa:

Si 2 imágenes son idénticas, su MD5 será exactamente igual

Si una imagen cambia 1 solo pixel, tendrá un hash diferente

🔍 ¿Cómo se evita insertar duplicados?

En la tabla imagenes, definimos:

hash TEXT NOT NULL UNIQUE


Eso significa que no se pueden repetir hashes.

Cuando el script intenta insertar una imagen repetida:

cur.execute(...)


SQLite encuentra que el hash ya existe → lanza:

sqlite3.IntegrityError


Y nuestro código captura esto:

print(f"⚠ Imagen duplicada: {ruta_img}")


y NO inserta la imagen duplicada.



  
# Arquitectura General del ETL

El pipeline se divide en 3 módulos principales:

Extracción → obtención de imágenes desde el directorio de scraping.

Transformación → limpieza, validación y preprocesamiento de cada imagen.

Carga → almacenamiento ordenado en directorios por clase + exportación de metadat

# 2.1. Módulo de Extracción

Objetivo: Leer todas las imágenes descargadas en el punto 1 y validarlas previo al procesamiento.

 Se recorren las carpetas generadas por el scraping:

scraping/images/<nombre_de_clase>/


 Se cuentan todas las imágenes de cada categoría.
 Se detectan archivos dañados mostrando advertencias como:

 # 2.2. Módulo de Transformación

Este módulo ejecuta:

 1. Limpieza y validación

Se intenta abrir cada imagen con OpenCV.

Si falla → se descarta automáticamente.

Se evita cargar imágenes con tamaño incorrecto o corruptas.

# 2.3 Estandarización

Cada imagen se transforma mediante:

Redimensionamiento a 224×224 px.

Normalización de valores entre 0 y 1.

Conversión a formato .npy optimizado para ML.

Generación de un hash MD5 por imagen
Esto permite:

Detectar duplicados

Evitar procesar imágenes repetidas

Mantener un dataset limpio y sin ruido

 Aquí insertas la imagen donde se muestran los duplicados omitidos:

Ejemplo del mensaje real:
 ```python
 Duplicado omitido (hash repetido): etl/data/processed/transistor/transistor_90.jpg.npy
```

# 3. Manejo multihilo (threads)

Para acelerar el proceso, cada clase se procesa en paralelo:

Un hilo por categoría (raspberry, osciloscopio, fuente, etc.)

Mutex para operaciones críticas

Sincronización de escritura en disco

Esto reduce radicalmente el tiempo total del ETL.

# 2.3. Módulo de Carga

Luego de validar y transformar todas las imágenes:

 Se guardan las imágenes limpias en:
etl/data/processed/<nombre_de_clase>/

 Se registran estadísticas globales:

Número de imágenes finales por categoría

Total final del dataset

Número de duplicados eliminados

Imágenes descartadas por corrupción

 Aquí insertas tu imagen donde se ve la finalización con total 1850 imágenes:

Ejemplo real:

 Clase 'transistor' cargada correctamente.
 Carga finalizada. Total imágenes registradas: 1850

# Estructura final generada

etl/
 ├── data/
 │    ├── raw/
 │    ├── processed/
 │    │     ├── raspberry/
 │    │     ├── osciloscopio/
 │    │     ├── transistor/
 │    │     ├── generador_de_señales/
 │    │     └── …
 │    └── metadata.json
 ├── etl_extract.py
 ├── etl_transform.py
 └── etl_load.py

 # Código del ETL (resumen técnico)
 Transformación (etl_transform.py)
  ```python
# Procesamiento: resize, normalización y hash
image = cv2.resize(image, (224, 224))
image = image.astype("float32") / 255.0

hash_value = hashlib.md5(image.tobytes()).hexdigest()

output_path = f"{processed_dir}/{filename}.npy"
np.save(output_path, image)
 ```

 Carga (etl_load.py)
 ```python
if hash_value in hash_registry[class_name]:
    print(f"⚠️ Duplicado omitido (hash repetido): {output_file}")
else:
    hash_registry[class_name].add(hash_value)
    np.save(output_file, image)
 ```

 Conclusiones del ETL

El dataset quedó completamente limpio, sin archivos corruptos.

Se eliminaron centenas de imágenes duplicadas mediante hashing.

El proceso final contiene 1850 imágenes válidas, perfectas para entrenamiento.

La arquitectura ETL es profesional, modular y escalable, lista para integrarse con el modelo del Punto 3.

# Imágenes

<img width="876" height="437" alt="image" src="https://github.com/user-attachments/assets/1e53dcbe-7128-4060-a0f9-eaa577819b9b" />










# PUNTO 3 — Sistema de Clasificación de Objetos + Detección y Velocidad de Personas (Modelo Simple + OpenCV HOG + Multithreading)

El tercer punto del proyecto implementa un sistema completo de visión artificial en tiempo real que:

Clasifica herramientas de laboratorio usando un modelo propio entrenado con el dataset del ETL.

Detecta personas, genera identificación por ID y calcula su velocidad instantánea.

Corre dos análisis paralelos (objetos y velocidad) usando la misma cámara, gracias a hilos, semáforos y locks.

Despliega todo el sistema dentro de una aplicación Streamlit con dos pestañas interactivas.

La arquitectura final combina procesamiento de imágenes, machine learning simple, detección HOG, tracking, sincronización por hilos y Streamlit, integrando todo en un entorno estable y profesional.

#  Arquitectura General del Punto 3

El sistema está compuesto por 3 hilos principales:

CamGrabber → productor de frames (un solo hilo para toda la app)

PredictorThread → clasifica herramientas con el modelo entrenado

PeopleSpeedThread → detecta personas, les asigna ID y calcula velocidad

Cada uno opera de forma independiente, pero sincronizados mediante:

Locks → para acceso seguro a los frames compartidos

Semáforos → para controlar el procesamiento simultáneo de predicciones

Threads Daemon → para ejecutar tareas en paralelo sin bloquear la UI

# 3.1. Captura de Cámara – Hilo CamGrabber

Este hilo es el corazón de la app:
- captura continuamente los frames de la webcam
- los entrega al predictor y al módulo de velocidad
  - calcula FPS en tiempo real
 ```python
self.lock = threading.Lock()
self.frame = None

ret, frame = self.cap.read()
with self.lock:
    self.frame = frame.copy()
 ```

Ventajas:

Evita capturas duplicadas

Garantiza que todos los hilos usan el mismo frame sincronizado

Minimiza consumo de CPU y evita retardos


 <img width="730" height="525" alt="image" src="https://github.com/user-attachments/assets/b2dfa864-5b3c-47db-bee8-916d6e451bd3" />


# 3.2. Clasificación de Objetos – Hilo PredictorThread

Este módulo usa el modelo entrenado en el Punto 2:

modelo lineal con pesos W y b

extracción de características manual con kernels

clasificación sin umbral (siempre muestra la predicción)

 Pipeline del predictor:

Convertir frame a escala de grises

Redimensionar a 256×256

Normalizar

Extraer características mediante 3 kernels

Aplicar pooling

Normalizar vector de características

Multiplicar por W y sumar b

Aplicar softmax

Ejemplo de predicción overlay:
```python

text = f"{pred_t.pred} ({pred_t.conf:.2f})"
cv2.putText(frame_obj, text, (20,50), ...)

```
<img width="619" height="823" alt="image" src="https://github.com/user-attachments/assets/1a64d7f0-a8c7-4bcb-8e7f-f466677b3ddc" />
<img width="663" height="813" alt="image" src="https://github.com/user-attachments/assets/412e8f0d-a06c-448b-a82a-86f622607dc4" />
<img width="632" height="737" alt="image" src="https://github.com/user-attachments/assets/c2bc143b-eeb9-443a-b4d1-5e75cdf60785" />


3.3. Detección de Personas + Velocidad – Hilo PeopleSpeedThread

Este módulo implementa:

Detector HOG de OpenCV (cv2.HOGDescriptor)

Asignación de IDs por cercanía de centroides

Memoria de últimos frames

Cálculo de velocidad por persona

Eliminación de tracks desaparecidos

##  Cálculo de Velocidad del Objeto

La velocidad se calcula como:

**velocidad (px/s) = Δdistancia(px) / Δtiempo(s)**

Donde:

- **Δdistancia(px)** = diferencia del centroide entre dos frames.
- **Δtiempo(s)** = tiempo transcurrido entre capturas consecutivas.

### Fórmula utilizada

**distancia_px = √((x₂ - x₁)² + (y₂ - y₁)²)**  
**velocidad = distancia_px / dt**

### Implementación en Python

```python
dist_px = ((cx - prev["centroid"][0])**2 + (cy - prev["centroid"][1])**2)**0.5
speed = dist_px / dt

```
# 3.4. Multithreading: Locks + Semáforos
Locks

Usados para evitar corrupción de memoria y lectura simultánea del frame.
```python
with self.lock:
    self.frame = frame.copy()
```
 Semáforos

Controlan cuántas predicciones simultáneas puede hacer el predictor.
```python
self.sema = threading.Semaphore(1)
```

Esto evita:

saturación del CPU

bloqueos en la lectura de la cámara

predicciones repetidas sobre el mismo frame

# 3.5. Interfaz Streamlit con Pestañas Dinámicas

La app presenta dos vistas en tiempo real:

 Pestaña 1: “Objetos”

Muestra predicción del modelo

Superpone la etiqueta y la confianza

Muestra FPS del sistema

Pestaña 2: “Velocidad”

Muestra bounding boxes de cada persona

Dibuja centroides

Muestra velocidad en px/s

# 3.6. Flujo Completo del Sistema

```mermaid
flowchart TD

    %% Nodos principales
    CAM[Camara]
    GRAB[CamGrabber - Frame Compartido]

    PRED[PredictorThread - Clasificacion]
    SPEED[PeopleSpeedThread - Tracking y Velocidad]

    UI[Streamlit UI]

    %% Indicador de Tracks Activos
    TRACKS[Tracks Activos]

    %% Flujo
    CAM --> GRAB

    GRAB --> PRED
    GRAB --> SPEED

    PRED --> UI
    SPEED --> UI

    SPEED --> TRACKS


```

Conclusiones del Punto 3

Se implementó un sistema completo de visión artificial en tiempo real.

Se integraron dos modelos paralelos:

Clasificación de herramientas

Detección + velocidad de personas

Los hilos funcionan de manera segura mediante locks y semáforos.

La interfaz en Streamlit es clara, funcional y permite alternar entre vistas sin detener la cámara.

Sistema apto para laboratorios inteligentes, robótica o vigilancia.












# 4. Despliegue de la Aplicación (Docker + Streamlit WebApp)

Este proyecto fue completamente contenedorizado, ejecutado y desplegado usando Docker y Streamlit, cumpliendo todos los requisitos del cuarto punto del entregable. A continuación se muestra el procedimiento completo.

#  4.1. Construcción del contenedor Docker

El proyecto incluye un Dockerfile totalmente funcional.
Para construir la imagen localmente:
```python
docker build -t streamlit-detector .

```
Una vez finalizada la compilación, confirmar que la imagen existe:
```python
docker images
```

 La imagen debe aparecer como streamlit-detector.

 <img width="1280" height="650" alt="image" src="https://github.com/user-attachments/assets/c155f91d-423f-4be8-b6ae-cb07151bbf1b" />


 # 4.2. Ejecución local del contenedor

Para ejecutar el contenedor en tu propio equipo, usé el siguiente comando:

docker run -p 8501:8501 --name visionapp streamlit-detector


Luego, abrir en el navegador:

 http://localhost:8501

Donde se cargan simultáneamente:

- Detector de Velocidad (MediaPipe + tracking)

- Detector de Objetos (modelo simple entrenado)

- Interfaz Streamlit con ambas vistas lado a lado

### 4.3. Despliegue de la imagen en Docker Hub

La imagen final fue subida al repositorio público:

Docker Hub:
 https://hub.docker.com/r/jefersonmvp/streamlit-detector

Para descargarla y ejecutarla desde cualquier equipo:
```python
docker pull jefersonmvp/streamlit-detector
docker run -p 8501:8501 jefersonmvp/streamlit-detector
```

#  4.4. Despliegue de la aplicación vía Streamlit Web

La aplicación también se despliega vía Streamlit Web, permitiendo acceso desde navegador sin instalación local:

Contiene:

Interfaz doble (Velocidad + Objetos)

Hilos independientes

FPS en tiempo real

Sincronización entre pipelines

Procesamiento simultáneo por la misma cámara

(https://hub.docker.com/r/jefersonmvp/streamlit-detector)

# 4.5. Evidencias del despliegue
🔧 Ejecución correcta del contenedor

<img width="1280" height="655" alt="image" src="https://github.com/user-attachments/assets/47f3da5f-094b-4f9a-a61a-07af2699ccd2" />

 Streamlit funcionando con doble vista

<img width="1278" height="855" alt="image" src="https://github.com/user-attachments/assets/71bce428-0eb8-4870-980e-3118a7bba636" />

<img width="1280" height="655" alt="image" src="https://github.com/user-attachments/assets/e72dd60d-f3a1-46ea-8d33-16bcd378cc01" />

 Detector de Objetos funcionando

<img width="1280" height="574" alt="image" src="https://github.com/user-attachments/assets/2770e177-0901-4268-83f2-b606ad2080eb" />

<img width="1280" height="554" alt="image" src="https://github.com/user-attachments/assets/5401add8-41dc-467a-88cc-d0fe7c2eba9e" />

<img width="1280" height="606" alt="image" src="https://github.com/user-attachments/assets/59060e1d-bb4a-475e-870f-ae1de67f5649" />

<img width="1280" height="621" alt="image" src="https://github.com/user-attachments/assets/4534a711-4fd3-4616-8527-ced27c50efb7" />

<img width="1280" height="598" alt="image" src="https://github.com/user-attachments/assets/70ba03c7-0d02-4238-9f6c-a810ccc9c485" />

<img width="1280" height="574" alt="image" src="https://github.com/user-attachments/assets/967fddb4-cb3f-4e42-a7c1-76accb72dcae" />

<img width="1280" height="641" alt="image" src="https://github.com/user-attachments/assets/72aad00c-4cb7-46a1-bf75-d13222009a86" />

<img width="1280" height="596" alt="image" src="https://github.com/user-attachments/assets/af524f05-e7d2-4f8c-bdfa-3b1d60e97541" />

<img width="1280" height="636" alt="image" src="https://github.com/user-attachments/assets/64f39c9f-a317-4861-a963-a2b6b4036e14" />


Detector de Velocidad funcionando

<img width="1176" height="864" alt="image" src="https://github.com/user-attachments/assets/be4cc872-133c-4071-9db4-f819dee178e8" />

<img width="1278" height="855" alt="image" src="https://github.com/user-attachments/assets/e6875ffc-49e9-4fec-8246-c598ee9faff4" />


### 4.6. Conclusiones del despliegue

El proyecto es completamente portable gracias a Docker.

La aplicación puede ejecutarse sin dependencias en cualquier máquina.

El código integra simultáneamente dos sistemas avanzados de visión por computador en producción.

La documentación y el despliegue cumplen todos los requisitos del punto 4 del entregable.
