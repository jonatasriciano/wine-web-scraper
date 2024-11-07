
* * *

Wine Web Scraping API Documentation
===================================

Project Overview
----------------

This API was developed to automate the collection of wine data from various online retailers across Canada. Each bot (scraper script) is responsible for accessing a specific website and extracting details about available wines, such as name, price, type, vintage, region, and more. This solution is part of an RPA (Robotic Process Automation) project, where each bot functions independently to gather information from different sites.

Project Structure
-----------------

The project follows a modular design, making it easier to maintain and update individual bots. Here is a suggested structure:

bash

Copy code

`wine_scraper_project/ ├── bots/                   # Contains site-specific scraping scripts │   ├── site1_scraper.py │   ├── site2_scraper.py │   └── siteN_scraper.py ├── core/                   # Base code and common functions │   ├── __init__.py │   ├── base_scraper.py     # Base class for scraper bots │   └── utils.py            # Utility functions (e.g., HTML parsing, data handling) ├── api/                    # API code │   ├── __init__.py │   ├── main.py             # Main API endpoints │   └── schemas.py          # Data schemas for API endpoints ├── data/                   # Database or CSVs where data is stored └── docs/                   # Documentation and supporting files`

Technologies Used
-----------------

*   **Python**: Main programming language.
*   **FastAPI or Flask**: Framework for building the REST API.
*   **BeautifulSoup and/or Scrapy**: For parsing and extracting data from websites.
*   **SQLite or PostgreSQL**: Database to store the scraped data (or CSV, depending on the setup).
*   **Docker**: Containerizes the application for easy deployment.
*   **Selenium** (optional): For scraping dynamic (JavaScript) websites.

### Environment Requirements

To run the project locally:

1.  **Python 3.8+**
2.  **Libraries**:
    *   Install dependencies via `requirements.txt`:
        
        bash
        
        Copy code
        
        `pip install -r requirements.txt`
        
3.  **Docker** (optional): To isolate the development environment.

Project Dependencies
--------------------

Each tool and library was chosen for ease of development and scalability:

#### Example `requirements.txt`

plaintext

Copy code

`fastapi uvicorn beautifulsoup4 requests sqlalchemy`

Install these dependencies with:

bash

Copy code

`pip install -r requirements.txt`

Base Scraper Class (`base_scraper.py`)
--------------------------------------

The `BaseScraper` class serves as a foundation for all the bots, defining basic methods like `fetch_page`, which retrieves the HTML page content, and `save_data`, which saves the data to a database or file. Each bot will inherit from this class and implement its own method for extracting site-specific information.

python

Copy code

`# core/base_scraper.py import requests from bs4 import BeautifulSoup from abc import ABC, abstractmethod import sqlite3  class BaseScraper(ABC):     def __init__(self, url):         self.url = url         self.session = requests.Session()      def fetch_page(self):         response = self.session.get(self.url)         response.raise_for_status()  # Raises an error if the request fails         return response.text      @abstractmethod     def parse_page(self, html):         """Abstract method to be implemented by subclasses to parse the page"""         pass      def save_data(self, data):         """Saves data to a SQLite database"""         connection = sqlite3.connect('data/database.db')         cursor = connection.cursor()          cursor.executemany(             'INSERT INTO wines (name, price, region) VALUES (?, ?, ?)',             [(item['name'], item['price'], item['region']) for item in data]         )          connection.commit()         connection.close()`

Individual Site Bots
--------------------

Each bot is a class that inherits from `BaseScraper` and implements the `parse_page` method, which varies based on the layout of each site.

python

Copy code

`# bots/site1_scraper.py from core.base_scraper import BaseScraper from bs4 import BeautifulSoup  class Site1Scraper(BaseScraper):     def parse_page(self, html):         soup = BeautifulSoup(html, 'html.parser')         wines = []                  # Example data extraction         for item in soup.select('.wine-item'):             wine = {                 'name': item.select_one('.wine-name').get_text(strip=True),                 'price': item.select_one('.wine-price').get_text(strip=True),                 'region': item.select_one('.wine-region').get_text(strip=True),             }             wines.append(wine)                  self.save_data(wines)`

