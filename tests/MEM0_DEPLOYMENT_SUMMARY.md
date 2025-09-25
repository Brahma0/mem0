# Mem0 Deployment Summary

This document summarizes the key configurations, commands, and architectural decisions made during the deployment of the Mem0 service within the TIANCE project.

---

## 1. Final Architecture

The Mem0 service is deployed using a hybrid model that leverages existing infrastructure while providing necessary isolated components.

-   **Vector Store**: Reuses the existing `tiance-postgres` service (PostgreSQL 15.4 with pgvector extension).
-   **Graph Store**: A new, dedicated `mem0-neo4j` container.
-   **API Service**: A new, dedicated `mem0-server` container built from the local repository.
-   **Network**: All services are connected to the pre-existing `tiance-network` for seamless communication.
-   **LLM Provider**: Configured to use **Google Gemini 2.5 Pro** via API key.

---

## 2. Core Configuration Files

### `docker-compose.mem0.yml`
*(Located in the project root)*
```yaml
version: '3.8'

services:
  mem0-server:
    build:
      context: ./mem0
      dockerfile: server/Dockerfile
    container_name: mem0-server
    restart: unless-stopped
    ports:
      - "8001:8000"
    env_file:
      - mem0.env
    environment:
      - POSTGRES_HOST=tiance-postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=tiance
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_COLLECTION_NAME=mem0_memories
      - NEO4J_URI=bolt://mem0-neo4j:7687
      - NEO4J_USERNAME=neo4j
      - NEO4J_PASSWORD=mem0graph
      - LLM_PROVIDER=google
    depends_on:
      - mem0-neo4j
    networks:
      - tiance-network

  mem0-neo4j:
    image: neo4j:5.20.0
    container_name: mem0-neo4j
    restart: unless-stopped
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/mem0graph
      - NEO4J_PLUGINS=["apoc"]
    volumes:
      - ./data/neo4j/data:/data
      - ./data/neo4j/logs:/logs
    networks:
      - tiance-network

networks:
  tiance-network:
    external: true
```

### `mem0.env`
*(Located in the project root)*
```env
# --- LLM Provider Configuration ---
GOOGLE_API_KEY=AIzaSyBk6T-5NSE8Vgi_nHv_ZzExQcERW_DE3ik

# --- Mem0 Service Configuration ---
LOG_LEVEL=INFO
```

---

## 3. Service Management Commands

All commands should be run from the project root directory (`/Users/a10645/.../Tiance`).

### Start Mem0 Services
```bash
docker-compose -f docker-compose.mem0.yml --env-file mem0.env up -d --build
```

### Stop Mem0 Services
```bash
docker-compose -f docker-compose.mem0.yml down
```

### View Logs
```bash
# View API server logs
docker logs -f mem0-server

# View Neo4j logs
docker logs -f mem0-neo4j
```

### Check Status
```bash
docker ps
```

---

## 4. Access Endpoints & Credentials

-   **Mem0 API Documentation**:
    -   **URL**: `http://localhost:8001/docs`

-   **Neo4j Browser Interface**:
    -   **URL**: `http://localhost:7474`
    -   **Username**: `neo4j`
    -   **Password**: `mem0graph`

-   **PostgreSQL Database (Shared)**:
    -   **Host (from local machine)**: `localhost`
    -   **Port**: `15432`
    -   **Database**: `tiance`
    -   **Username**: `postgres`
    -   **Password**: `postgres`

---

## 5. Key Decisions & Troubleshooting Summary

-   **PostgreSQL Migration**: The existing `tiance-postgres` database was migrated from PostgreSQL 16 to a PostgreSQL 15.4-based image (`ankane/pgvector:v0.5.1`) to support the `pgvector` extension required by Mem0.
-   **Data Persistence**: PostgreSQL data was migrated from a Docker-managed volume to an external host directory (`./data/postgres`) for better security and accessibility.
-   **File System Permissions**: Encountered and resolved several `Read-only file system` errors, likely caused by a combination of the IDE's tool environment and OneDrive syncing. Restarting the machine and correcting file ownership/permissions were key steps.
-   **Dockerfile Context**: Corrected the `Dockerfile` for `mem0-server` to use the correct relative paths for `COPY` commands based on the build context defined in the `docker-compose.yml`.

This document serves as a comprehensive guide to the deployed Mem0 instance.

---

## 6. API Integration Guide for TIANCE Agents

