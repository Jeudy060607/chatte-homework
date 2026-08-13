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
