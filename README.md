# Gowalla High-Dimensional Query Web App

## 1. Project Overview

### Motivation

This web application explores meaningful patterns from the Gowalla dataset, which contains millions of user check-ins and social connections. The goal is to enable users to analyze their mobility behavior, discover overlapping trajectories, and uncover influence-based social interactions via spatial-temporal queries.

Real-world use cases include:

- Location recommendation  
- Social movement analysis  
- Influence modeling  
- Interactive geospatial visualization

### Web Application Functions

- Query friends who checked in within 200km of your most-visited location
- Identify users whose check-in trajectories highly overlap with yours
- Detect friends who frequently visit locations after you (within 30 days)

Each function leverages high-dimensional queries that combine:

- Spatial (geographic coordinates)
- Temporal (timestamp)
- Social (friendship network) dimensions

---

## 2. Technology Stack

### Programming Languages & Frameworks

- **Backend:** Node.js (API routes inside Next.js)  
- **Frontend:** Next.js (React) + Leaflet  
- **Database:** PostgreSQL 15 + PostGIS extension  

### Packages & Dependencies

| Package         | Purpose                                      |
|----------------|----------------------------------------------|
| `pg`           | Connect backend to PostgreSQL                |
| `react-leaflet`| Render interactive maps                      |
| `leaflet`      | Core geospatial map rendering library        |
| `tailwindcss`  | Utility-first CSS framework for styling      |
| `postgis`      | PostgreSQL extension for spatial queries     |

---

## 3. Setup Instructions
### Environment Setup
First, install dependencies.
```bash
cd gis/my-app

# Install required dependencies
npm install  # For next.js projects
```

### Database Configuration
- **Database Schema**: Provide an overview of the database structure.
  ```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE checkins_raw (
    user_id BIGINT,
    checkin_time TIMESTAMPTZ,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    location_id BIGINT
);

CREATE TABLE checkins (
    id SERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    checkin_time TIMESTAMPTZ NOT NULL,
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    location_id BIGINT NOT NULL
);

CREATE INDEX idx_checkins_location ON checkins USING GIST(location);
CREATE INDEX idx_checkins_user_time ON checkins(user_id, checkin_time);

CREATE TABLE friendships (
    user_id BIGINT,
    friend_id BIGINT,
    PRIMARY KEY (user_id, friend_id)
);

  ```
  
- **How to Initialize Database**:
  In the gis directory
  ```bash
  createdb gowalla_project
  psql -U postgres -d gowalla_project -c "CREATE EXTENSION IF NOT EXISTS postgis;"
  psql -U postgres -d gowalla_project -f schema.sql

  psql -U postgres -d gowalla_project
  ```
- **How to Load Data**: e.g., from a csv file.
  ```sql

  # Import raw check-in data from CSV into the checkins_raw table
  \copy checkins_raw(user_id, checkin_time, latitude, longitude, location_id) FROM 'checkins.csv' WITH (FORMAT csv);

  # Import friendship edges from CSV into the friendships table
  \copy friendships(user_id, friend_id) FROM 'friendships.csv' WITH (FORMAT csv);

  # Convert raw check-in records to structured check-ins table with geography type for spatial queries
  INSERT INTO checkins (user_id, checkin_time, location, location_id)
    SELECT
      user_id,
      checkin_time,
      ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)::GEOGRAPHY,
      location_id
      FROM checkins_raw;
  ```
### Configurate

---
