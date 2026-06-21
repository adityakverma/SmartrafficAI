# SmartTraffic AI — Production System Architecture & Scalability Plan

This document outlines the production-ready architecture designed to scale the **SmartTraffic AI** prototype to city-wide operations (e.g., Bengaluru city traffic management under ASTraM). 

---

## 1. High-Level Architecture Overview

In a real-world deployment, training models and running heavy predictions directly in the client's browser is not viable due to data volumes, security, and multi-user coordination constraints. 

The production architecture decouples the frontend client console from the data ingestion, storage, and AI processing layers.

```
                    ┌──────────────────────────────────────────────┐
                    │               DATA INGESTION                 │
                    │   (MapmyIndia API, GPS Feeds, IoT Sensors)   │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │             KAFKA EVENT STREAM               │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │           REAL-TIME PROCESSING               │
                    │         (Apache Spark / Flink)               │
                    └──────────────────────┬───────────────────────┘
                                           │
                        ┌──────────────────┴──────────────────┐
                        ▼                                     ▼
      ┌───────────────────────────────────┐ ┌───────────────────────────────────┐
      │          STORAGE LAYER            │ │       AI PREDICTION SERVICE       │
      │   PostgreSQL + PostGIS (Geo)     │ │     FastAPI + XGBoost/RF Model    │
      │   Redis (Caching & Live State)    │ │     MLflow (Model Registry)       │
      └─────────────────┬─────────────────┘ └─────────────────┬─────────────────┘
                        │                                     │
                        └──────────────────┬──────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │               API GATEWAY &                  │
                    │             WEBSOCKET SERVER                 │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │            NEXT.JS CLIENT DASHBOARD          │
                    │     (Visual Twin, Dispatcher Console)        │
                    └──────────────────────────────────────────────┘
```

---

## 2. Component Breakdown

### A. Data Ingestion & Streaming Layer
*   **Data Sources:** GPS feeds from active delivery fleets (Flipkart/e-cart), sensor loops, CCTV-based automated vehicle counters, and MapmyIndia Traffic APIs.
*   **Apache Kafka:** Acts as the central event bus to ingest thousands of vehicle coordinate updates and traffic incident feeds per second.
*   **Apache Spark / Flink:** Processes the raw incoming streams, extracts features in real-time, and calculates running traffic densities across major city corridors.

### B. Machine Learning & Prediction Layer
*   **Model Hosting (Python/FastAPI):** Machine learning models (XGBoost, Random Forest, or LSTM for timeseries) are served via Docker containers in a Python environment.
*   **Inference API:** The frontend app calls REST endpoints (e.g., `/api/v1/predict`) for immediate predictions, keeping the model logic secure and scalable.
*   **MLflow & Model Registry:** Manages model versions, tracks training runs, and facilitates automated periodic retraining as new historical data accumulates in the database.

### C. Data & State Storage Layer
*   **PostgreSQL with PostGIS Extension:** A relational database optimized for geographical data. It stores historical traffic incidents, coordinates, junction shapes, and corridor polygons.
*   **Redis Cache:** Caches popular forecast queries (e.g., peak-hour corridors) and maintains the active dispatch state (which officers are deployed where) for real-time sync across multiple dispatch consoles.

### D. Real-Time Application Backend (Node.js / Websockets)
*   **Next.js Server / API Gateway:** Handles authentication, serves the client-side dashboard shell, and acts as the secure interface to the internal AI microservices.
*   **WebSockets (Socket.io):** Pushes live updates to dispatchers. When an incident is registered, it instantly updates the digital twin visualization and map markers for all logged-in police dispatch consoles.

---

## 3. Scalability & Operational Feasibility

| Evaluation Metric | Prototype Implementation | Production Target |
| :--- | :--- | :--- |
| **Model Location** | In-browser (TypeScript WebWorker/Main Thread) | Server-side microservice (Python + FastAPI) |
| **Data Capacity** | Pre-filtered 4.5 MB CSV (~8,000 records) | Terabytes of geographical data in PostgreSQL |
| **Latency** | <5ms execution, but page load needs ~1s training | <50ms API request (using cached model weights) |
| **Collaboration** | Local-only (State lost on refresh) | Centralized DB with WebSockets for multi-user sync |
| **Hardware Reqs** | Client CPU (Laptop/Mobile Browser) | Scalable Kubernetes Node Pool (CPU / GPU) |

---

## 4. Why This Architecture Wins Hackathons
1.  **Saves Compute Overhead:** Edge inference in the browser is demonstrated (via this Next.js app) for lightweight local use, while our backend API architecture handles enterprise scaling.
2.  **Standards-Compliant:** Uses industry-standard tools (Python FastAPI, PostGIS, Kafka) that match Flipkart's actual infrastructure.
3.  **Real-Time Ready:** Ready to ingest external mapping SDKs (MapmyIndia / Google Maps) to display live routing instead of static SVG placeholders.
