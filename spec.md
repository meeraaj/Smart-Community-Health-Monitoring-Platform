# Smart Community Health Monitoring Platform (SIH 2025)

## Goal
A digital health platform with two parts:
1. **User Reporting System** – users report symptoms/cases.
2. **Authority Dashboard** – map & alerts for health authorities.

---

## Step 1: Project Structure
Use a monorepo with:
```
/project-root
  /frontend   # React app
  /backend    # FastAPI app
  /db         # MongoDB (via Docker)
  docker-compose.yml
```

---

## Step 2: Docker Setup
- Each service (`frontend`, `backend`, `db`) has its own `Dockerfile`.
- Example `docker-compose.yml`:

```yaml
version: "3.9"
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - MONGO_URI=mongodb://db:27017/healthdb
    depends_on:
      - db

  db:
    image: mongo:latest
    container_name: mongo
    ports:
      - "27017:27017"
    volumes:
      - ./db/data:/data/db
```

---

## Step 3: Database Schema
MongoDB collection: `reports`
```json
{
  "id": "string",
  "userId": "string",
  "location": { "lat": 12.34, "lng": 56.78 },
  "description": "string",
  "timestamp": "ISODate",
  "status": "pending|confirmed"
}
```

---

## Step 4: Backend (FastAPI)
FastAPI endpoints:

```python
from fastapi import FastAPI
from pymongo import MongoClient
from datetime import datetime

app = FastAPI()
client = MongoClient("mongodb://db:27017/")
db = client.healthdb

@app.post("/reports")
def create_report(report: dict):
    report["timestamp"] = datetime.utcnow()
    db.reports.insert_one(report)
    return {"message": "Report submitted"}

@app.get("/reports")
def get_reports():
    reports = list(db.reports.find().sort("timestamp", -1).limit(10))
    return reports

@app.get("/hotspots")
def get_hotspots():
    # simple rule-based classifier: count cases by location
    pipeline = [
        {"$group": {"_id": "$location", "count": {"$sum": 1}}},
        {"$match": {"count": {"$gte": 5}}}
    ]
    return list(db.reports.aggregate(pipeline))
```

---

## Step 5: Frontend (React)
- Sidebar navigation with 2 options:
  1. **Map View** → use React-Leaflet or Google Maps API to show hotspots (red markers).
  2. **Alert View** → fetch `/reports` and list latest reports.

Example React fetch:

```js
useEffect(() => {
  fetch("http://localhost:8000/reports")
    .then(res => res.json())
    .then(data => setReports(data));
}, []);
```

---

## Step 6: Rule-Based Classifier
- If **≥ 5 cases in last 7 days** in the same area → mark as hotspot.
- Hotspots returned by `/hotspots`.
- Frontend marks them in **red on the map**.

---

## Step 7: Deployment
- Build containers with:
  ```bash
  docker-compose build
  docker-compose up
  ```
- Push images to Azure VM (or Azure Container Registry).
- Run `docker-compose up -d` in VM.

---

## Notes
- Start small: basic connectivity React ↔ FastAPI ↔ MongoDB.
- Add map + alerts → then classifier logic.
- Keep modular for AI integration later.
