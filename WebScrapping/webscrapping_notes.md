# Web Scraping — Beginner to Advanced (Interview-Ready Notes)
 
---
 
## 1. Fundamentals
 
**What is web scraping?** Programmatically extracting data from websites by fetching HTML and parsing it, instead of manual copy-paste.
 
**Scraping pipeline:**
```
Request → Response (HTML) → Parse →Extract → Clean → Store
```
 
**Static vs Dynamic sites:**
| Type | How content loads | Tool needed |
|---|---|---|
| Static | Full HTML returned by server on first request | `requests` + `BeautifulSoup` |
| Dynamic (JS-rendered) | Content injected by JavaScript after page load | `Selenium` / `Playwright` |
 
**How to tell which one you're dealing with:** View page source (`Ctrl+U`, raw HTML) vs Inspect (`F12`, live DOM). If data is in Inspect but missing from "view source" → JS-rendered.
 
---
 
## 2. Core Libraries
 
| Library | Purpose |
|---|---|
| `requests` | Send HTTP requests, get raw HTML |
| `BeautifulSoup` (bs4) | Parse HTML, navigate/search DOM tree |
| `lxml` | Faster parser backend, used under bs4 or with XPath |
| `Selenium` | Automates a real browser — handles JS rendering, clicks, scrolling |
| `Playwright` | Modern alternative to Selenium; faster, better async support |
| `Scrapy` | Full scraping *framework* — async, built-in pipelines, retries, scheduling |
| `pandas` | Store/clean/export scraped data (CSV, Excel) |
 
---
 
## 3. Basic Requests + BeautifulSoup Workflow
 
```python
import requests
from bs4 import BeautifulSoup
 
headers = {'User-Agent': 'Mozilla/5.0'}  # avoid being blocked as a bot
response = requests.get(url, headers=headers)
 
soup = BeautifulSoup(response.text, 'html.parser')  # or 'lxml' (faster)
 
cards = soup.find_all('div', class_='card')
for card in cards:
    title = card.find('h2')
    print(title.text.strip() if title else None)
```
 
**`response.status_code` cheat sheet:**
- `200` OK
- `403` Forbidden (often bot-blocked — fix headers/User-Agent)
- `404` Not found
- `429` Too many requests (rate-limited)
- `503` Server unavailable
 
---
 
## 4. BeautifulSoup Selection Methods
 
| Method | Returns | Notes |
|---|---|---|
| `find(tag, attrs)` | First match / `None` | Use for single elements |
| `find_all(tag, attrs)` | List of all matches | Most common for card/row loops |
| `select_one(css)` | First match / `None` | CSS selector syntax |
| `select(css)` | List of matches | Best for multi-class / nested selectors |
| `.get('attr')` | Attribute value | e.g. `tag.get('href')` for links |
| `.text` / `.get_text()` | Inner text | `.get_text(strip=True)` avoids manual `.strip()` |
 
**The `class_` multi-value trap:**
```python
# WRONG — treats string as ONE literal class name (won't match)
soup.find('div', class_='rating_text rating_text--md')
 
# RIGHT — single class matches fine
soup.find('div', class_='rating_text')
 
# RIGHT — CSS selector for compound classes
soup.select_one('.rating_text.rating_text--md')
```
 
**Navigating the tree:**
```python
tag.parent          # parent element
tag.find_next_sibling()
tag.find_previous_sibling()
tag.children        # direct children
tag.descendants      # all nested elements
```
 
---
 
## 5. Defensive Coding (avoid `NoneType` crashes)
 
```python
tag = card.find('span', class_='price')
value = tag.text.strip() if tag else None
```
 
**Why this matters:** real sites have missing fields, ads, inconsistent cards. Never chain `.text` directly on `.find()` — one missing element shouldn't crash the whole loop.
 
```python
try:
    data = card.find('div', class_='x').text.strip()
except AttributeError:
    data = None
```
 
---
 
## 6. Handling Dynamic (JavaScript-rendered) Content
 
**Option A — Selenium (simulate a real browser):**
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
 
driver = webdriver.Chrome()
driver.get(url)
 
WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CLASS_NAME, 'card'))
)
 
elements = driver.find_elements(By.CLASS_NAME, 'card')
```
 
**Option B — Find the underlying API call (preferred when possible):**
- Open DevTools → Network tab → filter `XHR/Fetch`.
- Reload page, look for a request returning JSON with your data.
- Hit that endpoint directly with `requests` — much faster and more stable than browser automation.
 
**Rule of thumb:** always check the Network tab first before reaching for Selenium. A hidden JSON API is faster, lighter, and less fragile than driving a browser.
 
---
 
## 7. Handling Pagination
 
```python
for page in range(1, 11):
    url = f"https://example.com/companies?page={page}"
    response = requests.get(url, headers=headers)
    # parse and extract...
```
- Infinite scroll → simulate scroll with Selenium (`driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")`) or find the paginated API call it triggers.
 
---
 
## 8. Being a Good/Undetected Scraper
 
