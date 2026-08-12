> **Nota sobre derechos de autor:** el código mostrado a continuación pertenece al repositorio oficial de ChatterBot (Copyright © Gunther Cox) y se reproduce bajo licencia BSD con fines educativos. Los comentarios marcados como `// ES:` fueron agregados por mi estudiante como documentación.

## 3. Código fuente documentado

### 3.1 `chatterbot.py` — Clase principal ChatBot

Este archivo define la clase `ChatBot`, el punto de entrada de toda la librería. Se muestran el constructor (`__init__`) y el método `generate_response`, que es el que decide qué respuesta enviar.

**📄 Archivo: `chatterbot.py`** *(chatterbot/chatterbot.py)*

```python
# ES: Este bloque construye el chatbot: prepara el almacenamiento, el
# ES: 'tagger' (analizador de texto) y los adaptadores de lógica.
class ChatBot(object):
    def __init__(self, name, stream=False, **kwargs):
        self.name = name
        self.stream = stream

        # ES: Cada bot recibe un ID único de conversación por defecto,
        # ES: usado si quien lo llama no especifica uno propio.
        self.default_conversation = uuid.uuid4().hex
        self.logger = kwargs.get('logger', logging.getLogger(__name__))

        # ES: Por defecto el bot guarda su conocimiento en una base de datos SQL,
        # ES: pero este adaptador se puede reemplazar por MongoDB, Redis, etc.
        storage_adapter = kwargs.get('storage_adapter', 'chatterbot.storage.SQLStorageAdapter')

        # ES: BestMatch es el 'cerebro' por defecto: compara la entrada del
        # ES: usuario contra lo aprendido y elige la mejor respuesta.
        logic_adapters = kwargs.get('logic_adapters', ['chatterbot.logic.BestMatch'])

        self.storage = utils.initialize_class(storage_adapter, **kwargs)

        # ES: El 'tagger' usa spaCy para analizar gramaticalmente el texto
        # ES: (lematización y etiquetas POS) y así comparar frases con más precisión.
        self.tagger = Tagger(language=tagger_language)

        # ES: Se pueden usar tres algoritmos de búsqueda distintos:
        # ES: texto indexado, texto simple o vectores semánticos (IA).
        primary_search_algorithm = IndexedTextSearch(self, **kwargs)
        text_search_algorithm = TextSearch(self, **kwargs)
        semantic_vector_search_algorithm = SemanticVectorSearch(self, **kwargs)

        # ES: Se inicializa cada adaptador de lógica configurado (por defecto, BestMatch).
        for adapter in logic_adapters:
            logic_adapter = utils.initialize_class(adapter, self, **kwargs)
            self.logic_adapters.append(logic_adapter)

        # ES: Si read_only=True, el bot responde pero NO aprende de la conversación.
        self.read_only = kwargs.get('read_only', False)


    # Método generate_response — aquí ocurre la decisión final de qué responder:

    # ES: Este método recorre TODOS los adaptadores de lógica activos y le
    # ES: pregunta a cada uno qué respuesta propone y con qué confianza.
    def generate_response(self, input_statement, additional_response_selection_parameters=None):
        results = []
        result = None
        max_confidence = -1

        for adapter in self.logic_adapters:
            if adapter.can_process(input_statement):
                output = adapter.process(input_statement, additional_response_selection_parameters)
                results.append(output)

                # ES: Se queda con la respuesta de MAYOR confianza encontrada hasta ahora.
                if output.confidence > max_confidence:
                    result = output
                    max_confidence = output.confidence

        # ES: Si hay 3 o más adaptadores activos y varios proponen la MISMA
        # ES: respuesta, esa respuesta gana por 'votación', aunque su confianza
        # ES: individual no fuera la más alta. Es un mecanismo de consenso.
        if len(results) >= 3:
            result_options = {}
            for result_option in results:
                result_string = result_option.text + ':' + (result_option.in_response_to or '')

                if result_string in result_options:
                    result_options[result_string].count += 1
                else:
                    result_options[result_string] = ResultOption(result_option)

            most_common = list(result_options.values())[0]
            for result_option in result_options.values():
                if result_option.count > most_common.count:
                    most_common = result_option

        # ES: Se construye el objeto Statement final que el bot enviará como
        # ES: respuesta, incluyendo su nivel de confianza.
        response = Statement(
            text=result.text,
            in_response_to=input_statement.text,
            conversation=input_statement.conversation,
            persona='bot:' + self.name
        )
        response.confidence = result.confidence
        return response
```

### 3.2 `logic/best_match.py` — Adaptador BestMatch

Este es el adaptador de lógica por defecto. Busca, entre lo que el bot ya aprendió, la frase más parecida a la del usuario (`closest_match`) y luego elige la respuesta asociada a esa frase.

**📄 Archivo: `best_match.py`** *(chatterbot/logic/best_match.py)*

