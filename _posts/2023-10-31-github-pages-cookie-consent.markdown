---
layout: post
title: "Adding a cookie consent banner to Github Pages"
date: 2023-10-31 13:00:00 +0200
categories: github pages jekyll cookie consent
---
This is a quick guide on how to add a cookie consent banner to your Github Pages website, assuming you are using the default [Minima](https://github.com/jekyll/minima) theme (like I do.) I had to do this because I'm using Google Analytics to track visitor behaviour on my blog, which uses cookies. I started out following [this guide](https://lucaf.eu/2022/03/26/cookies-jekyll.html), which assumes you use Jekyll, but I got a little confused at the last step. It took me an hour or so to figure this out.

# Cookiebot
Following the guide I linked above, I signed up for [Cookiebot](https://www.cookiebot.com/), and followed the step-by-step process to configure my website. Basically, all you have to do is fill in the complete URL for your Github Pages site, which is `<Github username>.github.io`. In the admin page, you can navigate to Script Embeds and find the piece of HTML that you will need to embed in your website. It looks something like this:
```html
<script id="Cookiebot" src="https://consent.cookiebot.com/uc.js" data-cbid="..." data-blockingmode="auto" type="text/javascript"></script>
```
This piece of HTML needs to go in the `<head>` part of every page in your website, before any other `<script>` tags. Achieving this in Github Pages was a bit confusing for me at first, but it's actually very easy.

# Adding the script to head
The first thing I did, is to create a file `/_includes/cookiebot.html` and paste the code in there. Now, you need to override `head.html`. To do this, you will need to copy the file from the Minima gem directory to the `_includes` directory in your project, and edit it. In my case, the gem directory is in my home directory: `~/gems/gems/minima-2.5.1/_includes/head.html`. I simply copied this file into the `/_includes` directory of my Github Pages project.

Now edit the file and include the `cookiebot.html` file:
```html
{% raw %}
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  {%- seo -%}
  <link rel="stylesheet" href="{{ "/assets/main.css" | relative_url }}">
  {%- feed_meta -%}
  {%- if jekyll.environment == 'production' and site.google_analytics -%}
    {%- include cookiebot.html -%} <!-- Add this line -->
    {%- include google-analytics.html -%}
  {%- endif -%}
</head>
{% endraw %}
```
And make sure the include happens *before* the include of `google-analytics.html`.

That should be it. You can test your site locally to make sure everything still works: `bundle exec jekyll serve`. If everything is ok, merge your changes to your `main` branch and the Cookie consent pop-up should appear on your site!

# Testing your cookie consent banner
Just a while after I added the Cookiebot banner, I received a report from Cookiebot saying that there was a compliance issue on my website and that Google Analytics was being enabled without the visitor's consent. This was a bit confusing for me, because I thought I had done everything correctly. So I verified that the Cookiebot banner is working correctly in this way. I guess it only works if your website is not frequently visited:

1. Open Google Analytics' realtime page. If your website is anything like mine, it shows 0 visitors.
2. Now open a private browser window and go to your website. The Cookiebot banner should show up. Make sure Statistics is not selected and click on `Allow selection`
3. Check Google Analytics. It should still show 0 visitors (give it a minute to update.)
4. Now open a private browser window again, and go to your website. This time, allow all cookies.
5. Check Google Analytics again. It should show 1 visitor. If this is the case, congratulations, the cookie consent banner is working correctly!
