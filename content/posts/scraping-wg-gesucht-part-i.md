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

Finding affordable appartments is hard. We may not be able to influence rental prices but at least we can bring some transparency into the market. Our goal: We want to collect appartment listings with corresponding meta information like rental prices, square meters and location. To achieve this we will:

1. Crawl [wg-gesucht.de](https://www.wg-gesucht.de/)
2. Write the results to a `sqlite` database
3. Write a small [Django](https://www.djangoproject.com/) to serve our scraped data as a dashboard

The final result will looks like this:

{{< iframe2 src="https://wg-watch-137914338338.europe-west1.run.app/" width="100%" height="400" >}}

This part of our series deals with the actual scraping.

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

In our code we simply use a templated string:

```python
URL_TEMPLATE = (
    "https://www.wg-gesucht.de/wg-zimmer-und-1-zimmer-wohnungen-und-wohnungen-und-haeuser"
    "-in-{city}.{city_id}.0+1+2+3.1.{page}.html?offer_filter=1"
    "&city_id={city_id}&sort_order=0&noDeact=1&categories%5B%5D=0"
    "&categories%5B%5D=1&categories%5B%5D=2&categories%5B%5D=3&pagination=4&pu="
)
```

We are going to query all listings without further filters.

## Our Humble Scraping Tech Stack
