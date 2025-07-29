+++
author = "Andreas Lay"
title = "Scraping German Rental Price Data – Part I: Whole Lotta Captchas"
date = "2025-07-28"
description = "Scraping wg-gesucht's Apartment Listings"
tags = ["python", "scraping", "web development"]
categories = ["Python", "Data Engineering", "Web Scraping"]
ShowToc = true
TocOpen = true
+++

## Our Goal

Finding affordable apartments is hard. We may not be able to influence rental prices but at least we can bring some transparency into the market. Our goal: We want to collect apartment listings with corresponding meta information like rental prices, square meters and location. To achieve this we will:

1. Crawl [wg-gesucht.de](https://www.wg-gesucht.de/)
2. Write the results to a `sqlite` database
3. Write a small [Django](https://www.djangoproject.com/) to serve our scraped data as a dashboard

You can find the source code on [Github](https://github.com/layandreas/wg-watch). The final result will looks like this and can be accessed [under this link](https://wg-watch-137914338338.europe-west1.run.app/):

![](/personal-blog/wg-gesucht-watch-screenshot.png)

This part of my series deals with the actual scraping.

## WG Gesucht?

The German abbreviation "WG" (Wohngemeinschaft) translates to "shared flat" and if you're looking for one you can try your luck on [wg-gesucht.de](https://www.wg-gesucht.de/). Despite its name wg-gesucht not only offers shared flats but also offers apartments for rent:

![Minio Bucket](/personal-blog/wg-gesucht-landing.png)

- WG-Zimmer = Shared flat
- 1-Zimmer Wohnung = One-room apartment
- Wohnung = Apartment
- Haus = House

## Let's Inspect the HTML…

The network tab in Chrome DevTools doesn't reveal any API we could directly query, but looking at the HTML of a search result for apartments in Berlin we see that listing data is embedded as [JSON-LD](https://json-ld.org/):

![](/personal-blog/wg-gesucht-listing.png)

Interestingly the square meter information is not part of this JSON, but only part of a `div`:

![](/personal-blog/wg-gesucht-listing-sqm.png)

This `div` has the id `liste-details-ad-<LISTING-ID>` and the listing id is also part of the URL field in the above JSON, so we can easily link these two parts.

Last but not least we wil need to loop over the result pages. At the bottom of the page we find a `div` with id `assets-list-pagination` which gives us information of how many pages of results we can scrape:

![](/personal-blog/wg-gesucht-listing-pagination.png)

That's enough for us now, we can work with that.

## …And the URL Too

We can parameterize the search result URL with these parameters:

- city
- city_id
- page

In my code I simply use a templated string:

```python
URL_TEMPLATE = (
    "https://www.wg-gesucht.de/wg-zimmer-und-1-zimmer-wohnungen-und-wohnungen-und-haeuser"
    "-in-{city}.{city_id}.0+1+2+3.1.{page}.html?offer_filter=1"
    "&city_id={city_id}&sort_order=0&noDeact=1&categories%5B%5D=0"
    "&categories%5B%5D=1&categories%5B%5D=2&categories%5B%5D=3&pagination=4&pu="
)
```

We are going to query all listings without further filters.

## Let's Scrape

The [scraper](https://github.com/layandreas/wg-watch/blob/main/scraper/main.py) in itself is quite simple. I am using [Zendriver](https://github.com/cdpdriver/zendriver) for scraping via Chrome and parse the scraped HTML with [Beautiful Soup](https://pypi.org/project/beautifulsoup4/).

Running the scraper will:

- open multiple Chrome instances and scrape the site for you
- load & parse the HTML
- write the extracted data into a local SQLite database.

This is how it looks:

{{< youtube btMRIabYSUA >}}

After a while you will run into Captchas:

![](/personal-blog/wg-gesucht-captcha.png)

My script currently detectes when a Captcha page pops up and stops scraping while waiting for the user to manually solve the captcha. After that scraping for the corresponding page will continue. It might be worth checking out [SeleniumBase](https://github.com/seleniumbase/SeleniumBase) which might be able to bypass Captchas.

## Result: A Database of Apartment Listings

After our scraper has finished its job we will see that our table in SQLite has filled up with listing data:

![](/personal-blog/wg-gesucht-db.png)

## Part II: Building a Dashboard

That's it for this part! Now that we've got our data collected we'd like get some insights into rental prices. Therefore in the upcoming part II I will show you how I've built a small UI using [Django](https://www.djangoproject.com/), [daisyUI](https://daisyui.com/) & [Alpine.js](https://alpinejs.dev/) to visualize our scraped data.
