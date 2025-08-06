+++
author = "Andreas Lay"
title = "Scraping German Rental Price Data – Part II: Building a Dashboard with Django, DaisyUI & Alpine.js"
date = "2025-08-06"
description = "A frontend for our Scraped Apartment Listings"
tags = ["python", "scraping", "web development"]
categories = ["Python", "Data Engineering", "Web Scraping", "Django"]
ShowToc = true
TocOpen = true
+++

## Our Goal

In the [first part of this series](/personal-blog/posts/scraping-wg-gesucht-part-i) we scraped apartment listings from [wg-gesucht.de](https://www.wg-gesucht.de/). In this part we will build a small [Django](https://www.djangoproject.com/) application to serve our scraped data as a dashboard.

## Django Backend: Templating & Component Based Design using Django Cotton

We are using [Django](https://www.djangoproject.com/) as our backend. To keep it simple we are using [Django's templating system](https://docs.djangoproject.com/en/5.2/topics/templates/) to render HTML server-side. Django's template are somewhat controversial as they're not as feature rich as for example [Jinja2](https://jinja.palletsprojects.com/en/stable/) templates. However when paired with [Django Cotton](https://django-cotton.com/) you can write reusable components to which you can pass Python objects as arguments.

As required by Django templates are placed in the `templates` directory. Reusable components are placed in the `cotton` folder:

![](/personal-blog/wg-gesucht-folder-layout.png)

Then for example the `cotton/chart.html` component is used in the `index.html` page:

```html
<div x-data="{ infoType: 'price' }">
  <!-- Charts section -->
  <div class="divider"></div>
  <div class="w-full">
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-5 md:gap-10">
      <c-chart chart_id="1" col_prefix="avg_price" title="Rent by Offer Type" />
      <c-chart
        chart_id="2"
        col_prefix="avg_price_per_square_meter"
        title="Rent/m² by Offer Type"
      />
    </div>
  </div>
  <!-- Table section -->
  <div class="divider"></div>
  <c-table />
</div>
```

Note the `c` prefix in `<c-chart ...>` is required by `Django Cotton`.

## Styling: DaisyUI & Tailwind CSS

The frontend itself uses [DaisyUI](https://daisyui.com/): Pure [Tailwind CSS](https://tailwindcss.com/) components which are easily themeable. We basically only require the standalone Tailwind CSS executable and the daisyUI bundled JS file (see [installation instructions](https://daisyui.com/docs/install/django/)). Tailwind CSS will create an `output.css` file which we serve via Django. There are no JavaScript dependencies served with our frontend.

The advantage of using `DaisyUI` is that it's framework agnostic: As it's just CSS it's not tied to frameworks like `React` or `Svelte`. However this also means its components are more basic compared to e.g. [Material UI](https://mui.com/). For instance it does not provide a multiselect component, for this I am using [ChoicesJS](https://github.com/Choices-js/Choices) instead which is manually styled to fit in with the `DaisyUI` theme.

## Interactivity: AlpineJS

For some basic interactivity without having to write too much JavaScript I am using [AlpineJS](https://alpinejs.dev/). For example it's used to control the theme:

```html
<html
  x-data="{ theme: 'dark' }"
  lang="en"
  x-bind:data-theme="theme"
  x-init="theme = ((localStorage.getItem('theme') || theme) === 'dark') ? 'dark' : 'light'; $watch('theme', value => localStorage.setItem('theme', value));$nextTick(() => { $store.chartsChangeThemeCallbackList.forEach(callback => callback()) });"
></html>
```

It sets the `theme` variable intially to `dark`, binds this variable to the `data-theme` attribute and stores the user preference on initial load in local storage. We also watch for theme changes triggered by the user and store any chosen preference in local storage using `AlpineJS`'s [$watch](https://alpinejs.dev/magics/watch) magic method.
