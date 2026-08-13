**📄 Archivo: `chatterbot.py`** *(chatterbot/chatterbot.py)*

```python
import logging
import uuid
from typing import Union
from chatterbot.storage import StorageAdapter
from chatterbot.logic import LogicAdapter
from chatterbot.search import TextSearch, IndexedTextSearch, SemanticVectorSearch
from chatterbot.tagging import PosLemmaTagger
from chatterbot.conversation import Statement
from chatterbot import languages
from chatterbot import utils
import spacy


class ChatBot(object):
    """
    A conversational dialog chat bot.

    :param name: A name is the only required parameter for the ChatBot class.
    :type name: str

    :keyword storage_adapter: The dot-notated import path to a storage adapter class.
                              Defaults to ``"chatterbot.storage.SQLStorageAdapter"``.
    :type storage_adapter: str

    :param logic_adapters: A list of dot-notated import paths to each logic adapter the bot uses.
                           Defaults to ``["chatterbot.logic.BestMatch"]``.
    :type logic_adapters: list

    :param tagger: The tagger to use for the chat bot.
                   Defaults to :class:`~chatterbot.tagging.PosLemmaTagger`
    :type tagger: object

    :param tagger_language: The language to use for the tagger.
                            Defaults to :class:`~chatterbot.languages.ENG`.
    :type tagger_language: object

    :param preprocessors: A list of preprocessor functions to use for the chat bot.
    :type preprocessors: list

    :param read_only: If True, the chat bot will not save any input it receives, defaults to False.
    :type read_only: bool

    :param logger: A ``Logger`` object.
    :type logger: logging.Logger

    :param model: A definition used to load a large language model.
                  Defaults to ``None``.
                  (Added in version 1.2.7)
    :type model: dict

    :param stream: Return output as a streaming responses when a ``model`` is defined.
                   (Added in version 1.2.7)
    """

    def __init__(self, name, stream=False, **kwargs):
        self.name = name

        self.stream = stream

        # Generate a default conversation ID for this ChatBot instance.
        # This is used as a fallback when callers don't provide an explicit
        # conversation ID, ensuring that conversation history is tracked
        # within a session. Conversation IDs are necessary for cases such as
        # the LLM-based logic adapters which require it to retrieve previous
        # messages.
        self.default_conversation = uuid.uuid4().hex

        self.logger = kwargs.get('logger', logging.getLogger(__name__))

        storage_adapter = kwargs.get('storage_adapter', 'chatterbot.storage.SQLStorageAdapter')

        logic_adapters = kwargs.get('logic_adapters', [
            'chatterbot.logic.BestMatch'
        ])

        # Check that each adapter is a valid subclass of it's respective parent
        utils.validate_adapter_class(storage_adapter, StorageAdapter)

        # Logic adapters used by the chat bot
        self.logic_adapters = []

        self.storage = utils.initialize_class(storage_adapter, **kwargs)

        tagger_language = kwargs.get('tagger_language', languages.ENG)

        # Check if storage adapter has a preferred tagger
        PreferredTagger = self.storage.get_preferred_tagger()

        if PreferredTagger is not None:
            # Storage adapter specifies its own tagger
            self.tagger = PreferredTagger(language=tagger_language)
        else:
            # Use default or user-specified tagger
            try:
                Tagger = kwargs.get('tagger', PosLemmaTagger)

                # Allow instances to be provided for performance optimization
                # (Example: a pre-loaded model in a tagger when unit testing)
                if not isinstance(Tagger, type):
                    self.tagger = Tagger
                else:
                    self.tagger = Tagger(language=tagger_language)
            except IOError as io_error:
                # Return a more helpful error message if possible
                if "Can't find model" in str(io_error):
                    model_name = utils.get_model_for_language(tagger_language)
                    if hasattr(tagger_language, 'ENGLISH_NAME'):
                        language_name = tagger_language.ENGLISH_NAME
                    else:
                        language_name = tagger_language
                    raise self.ChatBotException(
                        'Setup error:\n'
                        f'The Spacy model for "{language_name}" language is missing.\n'
                        'Please install the model using the command:\n\n'
                        f'python -m spacy download {model_name}\n\n'
                        'See https://spacy.io/usage/models for more information about available models.'
                    ) from io_error
                else:
                    raise io_error

        # Initialize search algorithms
        primary_search_algorithm = IndexedTextSearch(self, **kwargs)
        text_search_algorithm = TextSearch(self, **kwargs)
        semantic_vector_search_algorithm = SemanticVectorSearch(self, **kwargs)

        self.search_algorithms = {
            primary_search_algorithm.name: primary_search_algorithm,
            text_search_algorithm.name: text_search_algorithm,
            semantic_vector_search_algorithm.name: semantic_vector_search_algorithm
        }

        # Check if storage adapter has a preferred search algorithm
        preferred_search_algorithm = self.storage.get_preferred_search_algorithm()
        if preferred_search_algorithm and preferred_search_algorithm in self.search_algorithms:
            # Set as default for logic adapters that don't specify their own search algorithm
            # This ensures BestMatch and other adapters use the optimal search method
            self.logger.info(f'Storage adapter prefers search algorithm: {preferred_search_algorithm}')
            kwargs.setdefault('search_algorithm_name', preferred_search_algorithm)

        for adapter in logic_adapters:
            utils.validate_adapter_class(adapter, LogicAdapter)
            logic_adapter = utils.initialize_class(adapter, self, **kwargs)
            self.logic_adapters.append(logic_adapter)

        preprocessors = kwargs.get(
            'preprocessors', [
                'chatterbot.preprocessors.clean_whitespace'
            ]
        )

        self.preprocessors = []

        for preprocessor in preprocessors:
            self.preprocessors.append(utils.import_module(preprocessor))

        # NOTE: 'xx' is the language code for a multi-language model
        self.nlp = spacy.blank(self.tagger.language.ISO_639_1)

        # Allow the bot to save input it receives so that it can learn
        self.read_only = kwargs.get('read_only', False)

    def get_response(self, statement: Union[Statement, str, dict] = None, **kwargs) -> Statement:
        """
        Return the bot's response based on the input.

        :param statement: An statement object or string.
        :returns: A response to the input.

        :param additional_response_selection_parameters: Parameters to pass to the
            chat bot's logic adapters to control response selection.
        :type additional_response_selection_parameters: dict

        :param persist_values_to_response: Values that should be saved to the response
            that the chat bot generates.
        :type persist_values_to_response: dict
        """
        Statement = self.storage.get_object('statement')

        additional_response_selection_parameters = kwargs.pop('additional_response_selection_parameters', {})

        persist_values_to_response = kwargs.pop('persist_values_to_response', {})

        if isinstance(statement, str):
            kwargs['text'] = statement

        if isinstance(statement, dict):
            kwargs.update(statement)

        if statement is None and 'text' not in kwargs:
            raise self.ChatBotException(
                'Either a statement object or a "text" keyword '
                'argument is required. Neither was provided.'
            )

        if hasattr(statement, 'serialize'):
            kwargs.update(**statement.serialize())

        tags = kwargs.pop('tags', [])

        text = kwargs.pop('text')

        input_statement = Statement(text=text, **kwargs)

        input_statement.add_tags(*tags)

        # If no conversation ID was provided, use the default session ID
        # so that conversation history is tracked across calls. Callers
        # can override this by passing an explicit conversation kwarg or
        # setting it on the Statement object.
        if not input_statement.conversation:
            input_statement.conversation = self.default_conversation

        # Preprocess the input statement
        for preprocessor in self.preprocessors:
            input_statement = preprocessor(input_statement)

        # Mark the statement as being a response to the previous
        if input_statement.in_response_to is None:
            previous_statement = self.get_latest_response(input_statement.conversation)
            if previous_statement:
                input_statement.in_response_to = previous_statement.text

        # Make sure the input statement has its search text saved
        if not self.tagger.needs_text_indexing():
            # Tagger doesn't transform text, use it directly
            if not input_statement.search_text:
                input_statement.search_text = input_statement.text
            if not input_statement.search_in_response_to and input_statement.in_response_to:
                input_statement.search_in_response_to = input_statement.in_response_to
        else:
            # Use tagger for text indexing or transformations
            if not input_statement.search_text:
                _search_text = self.tagger.get_text_index_string(input_statement.text)
                input_statement.search_text = _search_text

            if not input_statement.search_in_response_to and input_statement.in_response_to:
                input_statement.search_in_response_to = self.tagger.get_text_index_string(
                    input_statement.in_response_to
                )

        response = self.generate_response(
            input_statement,
            additional_response_selection_parameters
        )

        # If streaming is enabled return the response immediately
        if self.stream:
            return response

        # Update any response data that needs to be changed
        if persist_values_to_response:
            for response_key in persist_values_to_response:
                response_value = persist_values_to_response[response_key]
                if response_key == 'tags':
                    input_statement.add_tags(*response_value)
                    response.add_tags(*response_value)
                else:
                    setattr(input_statement, response_key, response_value)
                    setattr(response, response_key, response_value)

        if not self.read_only:

            # Save the input statement
            self.storage.create(**input_statement.serialize())

            # Save the response generated for the input
            self.learn_response(response, previous_statement=input_statement)

        return response

    def generate_response(self, input_statement, additional_response_selection_parameters=None):
        """
        Return a response based on a given input statement.

        :param input_statement: The input statement to be processed.
        """
        Statement = self.storage.get_object('statement')

        results = []
        result = None
        max_confidence = -1

        for adapter in self.logic_adapters:
            if adapter.can_process(input_statement):

                output = adapter.process(input_statement, additional_response_selection_parameters)
                results.append(output)

                self.logger.info(
                    '{} selected "{}" as a response with a confidence of {}'.format(
                        adapter.class_name, output.text, output.confidence
                    )
                )

                if output.confidence > max_confidence:
                    result = output
                    max_confidence = output.confidence
            else:
                self.logger.info(
                    'Not processing the statement using {}'.format(adapter.class_name)
                )

        class ResultOption:
            def __init__(self, statement, count=1):
                self.statement = statement
                self.count = count

        # If multiple adapters agree on the same statement,
        # then that statement is more likely to be the correct response
        if len(results) >= 3:
            result_options = {}
            for result_option in results:
                result_string = result_option.text + ':' + (result_option.in_response_to or '')

                if result_string in result_options:
                    result_options[result_string].count += 1
                    if result_options[result_string].statement.confidence < result_option.confidence:
                        result_options[result_string].statement = result_option
                else:
                    result_options[result_string] = ResultOption(
                        result_option
                    )

            most_common = list(result_options.values())[0]

            for result_option in result_options.values():
                if result_option.count > most_common.count:
                    most_common = result_option

            self.logger.info('Selecting "{}" as the most common response'.format(most_common.statement.text))

            if most_common.count > 1:
                result = most_common.statement

        response = Statement(
            text=result.text,
            in_response_to=input_statement.text,
            conversation=input_statement.conversation,
            persona='bot:' + self.name
        )

        response.add_tags(*result.get_tags())

        response.confidence = result.confidence

        return response

    def learn_response(self, statement, previous_statement=None):
        """
        Learn that the statement provided is a valid response.
        """
        if not previous_statement:
            previous_statement = statement.in_response_to

        if not previous_statement:
            previous_statement = self.get_latest_response(statement.conversation)
            if previous_statement:
                previous_statement = previous_statement.text

        previous_statement_text = previous_statement

        if not isinstance(previous_statement, (str, type(None), )):
            statement.in_response_to = previous_statement.text
        elif isinstance(previous_statement, str):
            statement.in_response_to = previous_statement

        self.logger.info('Adding "{}" as a response to "{}"'.format(
            statement.text,
            previous_statement_text
        ))

        if not statement.persona:
            statement.persona = 'bot:' + self.name

        # Save the response statement
        return self.storage.create(**statement.serialize())

    def get_latest_response(self, conversation: str):
        """
        Returns the latest response in a conversation if it exists.
        Returns None if a matching conversation cannot be found.
        """
        conversation_statements = list(self.storage.filter(
            conversation=conversation,
            order_by=['id']
        ))

        # Get the most recent statement in the conversation if one exists
        latest_statement = conversation_statements[-1] if len(conversation_statements) else None

        return latest_statement

    class ChatBotException(Exception):
        pass
```

