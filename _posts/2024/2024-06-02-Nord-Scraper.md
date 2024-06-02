---
title: Nord Scraping
description: Python script that downloads all tcp ovpn files from nordvpn.com 
author: BW
date: 2024-06-02 10:00:00 - 0500
categories: [CyberSecurity, Privacy]
tags: [randon]     # TAG names should always be lowercase
image:
  path: /assets/img/posts/2024-06-02-Nord-Scraper/nordscraping.webp
  lqip: /assets/img/posts/2024-06-02-Nord-Scraper/nordscraping.webp
---

Proxies are great, but they tend to have poor reputations. 
For example, they tend to trigger cloudflares captcha page which can be annoying.
This script downloads all the ovpn file from nordvpn's website to be used for whatever you want, very handy for scripting. You'll have access to over 6000 different servers (and different public IPs) across many different countries.

```python
import requests
from bs4 import BeautifulSoup
import os

# URL of the NordVPN ovpn files page
url = 'https://nordvpn.com/nl/ovpn/'

# Directory to save the downloaded files
save_dir = 'nord_ovpn_tcp'
if not os.path.exists(save_dir):
    os.makedirs(save_dir)

# Function to download a file
def download_file(link, filename):
    response = requests.get(link)
    with open(filename, 'wb') as file:
        file.write(response.content)

# Request the page
response = requests.get(url)
soup = BeautifulSoup(response.text, 'html.parser')

# Find all TCP links
tcp_links = soup.find_all('a', href=True)
for link in tcp_links:
    href = link['href']
    if 'tcp' in href and href.endswith('.ovpn'):
        file_name = os.path.join(save_dir, href.split('/')[-1])
        print(f'Downloading {href} to {file_name}')
        download_file(href, file_name)

print('Download completed.')
```
{: file='nordscraper.py'}

> You will need an active NordVPN subscription to use openvpn
{: .prompt-warning }
