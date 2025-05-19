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

- Query friends who checked in within 50km of your most-visited location
- Identify top10 users whose check-in trajectories highly overlap with yours
- Detect top10 friends who frequently visit locations after you (within 30 days)

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
- **How to Load Data**: 
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

```bash
 vi my-app/src/lib/db.js
```
change it to your username and password
```bash
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'gowalla_project',
  user: 'postgres',
  password: 'mypassword123',
  max: 20,
  idleTimeoutMillis: 30000, 
  connectionTimeoutMillis: 2000,
});

```
---

## 4. Code Structure
### Frontend
- Location of frontend code: `src/app/` and `src/components/`
- Key frontend components:
  - `src/components/VisitMap.js`: Visualizes most visited locations
  - `src/components/FriendCheckinMap.js`: Shows nearby friend check-ins
  - `src/components/OverlappingUsersMap.js`: Displays overlapping trajectories

### Backend
- Location of backend code: `src/app/api/queries/`
- Key API endpoints:
  ```

  GET /api/queries/nearby-friend-checkins
  - Returns friend check-ins within 50km of user's locations
  - Parameters: user_id

  GET /api/queries/following-friends
  - Returns top 10 friends who follow user's check-in patterns
  - Parameters: user_id

  GET /api/queries/overlapping-users
  - Returns top 10 users with similar trajectories
  - Parameters: user_id
  ```

### Database Connection
- Location: `src/lib/db.js`
- Connection configuration:
  ```javascript
  const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'gowalla_project',
    user: 'postgres',
    password: 'mypassword123',
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  });
  ```

## 5. Task Queries
### Task 1: Nearby Friend Check-ins (50km Range)
**Goal**: Find friends who checked in within 50km of your most visited locations in 2010.

**Task Description**:
- Input: A user ID
- Output: List of friend check-ins within 50km radius
- Time Range: Year 2010
- Spatial Constraint: 50km radius
- Social Dimension: Only friends' check-ins

**SQL Implementation**:
```sql
WITH user_locations AS (
  SELECT location_id, location
  FROM checkins
  WHERE user_id = $1
    AND checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
)
SELECT 
  f.friend_id,
  friend.checkin_time,
  friend.location_id,
  ST_Y(friend.location::geometry) as lat,
  ST_X(friend.location::geometry) as lng
FROM friendships f
JOIN user_locations ul
JOIN checkins friend ON f.friend_id = friend.user_id
WHERE f.user_id = $1
  AND friend.checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
  AND ST_DWithin(ul.location, friend.location, 50000)
ORDER BY friend.checkin_time;
```

### Task 2: Who Follows My Steps? (30-Day Window)
**Goal**: Identify friends who tend to visit the same locations within 30 days after you.

**Task Description**:
- Input: A user ID
- Output: Top 10 friends ranked by follow count
- Time Window: Within 30 days after user's check-in
- Social Dimension: Only friends' check-ins
- Ranking: Based on frequency of following behavior

**SQL Implementation**:
```sql
SELECT 
  f.friend_id,
  COUNT(*) AS follow_count,
  json_agg(json_build_object(
    'your_checkin', json_build_object(
      'time', you.checkin_time,
      'location_id', you.location_id,
      'lat', ST_Y(you.location::geometry),
      'lng', ST_X(you.location::geometry)
    ),
    'friend_checkin', json_build_object(
      'time', friend.checkin_time,
      'location_id', friend.location_id,
      'lat', ST_Y(friend.location::geometry),
      'lng', ST_X(friend.location::geometry)
    ),
    'days_difference', EXTRACT(DAY FROM (friend.checkin_time - you.checkin_time))
  )) AS follow_details
FROM friendships f
JOIN checkins you ON f.user_id = you.user_id
JOIN checkins friend ON f.friend_id = friend.user_id
WHERE you.user_id = $1
  AND you.checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
  AND friend.checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
  AND you.location_id = friend.location_id
  AND friend.checkin_time > you.checkin_time
  AND friend.checkin_time <= you.checkin_time + INTERVAL '30 days'
GROUP BY f.friend_id
ORDER BY follow_count DESC
LIMIT 10;
```

### Task 3: Similar Movement Patterns
**Goal**: Find users who share similar movement patterns with you.

**Task Description**:
- Input: A user ID
- Output: Top 10 users with most overlapping check-in locations
- Time Range: Year 2010
- Ranking: Based on number of shared locations
- Visualization: Trajectories with color coding

**SQL Implementation**:
```sql
WITH user_checkins AS (
  SELECT DISTINCT location_id
  FROM checkins
  WHERE user_id = $1
    AND checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
)
SELECT 
  c.user_id,
  COUNT(DISTINCT c.location_id) as overlap_count,
  json_agg(DISTINCT jsonb_build_object(
    'location_id', c.location_id,
    'lat', ST_Y(c.location::geometry),
    'lng', ST_X(c.location::geometry),
    'checkin_time', c.checkin_time
  )) as trajectories
FROM checkins c
JOIN user_checkins uc ON c.location_id = uc.location_id
WHERE c.user_id != $1
  AND c.checkin_time BETWEEN '2010-01-01' AND '2010-12-31'
GROUP BY c.user_id
ORDER BY overlap_count DESC
LIMIT 10;
```

Each task combines multiple dimensions:
- Spatial: Geographic coordinates and distances
- Temporal: Time windows and sequences
- Social: Friendship networks and user interactions

The queries utilize PostGIS spatial functions and PostgreSQL's advanced features to efficiently process and analyze the check-in data.

## 6. How to Run the Application
```bash
# Install dependencies
npm install

# Start the development server
npm run dev
```

## 7. How to Use the Application
1. Visit [http://localhost:3000](http://localhost:3000) to access the main page
2. You'll see three main analysis functions:

### Who Follows My Steps?
- Click on "Who Follows My Steps?" card
- Enter your User ID in the input field
- The system will display a table showing your top 10 friends who tend to visit the same locations within 30 days after you
- Results include friend IDs and the number of times they followed your check-in patterns

### Nearby Check-ins
- Click on "Nearby Check-ins" card
- Enter your User ID in the input field
- View a map showing check-ins from your friends within 50km of your most visited locations
- Different colors represent different friends' check-ins
- Click on markers to see detailed check-in information

### Similar Trajectories
- Click on "Similar Trajectories" card
- Enter your User ID in the input field
- Explore a map showing trajectories of users who share similar movement patterns with you
- Each user's trajectory is shown in a different color
- The legend shows overlap counts for each user

## 8. Port Usage
- Application Port: 3000 (Next.js integrated frontend and backend)

## 9. UI Address
- Main Application: [http://localhost:3000](http://localhost:3000)
- API Endpoints: 
  - [http://localhost:3000/api/queries/nearby-friend-checkins](http://localhost:3000/api/queries/nearby-friend-checkins)
  - [http://localhost:3000/api/queries/following-friends](http://localhost:3000/api/queries/following-friends)
  - [http://localhost:3000/api/queries/overlapping-users](http://localhost:3000/api/queries/overlapping-users)

## 10. Additional Notes
### Assumptions
- All timestamps are in UTC
- Check-in coordinates are valid and within reasonable bounds
- User IDs exist in the database
- Friendship relationships are bidirectional

### External Resources
- [Leaflet](https://leafletjs.com/) for map visualization
- [PostGIS](https://postgis.net/) for spatial queries
- [TailwindCSS](https://tailwindcss.com/) for styling
- [Next.js](https://nextjs.org/) for the full-stack framework

### Data Source
- Gowalla check-in dataset
---