### 3.2 `logic/best_match.py` — Adaptador BestMatch

Este es el adaptador de lógica por defecto. Busca, entre lo que el bot ya aprendió, la frase más parecida a la del usuario (`closest_match`) y luego elige la respuesta asociada a esa frase.

**📄 Archivo: `best_match.py`** *(chatterbot/logic/best_match.py)*

```python
from chatterbot.logic import LogicAdapter
from chatterbot.conversation import Statement
from chatterbot import filters


class BestMatch(LogicAdapter):
    """
    A logic adapter that returns a response based on known responses to
    the closest matches to the input statement.

    :param excluded_words:
        The excluded_words parameter allows a list of words to be set that will
        prevent the logic adapter from returning statements that have text
        containing any of those words. This can be useful for preventing your
        chat bot from saying swears when it is being demonstrated in front of
        an audience.
        Defaults to None
    :type excluded_words: list
    """

    def __init__(self, chatbot, **kwargs):
        super().__init__(chatbot, **kwargs)

        self.excluded_words = kwargs.get('excluded_words')

    def process(self, input_statement: Statement, additional_response_selection_parameters=None) -> Statement:

        # Get all statements that have a response text similar to the input statement
        search_results = self.search_algorithm.search(input_statement)

        # Use the input statement as the closest match if no other results are found
        input_statement.confidence = 0  # Use 0 confidence when no other results are found
        closest_match = input_statement

        # Search for the closest match to the input statement
        for result in search_results:
            closest_match = result

            # Stop searching if a match that is close enough is found
            if result.confidence >= self.maximum_similarity_threshold:
                break

        self.chatbot.logger.info('Selecting "{}" as a response to "{}" with a confidence of {}'.format(
            closest_match.text, input_statement.text, closest_match.confidence
        ))

        # Semantic vector search vs indexed text search have different architectures:
        #
        # For SQL with indexed text search:
        #   - Phase 1 finds a match based on string similarity (Levenshtein distance)
        #   - Phase 2 finds variations of that match to get diverse responses
        #   - This makes sense because you might have multiple instances of similar statements
        #     learned from different conversations that provide different response options
        #
        # For Redis with semantic vectors:
        #   - Phase 1 finds semantically similar responses using vector embeddings
        #   - The semantic similarity already captures the "closeness" we want
        #   - Phase 2 would be redundant - we already have the best semantic match
        #   - The vector search inherently considers the entire semantic space, not just
        #     exact string matches, so additional variation searching is unnecessary
        #
        # NOTE: This difference of functionality may need to be modified in the future
        # if the redis adapter is determined to benefit from a Phase 2 style response
        # selection. The main symptom that would drive such a change would be low
        # quality or repetitive responses when using semantic vector search.
        #
        # Therefore, semantic vector search returns the Phase 1 result directly.
        if self.search_algorithm.name == 'semantic_vector_search' and closest_match.confidence > 0:
            response = closest_match
            self.chatbot.logger.info('Using semantic search result directly: "{}"'.format(response.text))
        else:
            # For other search algorithms (indexed_text_search, text_search),
            # we need to find responses to the closest match
            recent_repeated_responses = filters.get_recent_repeated_responses(
                self.chatbot,
                input_statement.conversation
            )

            for index, recent_repeated_response in enumerate(recent_repeated_responses):
                self.chatbot.logger.info('{}. Excluding recent repeated response of "{}"'.format(
                    index, recent_repeated_response
                ))

            response_selection_parameters = {
                'search_text': closest_match.search_text,
                'persona_not_startswith': 'bot:',
                'exclude_text': recent_repeated_responses,
                'exclude_text_words': self.excluded_words
            }

            alternate_response_selection_parameters = {
                'search_in_response_to': input_statement.search_text or self.chatbot.tagger.get_text_index_string(
                    input_statement.text
                ),
                'persona_not_startswith': 'bot:',
                'exclude_text': recent_repeated_responses,
                'exclude_text_words': self.excluded_words
            }

            if additional_response_selection_parameters:
                response_selection_parameters.update(
                    additional_response_selection_parameters
                )
                alternate_response_selection_parameters.update(
                    additional_response_selection_parameters
                )

            # Get all statements with text similar to the closest match
            response_list = list(self.chatbot.storage.filter(**response_selection_parameters))

            if response_list:
                response = self.select_response(
                    input_statement,
                    response_list,
                    self.chatbot.storage
                )

                response.confidence = closest_match.confidence
                self.chatbot.logger.info('Selecting "{}" from {} optimal responses.'.format(
                    response.text,
                    len(response_list)
                ))
            else:
                '''
                The case where there was no responses returned for the selected match
                but a value exists for the statement the match is in response to.
                '''
                self.chatbot.logger.info('No responses found. Generating alternate response list.')

                alternate_response_list = list(self.chatbot.storage.filter(
                    **alternate_response_selection_parameters
                ))

                if alternate_response_list:
                    response = self.select_response(
                        input_statement,
                        alternate_response_list,
                        self.chatbot.storage
                    )

                    response.confidence = closest_match.confidence
                    self.chatbot.logger.info('Selected alternative response "{}" from {} options'.format(
                        response.text,
                        len(alternate_response_list)
                    ))
                else:
                    response = self.get_default_response(input_statement)
                    self.chatbot.logger.info('Using "%s" as a default response.', response.text)

        return response
```

