```markdown
# ASX Company Data Scraper

This document outlines the Python code used to scrape financial data from the ASX (Australian Securities Exchange) website. The process involves fetching company information, extracting key statistics, and organizing the data into a structured format.

## 1. Importing Libraries

First, we import all the necessary libraries for web scraping, data manipulation, and parallel processing.

```python
import requests
from bs4 import BeautifulSoup
import re
import json
import pandas as pd
import time
import random
from tqdm import tqdm
from concurrent.futures import ThreadPoolExecutor
tqdm.pandas()
```

## 2. `get_response` Function

This function is designed to make HTTP GET requests to a given URL. It includes a comprehensive set of headers to mimic a real browser, which can help in bypassing basic anti-scraping measures. It returns the response object and the URL requested.

```python
def get_response(url):
    headers = {
        'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
        'accept-language': 'en-US,en;q=0.9',
        'if-modified-since': 'Wed, 11 Jun 2025 07:09:49 GMT',
        'priority': 'u=0, i',
        'sec-ch-ua': '"Google Chrome";v="137", "Chromium";v="137", "Not/A)Brand";v="24"',
        'sec-ch-ua-mobile': '?0',
        'sec-ch-ua-platform': '"Windows"',
        'sec-fetch-dest': 'document',
        'sec-fetch-mode': 'navigate',
        'sec-fetch-site': 'none',
        'sec-fetch-user': '?1',
        'upgrade-insecure-requests': '1',
        'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36',
        # 'cookie': 'affinity="595e506924014dfa"; visid_incap_2835827=/qoX0VrOTxeFGByMdUvnhJguSWgAAAAAQUIPAAAAAAD6MU/Tkj0dy1DF8z8kKHRT; _gcl_au=1.1.1568334597.1749626523; _ga=GA1.1.956764175.1749626524; OptanonAlertBoxClosed=2025-06-11T07:22:07.553Z; _hjSessionUser_3043058=eyJpZCI6IjUxYjUxOTJmLTAwNmUtNWIxZC05MDg5LWEwZDlmMGNkMGVlZSIsImNyZWF0ZWQiOjE3NDk2MjY1Mjc4ODcsImV4aXN0aW5nIjp0cnVlfQ==; ROUTEID=.node1; nlbi_2835827_2708396=tIveJ7LdA2+591p52S5TNgAAAABUs8YLT8nrFyroRb39cths; _hjMinimizedPolls=866717; incap_ses_1563_2835827=M+lIWfIvjG3WoAlKh+SwFb1TSWgAAAAAyeD0bXNrYRq7HWV9DXtGSw==; doasite=ef901639-aa36-4e34-bdcc-acd08d903180; _hjSession_3043058=eyJpZCI6ImFmZDdkYTNmLWY5MmEtNDdjZS05ODBkLTY5MmY4OTI0ZDlmZSIsImMiOjE3NDk2MzYwMzMzNzQsInMiOjEsInIiOjAsInNiIjowLCJzciI6MCwic2UiOjAsImZzIjowLCJzcCI6MX0=; TS019c39fc=01856a822aac9d679bdb515a8620ec691370da62abe7c87d58f543f061ce2854a957c79401dde7deb332af0fd83c1d54cd637835d2; OptanonConsent=isIABGlobal=false&datestamp=Wed+Jun+11+2025+15%3A48%3A32+GMT%2B0530+(India+Standard+Time)&version=6.13.0&hosts=&consentId=5321c249-d4f2-49b1-a128-0a2e30b90c31&interactionCount=1&landingPath=NotLandingPage&groups=C0001%3A1%2CC0002%3A1%2CC0003%3A1%2CC0004%3A1&geolocation=IN%3BTN&AwaitingReconsent=false; _ga_J1L799T374=GS2.1.s1749636034$o2$g1$t1749637123$j47$l0$h0; _ga_PMPRR01441=GS2.1.s1749636033$o2$g1$t1749637123$j31$l0$h0; nlbi_2835827=KBZVNsTa83BZnl9H2S5TNgAAAAA53CCX4ZWr4duCmYk+3TjF',
    }
    response = requests.get(url, headers=headers)
    return response, url
```

### Example Usage of `get_response`

Here's an example of how to use the `get_response` function to fetch data for a specific ASX company.

```python
get_response('https://www.asx.com.au/markets/company/14d')
```

## 3. Loading Input Data

The script starts by loading a CSV file containing a list of ASX-listed companies. This file is expected to contain a column named `inner_page_url` which holds the direct URL to each company's individual page.

```python
input_df = pd.read_csv('/content/drive/MyDrive/Company Project/ASX_Financial/data/ASX_Listed_Companies_11-06-2025_08-16-11_AEST.csv')
```

### Displaying Input DataFrame

The loaded DataFrame is then displayed.

```python
input_df
```

## 4. Parallel Fetching of Company Pages

To speed up the process, the `get_response` function is executed in parallel for all company URLs in the `input_df`. A `ThreadPoolExecutor` is used with `max_workers=10` to handle multiple requests concurrently. `tqdm` is used to display a progress bar.

```python
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(tqdm(executor.map(get_response, input_df['inner_page_url']), total=len(input_df)))
```

### Inspecting First Result

The first result from the parallel fetching is shown for inspection. This will be a tuple containing the `requests.Response` object and the URL.

```python
results[0]
```

## 5. `details_extract` Function

This function fetches key statistics for a given company ticker symbol (`tic`). It makes a request to a specific ASX API endpoint and parses the JSON response. It also adds the ticker symbol to the returned data for easier identification. Error handling is included to return a dictionary with just the ticker if the request fails.

```python
def details_extract(tic):
    res = requests.get(f'https://asx.api.markitdigital.com/asx-research/1.0/companies/{tic}/key-statistics')
    try:
      data = res.json()
      data['data']['tic'] = tic
      return data
    except:return {'data':{'tic':tic}}
```

### Example Usage of `details_extract`

An example call to `details_extract` with a sample ticker.

```python
details_extract('1f4D')
```

## 6. Creating Output DataFrame

The results from the initial page fetching are converted into a Pandas DataFrame.

```python
out_df = pd.DataFrame(results)
```

## 7. Parallel Extraction of Company Details

Now, the `details_extract` function is applied in parallel to extract key statistics for each company. The ticker symbols are extracted from the URLs stored in the `out_df`.

```python
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(tqdm(executor.map(details_extract, [i.split('/')[-1] for i in out_df[1]]), total=len(out_df)))
```

## 8. Normalizing JSON Data

The `results` list, which contains dictionaries from the `details_extract` function, is normalized into a new DataFrame called `harvest_out`. This flattens nested JSON structures.

```python
harvest_out = pd.json_normalize(results)
```

### Displaying Columns of `harvest_out`

The columns of the newly created `harvest_out` DataFrame are displayed.

```python
harvest_out.columns
```

## 9. Saving Data (Commented Out)

The following line is commented out, but it shows how to save the extracted company details to a CSV file.

```python
# harvest_out.to_csv('/content/drive/MyDrive/Company Project/ASX_Financial/data/company_details.csv', index=False)
```

## 10. Displaying Intermediate DataFrame

The `out_df` DataFrame (containing the response objects and URLs from the first fetching step) is displayed again.

```python
out_df
```

```

```