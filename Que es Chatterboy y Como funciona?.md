<p align="center">
  <img src="logo-itla.png" alt="ITLA - Instituto Tecnológico de las Américas" width="500">
</p>

<br>

<h1 align="center">Introducción a la Inteligencia Artificial</h1>

<br>

<p align="center">
  <strong>Estudiante:</strong> Jeudy José Rosario de Jesús
</p>

<p align="center">
  <strong>Docente:</strong> Luis Bessewell Feliz
</p>

<p align="center">
  <strong>Asignatura:</strong> Introducción a la Inteligencia Artificial
</p>

<br>

<p align="center">
  Instituto Tecnológico de las Américas (ITLA)
</p>

## 1. ¿Qué es el producto?

ChatterBot es un motor de diálogo conversacional (chatbot) escrito en Python que utiliza aprendizaje automático (machine learning) para generar respuestas a partir de colecciones de conversaciones previas. No depende de reglas fijas: aprende comparando la entrada del usuario con ejemplos que ya conoce y selecciona la respuesta estadísticamente más adecuada.

- **Repositorio oficial:** https://github.com/gunthercox/ChatterBot
- **Lenguaje de programación:** Python 3
- **Licencia:** BSD 3-Clause (código abierto, permite uso, copia, modificación y redistribución citando al autor original: Gunther Cox)
- **Dependencias principales:** spaCy (procesamiento de lenguaje natural), SQLAlchemy (almacenamiento en base de datos), tqdm

> **Nota sobre derechos de autor:** el código mostrado a continuación pertenece al repositorio oficial de ChatterBot (Copyright © Gunther Cox) y se reproduce bajo licencia BSD con fines educativos. Los comentarios marcados como `// ES:` fueron agregados por mi como documentación.

## 2. Arquitectura general (cómo funciona)

ChatterBot se organiza en tres piezas que trabajan juntas cada vez que el usuario envía un mensaje:

1. **Storage Adapter:** guarda y consulta el historial de conversaciones (por defecto usa una base de datos SQL).
2. **Tagger / Search:** convierte el texto en una forma que se pueda comparar (etiquetado gramatical con spaCy) y busca las frases más parecidas ya aprendidas.
3. **Logic Adapter (ej. BestMatch):** decide, entre las coincidencias encontradas, cuál respuesta enviar, calculando un nivel de confianza (confidence).

El flujo operativo, en orden, es: **recibir texto → preprocesar → buscar coincidencias → generar respuesta con nivel de confianza → guardar la conversación para seguir aprendiendo** (a menos que esté en modo `read_only`).

### Guía de instalación paso a paso

A continuación se detallan los pasos para instalar y probar ChatterBot en un equipo local.

1. **Verificar la versión de Python instalada** (se requiere 3.10 o superior, hasta 3.14):
   ```bash
   python --version
   ```
2. **Crear un entorno virtual** (recomendado, evita conflictos con otras librerías):
   ```bash
   python -m venv chatterbot_env
   ```
3. **Activar el entorno virtual:**
   - En Windows: `chatterbot_env\Scripts\activate`
   - En Linux/macOS: `source chatterbot_env/bin/activate`
4. **Instalar la librería desde PyPI:**
   ```bash
   pip install chatterbot
   ```
5. **Instalar el modelo de idioma de spaCy** necesario para el análisis de texto. En español:
   ```bash
   python -m spacy download es_core_news_sm
   ```
6. **Crear un script de prueba** (`prueba.py`) que importe `ChatBot` y `ListTrainer`, entrene al bot con una lista corta de frases y solicite una respuesta con `get_response()`.
7. **Ejecutar el script** con `python prueba.py` — en la primera ejecución se crea automáticamente el archivo `db.sqlite3` donde el bot guarda lo aprendido.

**Script de prueba mínimo:**

```python
from chatterbot import ChatBot
from chatterbot.trainers import ListTrainer

bot = ChatBot("MiBot")
trainer = ListTrainer(bot)
trainer.train([
    "Hola",
    "Hola, ¿cómo estás?",
    "Bien, gracias",
    "Me alegra escuchar eso"
])

respuesta = bot.get_response("Hola")
print(respuesta)
```

**Errores comunes:**

- **`'pip' no reconocido`:** usar `python -m pip install chatterbot` en su lugar.
- **Error de compatibilidad con spaCy:** confirmar que Python esté entre las versiones 3.10 y 3.14.
- **No encuentra el modelo de idioma:** verificar que el Paso 5 terminó sin errores antes de ejecutar el script.

 Resumen de la lógica operativa (para el diagrama de flujo)

1. El usuario envía un mensaje de texto al bot (`get_response`).
2. El texto se preprocesa (limpieza) y se etiqueta gramaticalmente con spaCy.
3. El motor de búsqueda compara el texto contra todas las frases ya aprendidas.
4. BestMatch identifica la frase más parecida (`closest_match`) y calcula un nivel de confianza.
5. Se buscan las respuestas históricas asociadas a esa frase; si no existen, se usa una respuesta por defecto.
6. Si hay varios adaptadores de lógica activos, se compara lo que proponen y gana la respuesta con más consenso o mayor confianza.
7. La respuesta se envía al usuario y, si el bot no está en modo `read_only`, la conversación completa se guarda para seguir aprendiendo.

## 5. Diagrama de flujo (lógica operativa del código)

Diagrama elaborado con Lucidchart (software de diagramas de flujo) a partir del análisis del código fuente documentado. Representa el recorrido completo de un mensaje dentro de ChatterBot: desde que el usuario escribe, hasta que el bot responde y aprende.

[Diagrama de flujo - Parte 1](https://lucid.app/lucidchart/a4b15380-1a93-48cc-907d-a1495b6267f0/edit?invitationId=inv_dbfe5370-2f79-4fc6-818a-d3105395570d)



