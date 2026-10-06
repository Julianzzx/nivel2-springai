# Nivel 2 — Spring AI + Ollama (local)

Desarrollo Web Avanzado — Laboratorio LLM con Spring Boot

API REST de chat construida con **Spring Boot 3.5.6** y **Spring AI 1.0.3**. El modelo de lenguaje se ejecuta **de forma local con Ollama**, dentro de un contenedor Docker, sin API keys ni servicios externos.

| Elemento | Detalle |
|---|---|
| Modelo de lenguaje | `llama3.2:1b` (Ollama en Docker) |
| Integración | `spring-ai-starter-model-ollama` |
| Framework | Spring Boot 3.5.6, Java 21 |
| Aplicación | Puerto `8081`, endpoints bajo `/api/v2/chat` |
| Servicio Ollama | `http://localhost:11434` |

## ¿Por qué Ollama con Spring AI?

Ollama ejecuta modelos de lenguaje en la propia máquina y los expone por una API HTTP. Spring AI lo soporta con un *starter* que autoconfigura el `ChatModel` a partir de `application.yml`. El servicio solo usa el `ChatClient` de Spring AI, por lo que el código Java no depende del modelo concreto. Además, al ser local, los datos de las consultas no salen de la máquina.

## Estructura del proyecto

```
nivel2-springai/
├── pom.xml                                   ← Spring AI BOM + spring-ai-starter-model-ollama
└── src/main/
    ├── java/com/universidad/chatbot/
    │   ├── Nivel2SpringaiApplication.java     ← Punto de entrada
    │   ├── config/
    │   │   ├── ChatDtos.java                  ← Records: ChatRequest y ChatResponse
    │   │   └── GlobalExceptionHandler.java    ← Manejo de errores amigable
    │   ├── service/
    │   │   └── SpringAiChatService.java       ← NÚCLEO: ChatClient de Spring AI
    │   └── controller/
    │       └── SpringAiChatController.java    ← Endpoints REST /api/v2/chat
    └── resources/
        └── application.yml                    ← URL de Ollama, modelo y parámetros
```

## Requisitos

- JDK 21
- Maven (o un IDE con Maven integrado, como VS Code con *Extension Pack for Java* o IntelliJ IDEA)
- Docker Desktop con el motor en ejecución (`Engine running`)
- Espacio libre en disco: la imagen de Ollama pesa unos 3.6 GB y el modelo, 1.3 GB

## Pasos para ejecutar

### 1. Levantar Ollama con Docker

```bash
docker run -d -p 11434:11434 --name ollama ollama/ollama
```

`-d` lo deja en segundo plano, `-p` publica el puerto 11434 y `--name` le asigna el nombre `ollama`. La primera vez descarga la imagen desde Docker Hub.

> Opcional: agrega `-v ollama:/root/.ollama` al comando para conservar los modelos si se recrea el contenedor.

### 2. Descargar el modelo dentro del contenedor

```bash
docker exec ollama ollama pull llama3.2:1b
```

### 3. Verificar el despliegue

```bash
docker ps
docker exec ollama ollama list
```

`docker ps` debe mostrar el contenedor `ollama` en estado `Up` con el puerto `0.0.0.0:11434->11434/tcp`, y `ollama list` debe incluir `llama3.2:1b`. Al abrir `http://localhost:11434` en el navegador debe aparecer **Ollama is running**.

### 4. Ejecutar la aplicación

Desde el IDE: abrir `Nivel2SpringaiApplication` y pulsar **Run**. O por consola, desde la carpeta del proyecto:

```bash
mvn spring-boot:run
```

La consola debe mostrar `Started Nivel2SpringaiApplication`.

## Configuración

`src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: nivel2-springai
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2:1b
          temperature: 0.7
          num-predict: 512

server:
  port: 8081
```

No se necesita ninguna API key ni variable de entorno.

## Probar los endpoints

### Endpoint 1 — POST con JSON (principal)

En CMD:

```bash
curl -X POST http://localhost:8081/api/v2/chat -H "Content-Type: application/json" -d "{\"pregunta\": \"¿Qué es Spring Boot en 2 oraciones?\", \"dominio\": \"Java\"}"
```

En PowerShell (usar `curl.exe` y comillas simples para el JSON):

```powershell
curl.exe -X POST http://localhost:8081/api/v2/chat -H "Content-Type: application/json" -d '{"pregunta": "¿Qué es Spring Boot en 2 oraciones?", "dominio": "Java"}'
```

Respuesta esperada:

```json
{
  "respuesta": "Spring Boot es un framework...",
  "modelo": "(etiqueta fija del controlador)",
  "dominio": "Java"
}
```

> El campo `modelo` es una etiqueta escrita en el controlador y no refleja el modelo real. El modelo que genera la respuesta es el configurado en `application.yml` (`llama3.2:1b`).

### Endpoint 2 — POST con parámetros en la URL

```bash
curl -X POST "http://localhost:8081/api/v2/chat/rapido?pregunta=Explica%20Maven&dominio=Java"
```

### Endpoint 3 — GET verificación de salud

```bash
curl http://localhost:8081/api/v2/chat/salud
```

## Comandos útiles de Docker y Ollama

| Comando | Para qué sirve |
|---|---|
| `docker ps` | Ver si el contenedor está corriendo |
| `docker start ollama` | Arrancar el contenedor (por ejemplo, tras reiniciar el equipo) |
| `docker stop ollama` | Detenerlo para liberar memoria |
| `docker exec ollama ollama list` | Listar los modelos descargados |
| `docker exec ollama ollama ps` | Ver el modelo cargado en memoria |

Para usar otro modelo: `docker exec ollama ollama pull <modelo>` y cambiar `model` en `application.yml`.

## Troubleshooting

| Error | Causa | Solución |
|---|---|---|
| `Connection refused` al llamar al chat | El contenedor de Ollama está detenido | `docker start ollama` |
| `model "llama3.2:1b" not found` | El modelo no se descargó dentro del contenedor | `docker exec ollama ollama pull llama3.2:1b` |
| La primera respuesta tarda | Ollama carga el modelo en memoria | Esperar unos segundos; las siguientes son más rápidas |
| Respuesta cortada | Límite `num-predict: 512` | Subir `num-predict` en `application.yml` y reiniciar |
| `Failed to connect` a `localhost:8081` | La aplicación no está corriendo | Ejecutar la aplicación y revisar la consola |
| Puerto 8081 ocupado | Otro proceso usa el puerto | Cambiar `server.port` en `application.yml` |
| Error de parseo JSON en PowerShell | Comillas escapadas al estilo CMD | Usar `curl.exe` con el JSON entre comillas simples |
| `Unsupported class file major version` | JDK distinto al esperado | Usar JDK 21 |
| Docker no logra descargar la imagen | Conexión inestable o motor trabado | Reiniciar Docker Desktop y repetir `docker pull ollama/ollama` |
| Fallos por falta de memoria | Docker/WSL consume demasiada RAM | Limitar la memoria de WSL con un archivo `.wslconfig` (`memory=4GB`) y ejecutar `wsl --shutdown` |
