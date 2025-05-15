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

```bash
cd gis/my-app
# Install frontend dependencies
npm install


### `npm run build` fails to minify

This section has moved here: [https://facebook.github.io/create-react-app/docs/troubleshooting#npm-run-build-fails-to-minify](https://facebook.github.io/create-react-app/docs/troubleshooting#npm-run-build-fails-to-minify)
