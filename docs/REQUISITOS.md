# Feature: API de diagnósticos y reparaciones + chatbot de taller

Rama: `feature/api-diagnosticos-reparaciones` (desde `develop`)

## 1. Objetivo
Aplicación web que funciona como **chatbot de asistencia de taller mecánico**. Responde
preguntas sobre diagnóstico y reparación basándose **exclusivamente en los datos de la
base MongoDB Atlas `taller_db`**, y usa un **LLM gratuito (Gemini)** para redactar la
respuesta de forma natural.

## 2. Alcance
- **Solo lectura** sobre MongoDB (sin altas, ediciones ni borrados).
- Backend: **Minimal API en C# (.NET)**.
- Frontend: **Vue 3 + Vite** (recomendado; ver sección 6).
- Fuera de alcance por ahora: autenticación de usuarios, escritura de datos, historial persistente de chats.

## 3. Modelo de datos (colección de conocimiento en `taller_db`)
Documentos tipo "chunk" de conocimiento técnico.

| Campo | Tipo | Obligatorio |
|---|---|---|
| `_id` | ObjectId | Sí |
| `chunk_id`, `category`, `sub_category`, `title` | string | Sí |
| `tags` | string[] | Sí |
| `sintomatologia`, `procedimiento_resolucion`, `requisitos_previos`, `buenas_practicas`, `intervalos_servicio` | string | No |
| `causas_probables`, `normativas_clave`, `pasos_desenergizacion` | string[] | No |
| `especificaciones` | objeto (`cadena_distribucion`, `correa_distribucion`) | No |

Los campos opcionales varían según el documento, por lo que el acceso desde C# debe ser
tolerante (modelo con propiedades nulables o `BsonDocument`).

## 4. Requisitos funcionales
- **RF1.** `GET /api/conocimiento` lista y filtra por `category`, `sub_category` y `tags`, con paginación.
- **RF2.** `GET /api/conocimiento/{chunkId}` devuelve un documento.
- **RF3.** `GET /api/conocimiento/buscar?q=` búsqueda de texto sobre título, tags, sintomatología y causas.
- **RF4.** `POST /api/chat` recibe la pregunta del usuario y:
  1. recupera de MongoDB los documentos más relevantes (RAG),
  2. construye un prompt con ese contexto,
  3. consulta a Gemini y devuelve la respuesta más las fuentes (`chunk_id`, `title`) usadas.
- **RF5.** Si no hay datos relevantes en la BD, el chatbot lo indica y **no inventa** información.
- **RF6.** El frontend ofrece una interfaz de chat con la respuesta y las fuentes consultadas.

## 5. Requisitos no funcionales
- **RNF1.** Cadena de conexión a Atlas y API key de Gemini fuera del repo: *user-secrets* en desarrollo, variables de entorno en despliegue.
- **RNF2.** La API key de Gemini solo se usa en el backend, nunca en el navegador.
- **RNF3.** Usuario de Atlas con permisos de **solo lectura** y acceso por IP restringido.
- **RNF4.** CORS limitado al origen del frontend.
- **RNF5.** Límite de peticiones (rate limiting) en `/api/chat` para respetar la cuota gratuita de Gemini y gestionar sus errores (429) con un mensaje claro.
- **RNF6.** El prompt del sistema obliga a responder solo con el contexto recuperado y en español.
- **RNF7.** Aviso en la UI de que las respuestas orientan pero no sustituyen el criterio del mecánico, sobre todo en `pasos_desenergizacion` y `normativas_clave`.

## 6. Decisiones técnicas
- **Driver:** `MongoDB.Driver` oficial.
- **Recuperación:** empezar con índice de texto de MongoDB (título, tags, sintomatología, causas). Evolucionar a Atlas Search o búsqueda vectorial si la precisión no basta.
- **LLM:** Gemini vía API de Google AI Studio (capa gratuita). Verificar el modelo y los límites de cuota vigentes antes de implementar, porque cambian con frecuencia. Encapsular el acceso tras una interfaz (`ILlmClient`) para poder cambiar de proveedor.
- **Frontend:** Vue 3 + Vite es más ligero y rápido de montar para una interfaz de chat con pocas pantallas. Angular sería preferible si el equipo ya lo domina o se prevé una app grande con muchos módulos.

## 7. Criterios de aceptación
- [ ] La API arranca y se conecta a Atlas usando solo configuración externa.
- [ ] Los endpoints RF1–RF3 devuelven datos reales de `taller_db`.
- [ ] Una pregunta como "el motor tiene tirones al acelerar" devuelve una respuesta en lenguaje natural que cita los `chunk_id` usados.
- [ ] Una pregunta sin datos relacionados recibe una respuesta de "no tengo información", sin datos inventados.
- [ ] El frontend permite conversar y ver las fuentes.
- [ ] Ningún secreto aparece en el repositorio.

## 8. Dudas abiertas
- Nombre exacto de la colección en `taller_db`.
- Volumen de documentos (afecta a la estrategia de recuperación).
- Versión de .NET a usar (se propone la LTS vigente).
- Despliegue previsto (local, Azure, otro).
