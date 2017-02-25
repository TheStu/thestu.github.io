---
layout: post
title: Scraping Thousands of Amazon Pages
---

There is a whole lot of useful data locked up in the Amazon marketplace, but not a lot of good ways to get at it. Amazon provides an API through their Affiliates program, but it's old and brittle. The responses are XML, rather than something more usable like JSON, and it's missing a lot of useful data that you'd think would be included, like prices on ebooks, or any information having to do with reviews.

So, let's scrape instead.

Which is where the pain begins. Amazon doesn't make it easy on us to scrape anything, which is totally understandable. Setting a user agent string on your request gets you past the rookie level challenge of a straight up 503 status response. Waiting for 1 second in between requests also seemed like a good idea. A courteous scraper script.

The first thing I tried was a fairly basic web interaction gem, namely [Mechanize](https://github.com/sparklemotion/mechanize). This works, but Amazon seems to recognize it as a javascript-less browser at least some of the time, and serves up an entirely different html version of the page, which means everything downstream, that is built on a specific html structure, breaks.

Mechanize also gets met with Amazon's main line of defense against scraping, which is the occasional Captcha challenge. Once you get hit wth the Captcha challenge you'll generally get it again and again until you go away and leave them alone for awhile. And ideally I'd like to be scraping as quickly as possible, so waiting isn't an option.

So out with Mechanize, and in with something that looks more like an actual browser. I tried a few different options before settling on a combination of [Capybara](https://github.com/teamcapybara/capybara) / [Poltergeist](https://github.com/teampoltergeist/poltergeist).

{% highlight ruby %}
session = Capybara::Session.new(:poltergeist)
session.driver.add_headers("User-Agent" => "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.98 Safari/537.36")
session.visit('http://amazon.com')
{% endhighlight %}

This duo solved the Captcha challenge problem, but I was planning on running this on Heroku, and Poltergeist requires a headless browser to be installed (Firefox generally, but there's a Chrome version available too). But installing something like that is problematic on Heroku.

After searching around for awhile, I found an alternative to the headless browser - [PhantomJS](https://github.com/stomita/heroku-buildpack-phantomjs). The PhantomJS can be added to Heroku, and once installed, will run Poltergeist.

{% highlight terminal %}
heroku buildpacks:add --index 2 https://github.com/stomita/heroku-buildpack-phantomjs
{% endhighlight %}

The index 2 flag tells Heroku that you want this buildpack to be run after the buildpack that runs your app. In my case I'm using the heroku/ruby buildpack to run a Rails app.

With everything running, I could finally get down to the work of actually scraping some stuff. Even with all this work, Amazon still complained in the form of a Captcha challenge, a lot, when trying to scrape any individual product page. They don't seem to appreciate those particular pages being scraped.

I'm still not sure if there's some further step that could be taken to get past this, or if Amazon is using something sophisticated to profile scrapers, that would take a lot of time and effort to figure out and bypass.

But this was just another setback. No problem. Amazon seems to have zero problems with index pages being scraped. So I took the data that I couldn't get from the API from the index pages instead, and grabbed everything that I would have gotten from the product page from the API instead. This is a bit circuitous, but it gets the job done. By combining the two I was able to get all the data I needed.

With this combination I'm able to grab a little over 30,000 database rows of data per day, plus a whole lot more associated data. With the 1 second delay in between each request it takes about 14 hours to complete, but I don't need more data than that, so getting it done faster is not worth any extra effort.

In the future, depending on how useful the data I'm getting is, there are a couple options for expanding. One is to use a pool of IP proxies to make it look like the requests are unrelated. The main benefit of this approach is that I wouldn't have to worry about the 1 second delay in between requests, and if one IP gets met with a Captcha challenge, I can just move on to the next IP.

Another option seriously worth considering is [Scraping Hub](https://scrapinghub.com/). Scraping smaller websites is simple enough... no Captcha challenges, no 503s. But for something more challenging like Amazon, it might be worth it just to outsource the headaches to someone else. If the project all of this is based on gets much larger, and especially if I start running in to more Captcha problems, I would probably just outsource the scraping to the professionals.