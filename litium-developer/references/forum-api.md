# Discourse Forum API (forum.litium.com)

## Overview

Public Discourse API for searching and reading forum posts. No authentication required for read operations.

## Base URL

```
https://forum.litium.com/
```

## Endpoints

### Search

```bash
GET /search.json?q=<query>&page=<page>
```

**Python Example:**
```python
import requests

def search_forum(query: str, page: int = 0):
    url = "https://forum.litium.com/search.json"
    params = {"q": query, "page": page}
    r = requests.get(url, params=params)
    return r.json()

# Usage
results = search_forum("elasticsearch error")
```

### Topic Details

```bash
GET /t/<topic-id>.json
```

**Python Example:**
```python
def get_topic(topic_id: int):
    url = f"https://forum.litium.com/t/{topic_id}.json"
    r = requests.get(url)
    return r.json()
```

### Latest Posts

```bash
GET /latest.json
```

### Tags

```bash
GET /tags.json
GET /tag/<tag-name>.json
```

## Search Query Syntax

```
# Basic
q=api

# Tag search
q=litium-8

# Username
q=@username

# Category
q=#category

# Date range
q=after:2023-01-01

# Combined
q=elasticsearch @username #category after:2023-01-01 order:latest
```

## Rate Limits

- Default: ~60 req/min for anonymous users
- Implement exponential backoff
- Cache results when possible

## Python Implementation

```python
import requests
import time
from typing import List, Dict

class LitiumForum:
    BASE_URL = "https://forum.litium.com"

    def __init__(self, delay: float = 1.0):
        self.delay = delay
        self.last_request = 0

    def _rate_limit(self):
        now = time.time()
        elapsed = now - self.last_request
        if elapsed < self.delay:
            time.sleep(self.delay - elapsed)
        self.last_request = time.time()

    def search(self, query: str, max_results: int = 20) -> List[Dict]:
        """Search forum with rate limiting"""
        self._rate_limit()
        results = []
        page = 0

        while len(results) < max_results:
            r = requests.get(
                f"{self.BASE_URL}/search.json",
                params={"q": query, "page": page}
            )
            r.raise_for_status()
            data = r.json()

            if "topics" not in data or not data["topics"]:
                break

            results.extend(data["topics"])
            page += 1

        return results[:max_results]

    def get_topic(self, topic_id: int) -> Dict:
        """Get full topic details"""
        self._rate_limit()
        r = requests.get(f"{self.BASE_URL}/t/{topic_id}.json")
        r.raise_for_status()
        return r.json()

    def get_by_tag(self, tag: str) -> List[Dict]:
        """Get topics by tag"""
        self._rate_limit()
        r = requests.get(f"{self.BASE_URL}/tag/{tag}.json")
        r.raise_for_status()
        return r.json().get("topic_list", {}).get("topics", [])
```

## Usage from Skill

```python
# From skill script — adjust the import path to your skill location
from forum_api import LitiumForum

forum = LitiumForum()
results = forum.search("accelerator setup issue")
```