This section outlines how to integrate Mem0's memory capabilities into your TIANCE agents using the REST API.

### Core Concepts
-   **`user_id` is Key**: Every memory is associated with a `user_id`. This is crucial for maintaining personalized memories for different users of your agent. Ensure your agent passes the correct `user_id` with each API call.
-   **Conversation Context**: Mem0 is designed to process conversations. You typically send a list of messages (user input and agent response) to the `add` endpoint, and Mem0 intelligently extracts and stores the key information.

### Main API Endpoints
The Mem0 API is accessible at `http://localhost:8001`.

| Method | Endpoint | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `POST` | `/memories` | **Add Memory**: Creates new memories from a conversation. | `messages`, `user_id` |
| `POST` | `/search` | **Search Memory**: Retrieves relevant memories based on a query. | `query`, `user_id` |
| `GET` | `/memories` | **Get All Memories**: Retrieves all memories for a specific user. | `user_id` |
| `GET` | `/memories/{memory_id}` | **Get a Specific Memory**: Retrieves a single memory by its ID. | `memory_id` |
| `PUT`| `/memories/{memory_id}`| **Update a Memory**: Modifies the content of an existing memory. | `memory_id`, `data` |
| `DELETE` | `/memories/{memory_id}` | **Delete a Memory**: Deletes a single memory. | `memory_id` |
| `DELETE` | `/memories` | **Delete All Memories**: Deletes all memories for a specific user. | `user_id` |

*For full details, visit the interactive API documentation at `http://localhost:8001/docs`.*

### Python Integration Example
Here is a practical example of how a TIANCE agent can use Mem0's API to have a contextual conversation.

```python
import requests
import json

# --- Configuration ---
MEM0_API_BASE_URL = "http://localhost:8001"
CURRENT_USER_ID = "user_12345" # This should be dynamically set for each user

def get_contextual_response(user_input: str) -> str:
    """
    Gets a response from an LLM, enriched with context from Mem0.
    """
    # 1. Search for relevant memories before generating a response
    try:
        search_payload = {
            "query": user_input,
            "user_id": CURRENT_USER_ID
        }
        response = requests.post(f"{MEM0_API_BASE_URL}/search", json=search_payload)
        response.raise_for_status()
        search_results = response.json().get("results", [])
        
        # Format the memories as context for the LLM
        memory_context = "\n".join([f"- {item['memory']}" for item in search_results])
        
    except requests.exceptions.RequestException as e:
        print(f"Error searching memories: {e}")
        memory_context = "No memories found."

    # 2. Generate a response using the LLM (simulated here)
    # In a real agent, you would pass 'user_input' and 'memory_context' to your LLM
    print(f"--- Sending to LLM with context ---\n{memory_context}\n---------------------------------")
    agent_response = f"Based on what I remember, here is a response to '{user_input}'."

    # 3. Add the new conversation turn to Mem0 to update the memory
    try:
        add_payload = {
            "messages": [
                {"role": "user", "content": user_input},
                {"role": "assistant", "content": agent_response}
            ],
            "user_id": CURRENT_USER_ID
        }
        response = requests.post(f"{MEM0_API_BASE_URL}/memories", json=add_payload)
        response.raise_for_status()
        print("Successfully added new conversation to memory.")
        
    except requests.exceptions.RequestException as e:
        print(f"Error adding memory: {e}")
        
    return agent_response

# --- Example Usage ---
if __name__ == "__main__":
    print("Agent is ready. Type 'exit' to quit.")
    while True:
        prompt = input("You: ")
        if prompt.lower() == 'exit':
            break
        
        response = get_contextual_response(prompt)
        print(f"Agent: {response}")

```

### Important Considerations

-   **Error Handling**: Always wrap your API calls in `try...except` blocks to handle potential network issues or API errors gracefully.
-   **User Identification**: The `user_id` is the cornerstone of personalization. Ensure your agent's user management system provides a unique and stable ID for each user that can be passed to Mem0.
-   **Asynchronous Calls**: For better performance in an async framework like FastAPI (which TIANCE uses), use an async HTTP client like `httpx` instead of `requests`.
-   **Performance**: The `/search` endpoint is designed to be called *before* you generate a response, providing the necessary context to your LLM. The `/memories` (add) endpoint is called *after* a conversation turn to record what was discussed.
-   **Configuration**: The Mem0 service itself is configured via the `.env.mem0` file. If you need to change the LLM or other settings, you can modify this file and restart the `mem0-server` container.