API Endpoints
-------------

The API exposes endpoints to initiate scraping and retrieve filtered data.

### API Code (`main.py`)

python

Copy code

`# api/main.py from fastapi import FastAPI, HTTPException from bots import site1_scraper, site2_scraper  app = FastAPI()  @app.post("/scrape/{site_id}") async def scrape(site_id: str):     """Starts scraping for the specified site"""     if site_id == "site1":         scraper = site1_scraper.Site1Scraper("https://example.com/wine")         html = scraper.fetch_page()         scraper.parse_page(html)         return {"message": "Scraping for site1 started successfully."}     else:         raise HTTPException(status_code=404, detail="Site not found")  @app.get("/wines/") async def get_wines(region: str = None, max_price: float = None):     """Returns stored wine data with optional filters"""     connection = sqlite3.connect('data/database.db')     cursor = connection.cursor()          query = "SELECT name, price, region FROM wines"     filters = []     params = []      if region:         filters.append("region = ?")         params.append(region)     if max_price:         filters.append("price <= ?")         params.append(max_price)      if filters:         query += " WHERE " + " AND ".join(filters)      cursor.execute(query, params)     wines = cursor.fetchall()     connection.close()      return {"wines": wines}`

### Example Endpoints

1.  **Start Scraping for a Site**
    
    *   **Endpoint**: `/scrape/{site_id}`
    *   **Method**: `POST`
    *   **Description**: Starts scraping for the specified site.
    *   **Parameters**:
        *   `site_id` (path): Site identifier.
    *   **Response**:
        *   `200 OK`: Confirms scraping has started.
        *   `404 Not Found`: If `site_id` is not implemented.
2.  **Retrieve Stored Data**
    
    *   **Endpoint**: `/wines/`
    *   **Method**: `GET`
    *   **Description**: Returns all stored wine data, with optional filters for region and price.
    *   **Parameters**:
        *   `region` (optional, query): Filter by wine region.
        *   `max_price` (optional, query): Filter by maximum price.
    *   **Response**:
        *   `200 OK`: Returns filtered or complete data.

Example Tests
-------------

Use testing to ensure that each bot and API endpoint works as expected:

python

Copy code

`# tests/test_scrapers.py import unittest from bots.site1_scraper import Site1Scraper  class TestSite1Scraper(unittest.TestCase):     def test_fetch_page(self):         scraper = Site1Scraper("https://example.com/wine")         html = scraper.fetch_page()         self.assertIn("<html", html)  # Checks if page was downloaded correctly      def test_parse_page(self):         scraper = Site1Scraper("https://example.com/wine")         html = "<html><div class='wine-item'>...</div></html>"         wines = scraper.parse_page(html)         self.assertGreater(len(wines), 0)  # Checks if data was extracted  if __name__ == "__main__":     unittest.main()`

Docker Setup
------------

For ease of deployment, the project is containerized. The `Dockerfile` defines the environment, and the `docker-compose.yml` can be used to run the app.

### Example Dockerfile

dockerfile

Copy code

`# Dockerfile FROM python:3.9 WORKDIR /app COPY . . RUN pip install -r requirements.txt CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]`

To build and run the container:

bash

Copy code

`docker build -t wine_scraper . docker run -d -p 8000:8000 wine_scraper`

Automation and Scheduling
-------------------------

Each bot can be scheduled to run periodically using **Cron** or a CI/CD tool like **GitHub Actions** or **GitLab CI**, based on the frequency with which each site updates its content.

Final Considerations
--------------------

1.  **Compliance**: Ensure web scraping is permitted under each website's terms of service.
2.  **Rate Limiting**: Respect request limits to avoid IP blocking.
3.  **Layout Updates**: Keep bots updated, as site layouts can change frequently.

* * *