### 3.3 `trainers.py` — Entrenamiento del bot (ListTrainer)

Antes de poder responder algo útil, el bot debe ser entrenado. `ListTrainer` recibe una lista de frases que representan una conversación de ejemplo y las guarda como pares pregunta→respuesta.

**📄 Archivo: `trainers.py`** *(chatterbot/trainers.py)*

```python
import os
import csv
import time
import glob
import json
import tarfile
from typing import List, Union
from tqdm import tqdm
from dateutil import parser as date_parser
from chatterbot.chatterbot import ChatBot
from chatterbot.conversation import Statement


class Trainer(object):
    """
    Base class for all other trainer classes.

    :param boolean show_training_progress: Show progress indicators for the
           trainer. The environment variable ``CHATTERBOT_SHOW_TRAINING_PROGRESS``
           can also be set to control this. ``show_training_progress`` will override
           the environment variable if it is set.
    """

    def __init__(self, chatbot: ChatBot, **kwargs):
        self.chatbot = chatbot

        environment_default = bool(int(os.environ.get('CHATTERBOT_SHOW_TRAINING_PROGRESS', True)))

        self.disable_progress = not kwargs.get(
            'show_training_progress',
            environment_default
        )

    def get_preprocessed_statement(self, input_statement: Statement) -> Statement:
        """
        Preprocess the input statement.
        """
        for preprocessor in self.chatbot.preprocessors:
            input_statement = preprocessor(input_statement)

        return input_statement

    def train(self, *args, **kwargs):
        """
        This method must be overridden by a child class.
        """
        raise self.TrainerInitializationException()

    class TrainerInitializationException(Exception):
        """
        Exception raised when a base class has not overridden
        the required methods on the Trainer base class.
        """

        def __init__(self, message=None):
            default = (
                'A training class must be specified before calling train(). '
                'See https://docs.chatterbot.us/training/'
            )
            super().__init__(message or default)

    def _generate_export_data(self) -> list:
        result = []
        for statement in self.chatbot.storage.filter():
            if statement.in_response_to:
                result.append([statement.in_response_to, statement.text])

        return result

    def export_for_training(self, file_path='./export.json'):
        """
        Create a file from the database that can be used to
        train other chat bots.
        """
        export = {'conversations': self._generate_export_data()}
        with open(file_path, 'w+', encoding='utf8') as jsonfile:
            json.dump(export, jsonfile, ensure_ascii=False)


class ListTrainer(Trainer):
    """
    Allows a chat bot to be trained using a list of strings
    where the list represents a conversation.
    """

    def train(self, conversation: List[str]):
        """
        Train the chat bot based on the provided list of
        statements that represents a single conversation.
        """
        previous_statement_text = None
        previous_statement_search_text = ''
        statements_to_create = []

        # Preprocess all text before NLP analysis
        preprocessed_texts = conversation
        for preprocessor in self.chatbot.preprocessors:
            preprocessed_texts = [
                preprocessor(Statement(text=text)).text
                for text in preprocessed_texts
            ]

        # Batch process with NLP
        documents = list(self.chatbot.tagger.as_nlp_pipeline(
            preprocessed_texts,
            batch_size=2000,
            # NOTE: Not all spaCy models support multi-processing
            n_process=1
        ))

        # Create statements from processed documents
        for document in tqdm(documents, desc='List Trainer', disable=self.disable_progress):
            # Handle both spaCy Doc objects and plain strings from NoOpTagger
            if isinstance(document, str):
                # NoOpTagger returns plain strings
                statement_text = document
                statement_search_text = self.chatbot.tagger.get_text_index_string(document)
            else:
                # Regular taggers return spaCy Doc objects
                statement_text = document.text
                statement_search_text = document._.search_index

            statement = Statement(
                text=statement_text,
                search_text=statement_search_text,
                in_response_to=previous_statement_text,
                search_in_response_to=previous_statement_search_text,
                conversation='training'
            )

            previous_statement_text = statement.text
            previous_statement_search_text = statement_search_text
            statements_to_create.append(statement)

        self.chatbot.storage.create_many(statements_to_create)


class ChatterBotCorpusTrainer(Trainer):
    """
    Allows the chat bot to be trained using data from the
    ChatterBot dialog corpus.
    """

    def train(self, *corpus_paths: Union[str, List[str]]):
        from chatterbot.corpus import load_corpus, list_corpus_files

        data_file_paths = []

        # Get the paths to each file the bot will be trained with
        for corpus_path in corpus_paths:
            data_file_paths.extend(list_corpus_files(corpus_path))

        for corpus, categories, _file_path in tqdm(
            load_corpus(*data_file_paths),
            desc='Training corpus',
            disable=self.disable_progress
        ):
            statements_to_create = []

            # Collect all texts from all conversations for batch processing
            all_texts = []
            conversation_lengths = []

            for conversation in corpus:
                conversation_lengths.append(len(conversation))
                all_texts.extend(conversation)

            # Preprocess all texts
            preprocessed_texts = all_texts
            for preprocessor in self.chatbot.preprocessors:
                preprocessed_texts = [
                    preprocessor(Statement(text=text)).text
                    for text in preprocessed_texts
                ]

            # Batch process all texts with NLP
            documents = list(self.chatbot.tagger.as_nlp_pipeline(
                preprocessed_texts,
                batch_size=2000,
                # NOTE: Not all spaCy models support multi-processing
                n_process=1
            ))

            # Reconstruct conversations from batch-processed documents
            doc_index = 0
            for conversation_length in conversation_lengths:
                previous_statement_text = None
                previous_statement_search_text = ''

                for _ in range(conversation_length):
                    document = documents[doc_index]
                    doc_index += 1

                    # Handle both spaCy Doc objects and plain strings from NoOpTagger
                    if isinstance(document, str):
                        # NoOpTagger returns plain strings
                        statement_text = document
                        statement_search_text = self.chatbot.tagger.get_text_index_string(document)
                    else:
                        # Regular taggers return spaCy Doc objects
                        statement_text = document.text
                        statement_search_text = document._.search_index

                    statement = Statement(
                        text=statement_text,
                        search_text=statement_search_text,
                        in_response_to=previous_statement_text,
                        search_in_response_to=previous_statement_search_text,
                        conversation='training'
                    )

                    statement.add_tags(*categories)

                    previous_statement_text = statement.text
                    previous_statement_search_text = statement_search_text
                    statements_to_create.append(statement)

            if statements_to_create:
                self.chatbot.storage.create_many(statements_to_create)


class GenericFileTrainer(Trainer):
    """
    Allows the chat bot to be trained using data from a CSV or JSON file,
    or directory of those file types.
    """

    # NOTE: If the value is an integer, this be the
    # column index instead of the key or header
    DEFAULT_STATEMENT_TO_HEADER_MAPPING = {
        'text': 'text',
        'conversation': 'conversation',
        'created_at': 'created_at',
        'persona': 'persona',
        'tags': 'tags'
    }

    def __init__(self, chatbot: ChatBot, **kwargs):
        """
        data_path: str The path to the data file or directory.
        field_map: dict A dictionary containing the column name to header mapping.
        """
        super().__init__(chatbot, **kwargs)

        self.file_extension = None

        self.field_map = kwargs.get(
            'field_map',
            self.DEFAULT_STATEMENT_TO_HEADER_MAPPING
        )

    def _get_file_list(self, data_path: str, limit: Union[int, None]):
        """
        Get a list of files to read from the data set.
        """

        if self.file_extension is None:
            raise self.TrainerInitializationException(
                'The file_extension attribute must be set before calling train().'
            )

        # List all csv or json files in the specified directory
        if os.path.isdir(data_path):
            glob_path = os.path.join(data_path, '**', f'*.{self.file_extension}')

            # Use iglob instead of glob for better performance with
            # large directories because it returns an iterator
            data_files = glob.iglob(glob_path, recursive=True)

            for index, file_path in enumerate(data_files):
                if limit is not None and index >= limit:
                    break

                yield file_path
        else:
            yield data_path

    def train(self, data_path: str, limit=None):
        """
        Train a chatbot with data from the data file.

        :param str data_path: The path to the data file or directory.
        :param int limit: The maximum number of files to train from.
        """

        if data_path is None:
            raise self.TrainerInitializationException(
                'The data_path argument must be set to the path of a file or directory.'
            )

        data_files = self._get_file_list(data_path, limit)

        files_processed = 0

        for data_file in tqdm(data_files, desc='Training', disable=self.disable_progress):

            previous_statement_text = None
            previous_statement_search_text = ''

            file_extension = data_file.split('.')[-1].lower()

            statements_to_create = []

            file_abspath = os.path.abspath(data_file)

            with open(file_abspath, 'r', encoding='utf-8') as file:

                if self.file_extension == 'json':
                    data = json.load(file)
                    data = data['conversation']
                elif file_extension == 'csv':
                    use_header = bool(isinstance(next(iter(self.field_map.values())), str))

                    if use_header:
                        data = csv.DictReader(file)
                    else:
                        data = csv.reader(file)
                elif file_extension == 'tsv':
                    use_header = bool(isinstance(next(iter(self.field_map.values())), str))

                    if use_header:
                        data = csv.DictReader(file, delimiter='\t')
                    else:
                        data = csv.reader(file, delimiter='\t')
                else:
                    self.logger.warning(f'Skipping unsupported file type: {file_extension}')
                    continue

                files_processed += 1

                text_row = self.field_map['text']

                # Collect all rows first to avoid re-reading file
                rows_list = [row for row in data if len(row) > 0]

                # Extract text and metadata for each row
                text_values = []
                contexts = []

                try:
                    for row in rows_list:
                        context = {
                            key: row[value]
                            for key, value in self.field_map.items()
                            if key != text_row
                        }
                        contexts.append(context)

                        # Preprocess text
                        text = row[text_row]
                        for preprocessor in self.chatbot.preprocessors:
                            text = preprocessor(Statement(text=text)).text

                        text_values.append((text, context))
                except KeyError as e:
                    raise KeyError(
                        f'{e}. Please check the field_map parameter used to initialize '
                        f'the training class and remove this value if it is not needed. '
                        f'Current mapping: {self.field_map}'
                    )

                # Batch process with NLP
                documents = self.chatbot.tagger.as_nlp_pipeline(
                    text_values,
                    batch_size=2000,
                    # NOTE: Not all spaCy models support multi-processing
                    n_process=1
                )

                # Convert to list for processing
                documents_list = list(documents)

            response_to_search_index_mapping = {}

            if 'in_response_to' in self.field_map.keys():
                # Process response references for search indexing
                in_response_to_field = self.field_map['in_response_to']
                response_texts = [
                    row[in_response_to_field]
                    for row in rows_list
                    if row[in_response_to_field] is not None
                ]

                if response_texts:
                    # Preprocess response texts
                    preprocessed_response_texts = response_texts
                    for preprocessor in self.chatbot.preprocessors:
                        preprocessed_response_texts = [
                            preprocessor(Statement(text=text)).text
                            for text in preprocessed_response_texts
                        ]

                    # Batch process response texts
                    response_documents = self.chatbot.tagger.as_nlp_pipeline(
                        preprocessed_response_texts,
                        batch_size=2000,
                        # NOTE: Not all spaCy models support multi-processing
                        n_process=1
                    )

                    for document in response_documents:
                        # Handle both spaCy Doc objects and plain strings from NoOpTagger
                        if isinstance(document, str):
                            # NoOpTagger returns plain strings
                            response_to_search_index_mapping[document] = self.chatbot.tagger.get_text_index_string(document)
                        else:
                            # Regular taggers return spaCy Doc objects
                            response_to_search_index_mapping[document.text] = document._.search_index

            # Create statements from processed documents
            for document, context in tqdm(documents_list, desc='Creating statements', disable=self.disable_progress, leave=False):
                # Handle both spaCy Doc objects and plain strings from NoOpTagger
                if isinstance(document, str):
                    # NoOpTagger returns plain strings
                    statement_text = document
                    statement_search_text = self.chatbot.tagger.get_text_index_string(document)
                else:
                    # Regular taggers return spaCy Doc objects
                    statement_text = document.text
                    statement_search_text = document._.search_index
                
                statement = Statement(
                    text=statement_text,
                    conversation=context.get('conversation', 'training'),
                    persona=context.get('persona', None),
                    tags=context.get('tags', [])
                )

                if 'created_at' in context:
                    statement.created_at = date_parser.parse(context['created_at'])

                statement.search_text = statement_search_text

                # Use the in_response_to attribute for the previous statement if
                # one is defined, otherwise use the last statement which was created
                if 'in_response_to' in self.field_map.keys():
                    statement.in_response_to = context.get(self.field_map['in_response_to'], None)
                    statement.search_in_response_to = response_to_search_index_mapping.get(
                        context.get(self.field_map['in_response_to'], None), ''
                    )
                else:
                    # List-type data such as CSVs with no response specified can use
                    # the previous statement as the in_response_to value
                    statement.in_response_to = previous_statement_text
                    statement.search_in_response_to = previous_statement_search_text

                previous_statement_text = statement.text
                previous_statement_search_text = statement.search_text

                statements_to_create.append(statement)

            self.chatbot.storage.create_many(statements_to_create)

        if files_processed:
            self.chatbot.logger.info(
                'Training completed. {} files were read.'.format(files_processed)
            )
        else:
            self.chatbot.logger.warning(
                'No [{}] files were detected at: {}'.format(
                    self.file_extension,
                    data_path
                )
            )


class CsvFileTrainer(GenericFileTrainer):
    """
    .. note::
        Added in version 1.2.4

    Allow chatbots to be trained with data from a CSV file or
    directory of CSV files.

    TSV files are also supported, as long as the file_extension
    parameter is set to 'tsv'.

    :param str file_extension: The file extension to look for when searching for files (defaults to 'csv').
    :param dict field_map: A dictionary containing the database column name to header mapping.
                          Values can be either the header name (str) or the column index (int).
    """

    def __init__(self, chatbot: ChatBot, **kwargs):
        super().__init__(chatbot, **kwargs)

        self.file_extension = kwargs.get('file_extension', 'csv')


class JsonFileTrainer(GenericFileTrainer):
    """
    .. note::
        Added in version 1.2.4

    Allow chatbots to be trained with data from a JSON file or
    directory of JSON files.

    :param dict field_map: A dictionary containing the database column name to header mapping.
    """

    DEFAULT_STATEMENT_TO_KEY_MAPPING = {
        'text': 'text',
        'conversation': 'conversation',
        'created_at': 'created_at',
        'in_response_to': 'in_response_to',
        'persona': 'persona',
        'tags': 'tags'
    }

    def __init__(self, chatbot: ChatBot, **kwargs):
        super().__init__(chatbot, **kwargs)

        self.file_extension = 'json'

        self.field_map = kwargs.get(
            'field_map',
            self.DEFAULT_STATEMENT_TO_KEY_MAPPING
        )


class UbuntuCorpusTrainer(CsvFileTrainer):
    """
    .. note::
        PENDING DEPRECATION: Please use the ``CsvFileTrainer`` for data formats similar to this one.

    Allow chatbots to be trained with the data from the Ubuntu Dialog Corpus.

    For more information about the Ubuntu Dialog Corpus visit:
    https://dataset.cs.mcgill.ca/ubuntu-corpus-1.0/

    :param str ubuntu_corpus_data_directory: The directory where the Ubuntu corpus data is already located, or where it should be downloaded and extracted.
    """

    def __init__(self, chatbot: ChatBot, **kwargs):
        super().__init__(chatbot, **kwargs)
        home_directory = os.path.expanduser('~')

        self.data_download_url = None

        self.data_directory = kwargs.get(
            'ubuntu_corpus_data_directory',
            os.path.join(home_directory, 'ubuntu_data')
        )

        # Directory containing extracted data
        self.data_path = os.path.join(
            self.data_directory, 'ubuntu_dialogs'
        )

        self.field_map = {
            'text': 3,
            'created_at': 0,
            'persona': 1,
        }

    def is_downloaded(self, file_path: str):
        """
        Check if the data file is already downloaded.
        """
        if os.path.exists(file_path):
            self.chatbot.logger.info('File is already downloaded')
            return True

        return False

    def is_extracted(self, file_path: str):
        """
        Check if the data file is already extracted.
        """

        if os.path.isdir(file_path):
            self.chatbot.logger.info('File is already extracted')
            return True
        return False

    def download(self, url: str, show_status=True):
        """
        Download a file from the given url.
        Show a progress indicator for the download status.
        """
        import requests

        # Create the data directory if it does not already exist
        if not os.path.exists(self.data_directory):
            os.makedirs(self.data_directory)

        file_name = url.split('/')[-1]
        file_path = os.path.join(self.data_directory, file_name)

        # Do not download the data if it already exists
        if self.is_downloaded(file_path):
            return file_path

        with open(file_path, 'wb') as open_file:
            if show_status:
                print('Downloading %s' % url)
            response = requests.get(url, stream=True)
            total_length = response.headers.get('content-length')

            if total_length is None:
                # No content length header
                open_file.write(response.content)
            else:
                for data in tqdm(
                    response.iter_content(chunk_size=4096),
                    desc='Downloading',
                    disable=not show_status
                ):
                    open_file.write(data)

        if show_status:
            print('Download location: %s' % file_path)
        return file_path

    def extract(self, file_path: str):
        """
        Extract a tar file at the specified file path.
        """
        if not self.disable_progress:
            print('Extracting {}'.format(file_path))

        if os.path.islink(self.data_path):
            raise self.TrainerInitializationException(
                'Refusing to extract archive to a symbolic link: {}'.format(self.data_path)
            )

        if not os.path.exists(self.data_path):
            os.makedirs(self.data_path)

        def is_within_directory(directory, target):

            abs_directory = os.path.realpath(directory)
            abs_target = os.path.realpath(target)

            prefix = os.path.commonprefix([abs_directory, abs_target])

            return prefix == abs_directory

        def safe_extract(tar, path='.', members=None, *, numeric_owner=False):

            for member in tar.getmembers():
                if member.issym() or member.islnk():
                    raise Exception('Symlinks and hard links are not permitted in the tar file')
                member_path = os.path.join(path, member.name)
                if not is_within_directory(path, member_path):
                    raise Exception('Attempted Path Traversal in Tar File')

            tar.extractall(path, members, numeric_owner=numeric_owner)

        try:
            with tarfile.open(file_path, 'r') as tar:
                safe_extract(tar, path=self.data_path, members=tqdm(tar, disable=self.disable_progress))
        except tarfile.ReadError as e:
            raise self.TrainerInitializationException(
                f'The provided data file is not a valid tar file: {file_path}'
            ) from e

        self.chatbot.logger.info('File extracted to {}'.format(self.data_path))

        return True

    def _get_file_list(self, data_path: str, limit: Union[int, None]):
        """
        Get a list of files to read from the data set.
        """

        if self.data_download_url is None:
            raise self.TrainerInitializationException(
                'The data_download_url attribute must be set before calling train().'
            )

        # Download and extract the Ubuntu dialog corpus if needed
        corpus_download_path = self.download(self.data_download_url)

        # Extract if the directory does not already exist
        if not self.is_extracted(data_path):
            self.extract(corpus_download_path)

        extracted_corpus_path = os.path.join(
            data_path, '**', '**', '*.tsv'
        )

        # Use iglob instead of glob for better performance with
        # large directories because it returns an iterator
        data_files = glob.iglob(extracted_corpus_path)

        for index, file_path in enumerate(data_files):
            if limit is not None and index >= limit:
                break

            yield file_path

    def train(self, data_download_url: str, limit: Union[int, None] = None):
        """
        :param str data_download_url: The URL to download the Ubuntu dialog corpus from.
        :param int limit: The maximum number of files to train from.
        """
        self.data_download_url = data_download_url

        start_time = time.time()
        super().train(self.data_path, limit=limit)

        if not self.disable_progress:
            print('Training took', time.time() - start_time, 'seconds.')
```
