> **Nota sobre derechos de autor:** el código mostrado a continuación pertenece al repositorio oficial de ChatterBot (Copyright © Gunther Cox) y se reproduce bajo licencia BSD con fines educativos. Los comentarios marcados como `// ES:` fueron agregados por mi estudiante como documentación.

## 7. Documento descriptivo del producto (Punto 3)

A continuación se narra todo lo relacionado al producto ChatterBot: lenguaje de programación y versión, plataformas en que opera, sistemas operativos compatibles, manejo de bases de datos y archivos, y otra documentación descriptiva del proyecto.

### 7.1 Lenguaje de programación y versión

ChatterBot está escrito completamente en Python 3. Según la configuración oficial del proyecto (`pyproject.toml`), la versión actual de la librería es la 1.2.14, y requiere Python 3.10 o superior (hasta 3.14). No es compatible con Python 2 ni con versiones antiguas de Python 3.

### 7.2 Plataforma(s) en que opera

Es una librería de backend (no una aplicación de escritorio con interfaz propia). Se ejecuta como parte de un programa Python: puede integrarse en una aplicación de consola, un bot de Telegram/Discord, una API web (por ejemplo con Flask o Django), o cualquier sistema que pueda importar el paquete `chatterbot` e invocar sus funciones. El propio proyecto incluye una extensión oficial para integrarse con Django (`chatterbot.ext.django_chatterbot`).

### 7.3 Sistema(s) operativo(s)

Es multiplataforma ("OS Independent", según su propia clasificación oficial en PyPI). Al estar escrito en Python puro, funciona igual en:

- **Windows:** 10/11, con Python instalado desde python.org o Microsoft Store.
- **Linux:** cualquier distribución con Python 3.10+ (Ubuntu, Debian, Fedora, etc.), como el entorno usado en este trabajo.
- **macOS:** compatible igualmente instalando Python 3.10+.

### 7.4 Base de datos / manejo de archivos

ChatterBot no depende de un único motor de base de datos: usa "adaptadores de almacenamiento" (storage adapters) intercambiables, ubicados en `chatterbot/storage/`:

- **SQLStorageAdapter (por defecto):** usa SQLAlchemy. Si no se especifica nada, crea automáticamente un archivo local SQLite (`db.sqlite3`); también puede conectarse a PostgreSQL, MySQL u otro motor SQL cambiando el `database_uri`.
- **MongoStorageAdapter:** permite guardar el conocimiento del bot en una base de datos MongoDB (NoSQL).
- **RedisStorageAdapter:** permite usar Redis como almacenamiento en memoria/caché.
- **DjangoStorageAdapter:** usa el ORM de Django, para integrarse con proyectos Django ya existentes.

Además del almacenamiento de conocimiento, el sistema puede exportar e importar datos de entrenamiento en archivos JSON y leer datos de entrenamiento propios en formato CSV, TSV o JSON (ver `GenericFileTrainer`, `CsvFileTrainer` y `JsonFileTrainer`), lo cual documentamos en la sección de código fuente.

### 7.5 Otras dependencias relevantes

- **spaCy (3.8.x):** librería de procesamiento de lenguaje natural (NLP) usada para analizar gramaticalmente el texto (lematización, etiquetado POS).
- **SQLAlchemy (2.0.x):** capa de acceso a bases de datos SQL.
- **python-dateutil:** para interpretar fechas dentro de los datos de entrenamiento.
- **tqdm:** para mostrar barras de progreso durante el entrenamiento.
- **mathparse:** permite al bot entender y resolver expresiones matemáticas simples escritas en lenguaje natural.

### 7.6 Otra documentación descriptiva del proyecto

- **Licencia:** BSD 3-Clause (código abierto, permite uso académico y comercial citando al autor original).
- **Autor / mantenedor:** Gunther Cox
- **Repositorio:** https://github.com/gunthercox/ChatterBot
- **Documentación oficial:** https://docs.chatterbot.us
- **Instalación:** `pip install chatterbot` (junto con el modelo de idioma de spaCy correspondiente, por ejemplo: `python -m spacy download es_core_news_sm` para español).
- **Estado del proyecto:** Beta ("Development Status :: 4 - Beta", según su clasificación oficial), en desarrollo activo y mantenimiento por la comunidad.


## 8. Referencia

Código fuente original: [github.com/gunthercox/ChatterBot](https://github.com/gunthercox/ChatterBot) — Copyright (c) 2016-2025, Gunther Cox. Distribuido bajo licencia BSD 3-Clause. Los comentarios en español (`// ES:`) de este documento fueron elaborados por el estudiante como parte del Trabajo de Investigación No. 3.
