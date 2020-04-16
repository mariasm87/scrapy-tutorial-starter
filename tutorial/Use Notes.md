Notes based on :    https://towardsdatascience.com/a-minimalist-end-to-end-scrapy-tutorial-part-i-11e350bcdec0
                    https://docs.scrapy.org/en/latest/intro/tutorial.html 
# Creating a project
The following command will create a "tutotial" directory with all the necessary contents to use scrapy:
scrapy startproject tutorial

# Spiders
## Create a spider
Spiders are classes that inform Scrapy how to scrap a website.A python file has to be created under "tutorial/spiders" directory with the following structure:
```
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        page = response.url.split("/")[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)
```

This is a spider subclass scrapy.Spider and defines some attributes and methods:

name: identifies the Spider. It must be unique within a project, that is, you can’t set the same name for different Spiders.

start_requests(): must return an iterable of Requests (you can return a list of requests or write a generator function) which the Spider will begin to crawl from. Subsequent requests will be generated successively from these initial requests.

parse(): a method that will be called to handle the response downloaded for each of the requests made. The response parameter is an instance of TextResponse that holds the page content and has further helpful methods to handle it.

The parse() method usually parses the response, extracting the scraped data as dicts and also finding new URLs to follow and creating new requests (Request) from them.

## Run a spider
The following command will run the "quotes" spider defined above:
scrapy crawl quotes

## Extracting data
Once the spider is run, the parse function creates two files that contain all the website information. To extract data from those files, selectors should be used with the scrapy shell:
scrapy shell "http://quotes.toscrape.com/page/1/"

Now we can select each CSS element using the response object:
```
response.css('title')
[<Selector xpath='descendant-or-self::title' data='<title>Quotes to Scrape</title>'>]
```

In order to extract only the text without tags:
```
response.css('title::text').getall()
['Quotes to Scrape']
```

the result of calling .getall() is a list: it is possible that a selector returns more than one result, so we extract them all. When you know you just want the first result, as in this case, you can do:
```
response.css('title::text')[0].get()
'Quotes to Scrape'
```

Besides the getall() and get() methods, you can also use the re() method to extract using regular expressions:
```
response.css('title::text').re(r'Quotes.*')
['Quotes to Scrape']
response.css('title::text').re(r'Q\w+')
['Quotes']
response.css('title::text').re(r'(\w+) to (\w+)')
['Quotes', 'Scrape']
```

In order to find the proper CSS selectors to use, **use your browser’s developer tools to inspect the HTML**.

Note: XPath could be used instead of CSS.

### Practical example

We have an html file extracted with the following contents:
```
<div class="quote">
    <span class="text">“The world as we have created it is a process of our
    thinking. It cannot be changed without changing our thinking.”</span>
    <span>
        by <small class="author">Albert Einstein</small>
        <a href="/author/Albert-Einstein">(about)</a>
    </span>
    <div class="tags">
        Tags:
        <a class="tag" href="/tag/change/page/1/">change</a>
        <a class="tag" href="/tag/deep-thoughts/page/1/">deep-thoughts</a>
        <a class="tag" href="/tag/thinking/page/1/">thinking</a>
        <a class="tag" href="/tag/world/page/1/">world</a>
    </div>
</div>
```

If we want to get the quote, author and tags, we should select the following items:
```
quote = response.css("div.quote")[0]
text = quote.css("span.text::text").get()
author = quote.css("small.author::text").get()
tags = quote.css("div.tags a.tag::text").getall()
```

Therefore, the final spider to extract all the information from the website could be:

```
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
                'tags': quote.css('div.tags a.tag::text').getall(),
            }
```
## Storing the scrapped data

Run the command:
scrapy crawl quotes -o quotes.json

## Following links
We need to find the link(href) for the following page by inspecting the current page:
```
<ul class="pager">
    <li class="next">
        <a href="/page/2/">Next <span aria-hidden="true">&rarr;</span></a>
    </li>
</ul>
```
To select the link, use:
```
response.css('li.next a').attrib['href']
'/page/2/'
```
And follow the next page as:
```
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('span small::text').get(),
                'tags': quote.css('div.tags a.tag::text').getall(),
            }

        next_page = response.css('li.next a::attr(href)').get()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
```
## Spider arguments
You can provide command line arguments to your spiders by using the -a option when running them:

scrapy crawl quotes -o quotes-humor.json -a tag=humor

These arguments are passed to the Spider’s __init__ method and become spider attributes by default.

In this example, the value provided for the tag argument will be available via self.tag. You can use this to make your spider fetch only quotes with a specific tag, building the URL based on the argument:

```
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"

    def start_requests(self):
        url = 'http://quotes.toscrape.com/'
        tag = getattr(self, 'tag', None)
        if tag is not None:
            url = url + 'tag/' + tag
        yield scrapy.Request(url, self.parse)

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
            }

        next_page = response.css('li.next a::attr(href)').get()
        if next_page is not None:
            yield response.follow(next_page, self.parse)
```
If you pass the tag=humor argument to this spider, you’ll notice that it will only visit URLs from the humor tag, such as http://quotes.toscrape.com/tag/humor