| Technique | Why |
|---|---|
| Set `User-Agent` header | Avoid default `python-requests` UA getting blocked |
| `time.sleep()` / random delays between requests | Avoid rate-limiting, look human |
| Rotate proxies/IPs | Avoid IP bans on large-scale scraping |
| Respect `robots.txt` | Ethical/legal — check `site.com/robots.txt` |
| Use sessions (`requests.Session()`) | Persist cookies/headers across requests |
| Handle retries with backoff | Networks fail; retry with exponential delay |
 
```python
import time, random
time.sleep(random.uniform(1, 3))
```
 
---
 
## 9. Scrapy (framework — advanced)
 
**Why Scrapy over requests+bs4:** built-in async requests (fast), automatic retry/backoff, pipelines for cleaning/storing data, middleware for proxies/headers, easy scheduling of large crawls.
 
```python
import scrapy
 
class CompanySpider(scrapy.Spider):
    name = 'companies'
    start_urls = ['https://example.com/companies']
 
    def parse(self, response):
        for card in response.css('div.card'):
            yield {
                'name': card.css('h2::text').get(),
                'rating': card.css('.rating::text').get(),
            }
        next_page = response.css('a.next::attr(href)').get()
        if next_page:
            yield response.follow(next_page, self.parse)
```
 
**Scrapy uses CSS/XPath selectors natively** — no BeautifulSoup needed inside it.
 
---
 
## 10. XPath Basics (used in Scrapy/Selenium)
 
| XPath | Meaning |
|---|---|
| `//div` | all `div` tags anywhere |
| `//div[@class="card"]` | div with exact class |
| `//div[contains(@class, "card")]` | div where class *contains* "card" |
| `//h2/text()` | text inside h2 |
| `//a/@href` | href attribute of a link |
 
---
 
## 11. Storing Scraped Data
 
```python
import pandas as pd
df = pd.DataFrame({'name': name_list, 'rating': rating_list})
df.to_csv('companies.csv', index=False)
```
- CSV for tabular data, JSON for nested/hierarchical data, SQLite/Postgres for large-scale persistent storage.
 
---
 
## 12. Common Errors & Fixes
 
| Error | Cause | Fix |
|---|---|---|
| `AttributeError: NoneType has no attribute 'text'` | `.find()` returned `None` | Check `if tag:` before accessing `.text` |
| `403 Forbidden` | Site detects bot | Add proper `User-Agent`, headers, session cookies |
| Empty results with correct selector | Content is JS-rendered | Use Selenium/Playwright or find the JSON API |
| `ConnectionError` / timeouts | Rate-limited or network issue | Add delays, retries, proxy rotation |
| Data appears in browser but not `requests.get()` | Same as above — client-side rendering | Same fix |
| `SSLError` | Site cert / `verify=True` issue | `requests.get(url, verify=False)` (use cautiously) |
 
---
 
## 13. Interview Q&A Rapid Fire
 
**Q: `requests` + BeautifulSoup vs Selenium — when do you pick which?**
A: `requests`+bs4 for static HTML (fast, lightweight). Selenium/Playwright when content is JS-rendered and no accessible API exists. Prefer finding the underlying API over Selenium when possible — much faster.
 
**Q: How do you scrape at scale (thousands of pages) efficiently?**
A: Use Scrapy (async, built-in concurrency) or `asyncio` + `aiohttp`, with proxy rotation, rate limiting, and retry/backoff logic.
 
**Q: How do you avoid getting blocked?**
A: Rotate User-Agents/proxies, add delays, respect `robots.txt`, use sessions, avoid hammering endpoints, mimic human request patterns.
 
**Q: `find_all` vs `select` in BeautifulSoup?**
A: `find_all` is bs4-native (tag/attrs based). `select`/`select_one` uses CSS selector syntax — better for compound/nested/multi-class selectors.
 
**Q: Is web scraping legal?**
A: Depends on jurisdiction, the site's Terms of Service, and `robots.txt`. Scraping publicly available data is generally lower-risk than bypassing logins/paywalls or scraping personal data — always check ToS; this isn't legal advice.
 
**Q: How do you handle a site that changes its HTML structure often?**
A: Write defensive extraction (fallback to `None`, log missing fields), use stable attributes (e.g. `data-testid`) over volatile class names when available, and monitor/alert on scrape failures.
 
**Q: `lxml` vs `html.parser`?**
A: `lxml` is faster and more lenient with malformed HTML; `html.parser` is built into Python (no extra install) but slower. `html5lib` is most lenient/slowest, used for very broken HTML.
 
---
 
## 14. Quick Glossary
 
- **DOM** — Document Object Model, the tree structure of an HTML page.
- **Selector** — a pattern (CSS or XPath) used to locate elements.
- **Headless browser** — a browser with no visible UI, used for automation (Selenium/Playwright can run headless).
- **Rate limiting** — server restricting how many requests you can make in a time window.
- **User-Agent** — HTTP header identifying the client (browser/bot) making the request.
- **robots.txt** — file at a site's root declaring which paths bots are allowed/disallowed to crawl.