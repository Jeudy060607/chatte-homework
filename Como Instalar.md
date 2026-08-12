Guía de instalación paso a paso

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
