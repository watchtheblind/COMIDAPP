# Food App — Smart Food Aggregator

**App Comida** is a mobile and web platform designed as a smart, local *"Trivago"*-style food aggregator. Its main purpose is to centralize, standardize, and unify the fragmented restaurant information currently scattered across social media and instant messaging, fully automating data extraction through Artificial Intelligence.

## Table of Contents

 1. [Project Description and Problem to be Solved](#descripción-del-proyecto-y-problema-a-resolver)

 2. [Technology Stack](#stack-tecnológico)

 3. [System Architecture and Diagrams](#arquitectura-y-diagramas-del-sistema)

    * [1. Architecture Diagram (Modular Monolith)](#1-diagrama-de-arquitectura-monolito-modular)

    * [2. Use Case Diagram](#2-diagrama-de-casos-de-uso)

    * [3. Entity-Relationship Diagram (ERD)](#3-diagrama-entidad-relación-der)

    * [4. Data Flow Diagram: Multi-Source Ingestion](#4-diagrama-de-flujo-de-datos-ingesta-multifuente)

 4. [Project Structure](#estructura-del-proyecto)

 5. [Relational Database Schema](#esquema-de-base-de-datos-relacional)

 6. [Intelligent Ingestion Pipeline and Key Endpoints](#pipeline-de-ingesta-inteligente-y-endpoints-clave)

 7. [Local Installation and Configuration](#instalación-y-configuración-local)

 8. [Use of the Platform](#uso-de-la-plataforma)

 9. [Infrastructure, Security and Performance](#infraestructura-seguridad-y-rendimiento)

10. [Roadmap](#hoja-de-ruta-roadmap)

11. [Contribution and License](#contribución-y-licencia)

## Project Description and Problem to be Solved

### The Problem

Today, finding places to eat or comparing prices and menus locally involves navigating a highly fragmented ecosystem:

* Menus published only in **Instagram Highlights or Reels** using images with floating text.

* Static catalogs in **PDF files** sent manually via **WhatsApp**.

* External links on platforms like **Beacons or Linktree** without the ability to search by dish or budget.

### The Solution

**Food App** eliminates friction for both the diner and the catalog management:

* **Centralization and Comparison:** A fast and clean search engine that allows you to filter places by specific dish, culinary category, geographical area or price range.

* **AI-Powered Automated Ingestion:** Instead of manually transcribing menus, the system allows you to enter a URL (Beacons/Linktree) or upload a screenshot/PDF. A multimodal engine (local Ollama or Gemini API) extracts, structures, and validates the information in JSON format, ready to be persisted in the database.

* **Direct Action:** Each restaurant listing offers immediate access to contact and location channels (direct links) `wa.me`, Google Maps e Instagram).

## Technology Stack

The project is built on modern technologies geared towards high asynchronous performance, strict data validation, and modular scalability:

* **Backend / Core API:** Python 3.11+ with **FastAPI** (REST architecture, strict validation with Pydantic v2 and automatic interactive documentation via Swagger/OpenAPI in `/docs`).

* **Main Database:** **PostgreSQL** with asynchronous driver `asyncpg` and ORM **SQLAlchemy** (ready to scale with geospatial queries using PostGIS).

* **Cache, Queues and Rate Limiting:** **Redis** for temporary storage of frequently searched queries and protection against abuse on AI endpoints.

* **AI Extraction Engine:** **Gemini API** (multimodal models) and support for **Ollama** (local execution of LLMs and vision) along with `HTTPX` / `Playwright` for web scraping.

* **Frontend (Initial Web/PWA Phase):** **Next.js** or **Vite + Tailwind CSS** optimized for mobile devices.

* **Frontend (Native Mobile Phase):** **React Native** with **Expo** for cross-platform deployment (iOS and Android).

* **Infrastructure and Network:** **Nginx** as reverse proxy and web server, secured and accelerated at the edge by **Cloudflare**.

## System Architecture and Diagrams

To avoid over-engineering and operational complexity of microservices in early stages, but ensuring that business domains remain strictly decoupled, a **Modular Monolith** architecture was implemented.

### 1. Architecture Diagram (Modular Monolith)

```mermaid
flowchart TD
    subgraph Client ["Client Layer"]
        App["React Native App / Web PWA"]
    end

    subgraph Edge ["Capa de Red & Proxy"]
        CF["Cloudflare (CDN / SSL / WAF)"]
        Nginx["Nginx (Reverse Proxy)"]
    end

    subgraph Backend ["Backend Monolito Modular (FastAPI)"]
        API["FastAPI Core"]
        M_Rest["Restaurants Module"]
        M_Menu["Menu Module"]
        M_AI["IA Intake Module"]
    end

    subgraph Data ["Data & Services Layer"]
        PG[("PostgreSQL\n(Base Relacional)")]
        Redis[("Redis\n(Hidden & Rate Limit)")]
        AI["Ollama (Local) / Gemini API\n(Text and Image Extraction)"]
    end

    App -->|HTTPS| CF -->|Proxy Pass| Nginx -->|Uvicorn / ASGI| API
    API --> M_Rest 
    API --> M_Menu 
    API --> M_AI
    
    M_Rest & M_Menu -->|SQLAlchemy / Asyncpg| PG
    API -->|Hidden / Tokens| Redis
    M_AI -->|Prompt / JSON Mode| AI

```

### 2. Use Case Diagram

```mermaid
flowchart LR
    User((Usuario / Dev)) --> UC1[Enter URL of Beacons / Menu Image]
    User --> UC2[Search and Filter by Dish or Zone]
    User --> UC3[View Profile and Open WhatsApp / Maps / IG]
    
    subgraph System ["Food App (FastAPI Backend)"]
        UC1 --> UC4[Process raw content with AI / Ollama]
        UC4 --> UC5[Validate JSON structure and save to database]
        UC2 --> UC6[Check indexes and cache]
        UC3 --> UC6
    end

```

### 3. Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    RESTAURANTS {
        uuid id PK
        varchar name
        varchar slug
        varchar category
        varchar zone
        varchar instagram_handle
        varchar whatsapp_number
        text maps_url
        int price_tier
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    MENU_CATEGORIES {
        serial id PK
        uuid restaurant_id FK
        varchar name
        int display_order
    }

    MENU_ITEMS {
        serial id PK
        int category_id FK
        varchar name
        text description
        decimal price
        varchar currency
        boolean is_available
        timestamp updated_at
    }

    RAW_SOURCES {
        serial id PK
        uuid restaurant_id FK
        varchar source_type
        text raw_content
        jsonb processed_data
        varchar status
        timestamp created_at
    }

    RESTAURANTS ||--o{ MENU_CATEGORIES : "contiene"
    MENU_CATEGORIES ||--o{ MENU_ITEMS : "agrupa"
    RESTAURANTS ||--o{ RAW_SOURCES : "registra fuente"

```

### 4. Data Flow Diagram: Multi-Source Ingestion

```mermaid
flowchart TD
    subgraph Sources ["External Data Sources (Chaos)"]
        IG_Story["📸 Instagram Stories\n(Screenshot / Floating Text)"]
        IG_Post["🖼️ IG Posts / Reels\n(Description / Visual Menu)"]
        WA_PDF["📄 Catalogs / PDFs\n(Sent via WhatsApp)"]
        Beacons["🔗 Links Externos\n(Beacons / Linktree)"]
    end

    subgraph Capture ["Entry Point (Ingestion)"]
        Input["Send to Bot / Upload Endpoint\n(URL or Base64 Image)"]
    end

    subgraph Core_IA ["Intelligent Extraction Engine (FastAPI)"]
        Scraper["Scraper Web / HTML Parser\n(Para Beacons / Links)"]
        OCR_Vision["Multimodal Model / OCR\n(Local Name or Gemini API)"]
        Parser["Strict Parser (Pydantic)\n(Normalizes to structured JSON)"]
    end

    subgraph Storage ["Relational Database (PostgreSQL)"]
        Raw["raw_sources\n(Audit / Raw Backup)"]
        Rest["restaurants\n(Main Listing)"]
        Categories["menu_categories\n(Starters, Main Courses, Drinks)"]
        Items["menu_items\n(Dishes and Prices)"]
    end

    IG_Story --> Input
    IG_Post --> Input
    WA_PDF --> Input
    Beacons --> Input

    Input -->|Es URL de link| Scraper
    Input -->|Es Imagen / PDF| OCR_Vision
    Scraper --> Parser
    OCR_Vision --> Parser

    Parser -->|Trace Keeper| Raw
    Parser -->|Create / Update| Rest
    Rest --> Categories
    Categories --> Items

```

## Project Structure

The backend source code follows a clear separation of responsibilities between the cross-core (`core/`) and the domain modules (`modules/`):

```
app/
├── core/
│ ├── config.py # Centralized configuration with Pydantic Settings (.env)
│ ├── database.py # Managing async sessions to PostgreSQL (SQLAlchemy / Asyncpg)
│ ├── redis.py # Redis client for query caching and rate limiting
│ └── security.py # Authentication and validation of API keys (ingestion protection)
├── modules/
│   ├── restaurants/
│ │ ├── router.py # REST endpoints for restaurant management and search
│ │ ├── models.py # ORM (SQLAlchemy) models of restaurants
│ │ ├── schemas.py # Pydantic Schemas (Input/Output Validation)
│ │ └── service.py # Business logic and filtering rules
│   ├── menus/
│ │ ├── router.py # Endpoints for dishes, prices, and categories
│ │ ├── models.py # ORM models for categories and menu items
│ │ ├── schemas.py # Pydantic schemas for menu structures
│ │ └── service.py # Logic for updating and availability of dishes
│ └── ai_ingestion/
│ ├── router.py # Endpoints for processing URLs/images with AI
│ ├── prompts.py # Structured prompts and extraction rules for LLM
│ └── service.py # Gemini API Client / Ollama and Multimodal Extractors
├── main.py # Entry point, FastAPI initialization, and router mounting
└── requirements.txt # Python project dependencies

```

## Relational Database Schema

Persistence in **PostgreSQL** is divided into four fundamental entities designed to maintain referential integrity and preserve complete traceability of each extraction performed by the AI:

### 1. `restaurants` (Main Sheet)

* `id` (`UUID` / `SERIAL`, PK): Unique identifier of the establishment.

* `name` (`VARCHAR`, Not Null): Restaurant trade name.

* `slug` (`VARCHAR`, Unique): User-friendly identifier for URLs.

* `category` (`VARCHAR`): Main gastronomic category (e.g., *Cafeteria, Fast Food, Sushi*).

* `zone` (`VARCHAR`): Sector, urbanization or municipality of location.

* `instagram_handle` (`VARCHAR`(Nullable): Official Instagram user.

* `whatsapp_number` (`VARCHAR`(Nullable): International format phone number ready for links `wa.me`.

* `maps_url` (`TEXT`(Nullable): Direct link to the location on Google Maps.

* `price_tier` (`INTEGER`): Numerical indicator of price range (`1` a `3`).

* `is_active` (`BOOLEAN`, Default `True`): Visibility status in the catalog.

* `created_at` / `updated_at` (`TIMESTAMP`): Audit timestamps.

### 2. `menu_categories` (Menu Sections)

* `id` (`SERIAL`, PK): Category identifier.

* `restaurant_id` (`UUID`, FK -> `restaurants.id`): Relationship with the restaurant owner.

* `name` (`VARCHAR`): Section name (e.g., *Starters, Main Courses, Drinks, Desserts*).

* `display_order` (`INTEGER`): Order of presentation in the user interface.

### 3. `menu_items` (Dish and Price Catalog)

* `id` (`SERIAL`, PK): Unique identifier of the dish.

* `category_id` (`INTEGER`, FK -> `menu_categories.id`): Category to which it belongs.

* `name` (`VARCHAR`): Name of the dish or product.

* `description` (`TEXT`(Nullable): Ingredients, details or accompaniments.

* `price` (`DECIMAL(10,2)`): Numerical price of the product.

* `currency` (`VARCHAR`): Currency code (`USD`, `VES`, etc.).

* `is_available` (`BOOLEAN`, Default `True`): Current availability of the dish.

* `updated_at` (`TIMESTAMP`): Latest price or availability update.

### 4. `raw_sources` (Intake Audit and Backup)

* `id` (`SERIAL`, PK): Audit record identifier.

* `restaurant_id` (`UUID`, FK -> `restaurants.id`(Nullable): Associated restaurant after approving the intake.

* `source_type` (`VARCHAR`): Data source (`URL_BEACONS`, `INSTAGRAM_IMAGE`, `WHATSAPP_PDF`).

* `raw_content` (`TEXT`): Raw extracted text or source URL.

* `processed_data` (`JSONB`): Raw JSON object returned by the AI ​​model before its final normalization.

* `status` (`VARCHAR`): Review flow status (`PENDING`, `APPROVED`, `REJECTED`).

* `created_at` (`TIMESTAMP`): Date and time of capture.

## Intelligent Ingestion Pipeline and Key Endpoints

### Step-by-Step Flow

1. **Entry Capture:** The user or administrator sends a URL (Beacons/Linktree) or uploads a screenshot/PDF of the menu.

2. **Gross Extraction:**

   * *If it's a URL:* An auxiliary scraper based on `HTTPX` / `Playwright` extracts clean text and relevant links.

   * *If it is an image or PDF:* The file is encoded in Base64 and sent along with a structured multimodal prompt to the Gemini API or a local model in Ollama.

3. **Strict Parsing (JSON Mode):** The LLM returns structured information, which is automatically validated using **Pydantic** schemas in FastAPI.

4. **Review and Persistence:** The trace is stored in `raw_sources` and, after confirming the structure, the records are created or updated in `restaurants`, `menu_categories` and `menu_items`.

### Main REST API Endpoints

| Method | Endpoint | Description | 
| ----- | ----- | ----- | 
| `POST` | `/restaurants/parse-ai` | Receives text, URLs, or images encoded in Base64, processes them using AI, and returns structured JSON ready for review. 
| `POST` | `/restaurants/` | The validated record of a restaurant, along with its categories and associated dishes, persists in PostgreSQL. | 
| `GET` | `/restaurants/` | Allows multi-criteria search and filtering by name, area, category or specific dish (e.g. `?q=hamburguesa&zona=chacao`). | 
| `GET` | `/restaurants/{id}` | Returns the complete listing of the establishment, its categorized menu, and quick links (`wa.me`, Google Maps, Instagram). | 

## Local Installation and Configuration

Follow these steps to set up the development environment on your local machine:

### Prerequisites

* Python 3.11 or higher

* PostgreSQL 15+

* Redis 7+

* A Google Gemini API key (or a local instance of Ollama running a model with vision/JSON support)

### 1. Clone the repository

```
git clone https://github.com/tu-usuario/app-comida.git
food app cd

```

### 2. Create and activate the virtual environment

```
python -m venv venv
source venv/bin/activate  # En Windows: venv\Scripts\activate

```

### 3. Install dependencies

```
pip install -r app/requirements.txt

```

### 4. Configure environment variables

Create a file `.env` at the root of the project, based on the following template:

```
# Application
APP_ENV=development
API_KEY_SECRET=your_secret_key_for_ing

# Base de Datos PostgreSQL (Async)
DATABASE_URL=postgresql+asyncpg://usuario:password@localhost:5432/app_comida

# Cache and Rate Limiting
REDIS_URL=redis://localhost:6379/0

# Artificial Intelligence Provider
AI_PROVIDER=gemini # Options: gemini | ollama
GEMINI_API_KEY=your_google_gemini_api_key
OLLAMA_BASE_URL=http://localhost:11434

```

### 5. Run the development server

```
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

```

Once started, you can access the complete interactive documentation at:
**`http://localhost:8000/docs`**

## Use of the Platform

### Example: Processing a menu with AI (`POST /restaurants/parse-ai`)

You can test the extraction pipeline by sending a request. `POST` with an external URL:

```
curl -X POST "http://localhost:8000/restaurants/parse-ai" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your_secret_key_for_ing" \
  -d '{
    "source_type": "URL_BEACONS",
    "content": "https://beacons.ai/ejemplo_restaurante"
  }'

```

### Example: Search for dishes by area (`GET /restaurants/`)

```
curl -X GET "http://localhost:8000/restaurants/?q=sushi&zona=las_mercedes"

```

## Infrastructure, Security and Performance

For production deployment, the system incorporates dedicated layers of optimization and protection:

* **Cloudflare (Edge Layer):** Provides low-latency DNS resolution, automatic SSL/TLS certificates, DDoS attack mitigation, WAF (Web Application Firewall), and edge caching for static frontend resources.

* **Nginx (Reverse Proxy):** Manages incoming traffic using `proxy_pass` towards the ASGI workers of **Uvicorn**, applying Gzip/Brotli compression and extended timeouts specifically on the AI ​​inference paths.

* **Redis (Optimization and Protection):**

  * **Query Cache:** Stores answers to frequently asked searches by area or category, minimizing repetitive queries to PostgreSQL.

  * **Rate Limiting:** Limits the request rate on AI ingestion endpoints (`/parse-ai`) to prevent abuse and control token consumption.

## Roadmap

* [x] **Phase 1: Architecture Design and Data Modeling** (Modular Monolith, ERD and ingestion flows).

* \[ \] **Phase 2: Core Backend & Pipeline IA (MVP)**

  * \[ \] Implementation of SQLAlchemy models and migrations with Alembic.

  * \[ \] Integration of `POST /restaurants/parse-ai` with Gemini API and Pydantic validation.

  * \[ \] Implementing cache and rate limiting with Redis.

* \[ \] **Phase 3: Web Interface / PWA (Personal Use)**

  * \[ \] Fast mobile-first search engine with Tailwind CSS.

  * \[ \] Quick review panel to approve/edit menus extracted by AI.

* **Phase 4: Native Mobile Application & Geolocation**

  * \[ \] Development in React Native (Expo) for iOS and Android.

  * \[ \] Activating the **PostGIS** extension in PostgreSQL for distance radius searches.

## Contribution and License

Contributions are welcome to improve source extractors, optimize normalization prompts, or expand the client interface:

1. Fork the repository.

2. Create a branch for your functionality (`git checkout -b feature/nueva-funcionalidad`).

3. Make your changes and commit (`git commit -m 'Agrega soporte para extracción de PDFs multipágina'`).

4. Push the changes to your branch (`git push origin feature/nueva-funcionalidad`).

5. Open a *Pull Request* detailing the improvements made.

This project is distributed under the **MIT** license. See the archive `LICENSE` for more details.