```python
class BestMatch(LogicAdapter):
    def process(self, input_statement, additional_response_selection_parameters=None):

        # ES: El algoritmo de búsqueda (search_algorithm) compara el texto de
        # ES: entrada con todas las frases ya conocidas por el bot.
        search_results = self.search_algorithm.search(input_statement)

        # ES: Si no encuentra ninguna coincidencia, usa la propia entrada del
        # ES: usuario como referencia, con confianza 0 (caso 'no sé qué responder').
        input_statement.confidence = 0
        closest_match = input_statement

        # ES: Recorre los resultados ordenados por similitud y se detiene apenas
        # ES: encuentra una coincidencia 'suficientemente buena' (umbral configurable).
        for result in search_results:
            closest_match = result
            if result.confidence >= self.maximum_similarity_threshold:
                break

        # ES: Con la mejor coincidencia identificada, se buscan en la base de
        # ES: datos todas las respuestas que en el pasado siguieron a esa frase.
        response_selection_parameters = {
            'search_text': closest_match.search_text,
            'persona_not_startswith': 'bot:',
            'exclude_text': recent_repeated_responses,
        }
        response_list = list(self.chatbot.storage.filter(**response_selection_parameters))

        # ES: Si existen varias respuestas posibles, select_response elige una
        # ES: entre ellas (por ejemplo, la más frecuente históricamente).
        if response_list:
            response = self.select_response(input_statement, response_list, self.chatbot.storage)
            response.confidence = closest_match.confidence
        else:
            # ES: Si no hay ninguna respuesta registrada, se entrega una respuesta
            # ES: por defecto (fallback) para que el bot nunca se quede sin contestar.
            response = self.get_default_response(input_statement)

        return response
```

### 3.3 `trainers.py` — Entrenamiento del bot (ListTrainer)

Antes de poder responder algo útil, el bot debe ser entrenado. `ListTrainer` recibe una lista de frases que representan una conversación de ejemplo y las guarda como pares pregunta→respuesta.

**📄 Archivo: `trainers.py`** *(chatterbot/trainers.py)*

```python
# ES: Clase base: define la estructura común que deben seguir todos los
# ES: 'entrenadores' (ListTrainer, ChatterBotCorpusTrainer, etc.).
class Trainer(object):
    def __init__(self, chatbot, **kwargs):
        self.chatbot = chatbot

    def train(self, *args, **kwargs):
        # ES: Este método es 'abstracto': obliga a cada subclase a implementar
        # ES: su propia forma de entrenar (lista, archivo, base de datos, etc.).
        raise self.TrainerInitializationException()


class ListTrainer(Trainer):
    def train(self, conversation):
        # ES: Cada elemento de la lista 'conversation' es una frase; se asume que
        # ES: la frase N es una respuesta a la frase N-1 (turnos de diálogo).
        previous_statement_text = None
        statements_to_create = []

        # ES: Antes del análisis de lenguaje natural, se limpia el texto
        # ES: (por ejemplo, espacios en blanco sobrantes).
        for preprocessor in self.chatbot.preprocessors:
            preprocessed_texts = [preprocessor(Statement(text=t)).text for t in preprocessed_texts]

        # ES: Todo el lote de frases se procesa de una vez con spaCy (más
        # ES: eficiente que analizarlas una por una).
        documents = list(self.chatbot.tagger.as_nlp_pipeline(preprocessed_texts, batch_size=2000))

        # ES: Cada frase se convierte en un objeto Statement y se enlaza con la
        # ES: frase anterior mediante 'in_response_to', formando así los pares
        # ES: pregunta→respuesta que luego usará BestMatch para contestar.
        for document in documents:
            statement_text = document.text
            # ... se crea el Statement y se guarda en el storage adapter
```

## 4. Resumen de la lógica operativa (para el diagrama de flujo)

1. El usuario envía un mensaje de texto al bot (`get_response`).
2. El texto se preprocesa (limpieza) y se etiqueta gramaticalmente con spaCy.
3. El motor de búsqueda compara el texto contra todas las frases ya aprendidas.
4. BestMatch identifica la frase más parecida (`closest_match`) y calcula un nivel de confianza.
5. Se buscan las respuestas históricas asociadas a esa frase; si no existen, se usa una respuesta por defecto.
6. Si hay varios adaptadores de lógica activos, se compara lo que proponen y gana la respuesta con más consenso o mayor confianza.
7. La respuesta se envía al usuario y, si el bot no está en modo `read_only`, la conversación completa se guarda para seguir aprendiendo.

## 5. Diagrama de flujo (lógica operativa del código)

Diagrama elaborado con Graphviz (software de diagramas de flujo) a partir del análisis del código fuente documentado en la sección 3. Representa el recorrido completo de un mensaje dentro de ChatterBot: desde que el usuario escribe, hasta que el bot responde y aprende.

**Parte 1 de 2**

![Diagrama de flujo - Parte 1](images/diagrama-flujo-parte1.png)

**Parte 2 de 2 (continuación)**

![Diagrama de flujo - Parte 2](images/diagrama-flujo-parte2.png)

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
