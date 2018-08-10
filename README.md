# Laravel Intelligent Scraper

[![Latest Version](https://img.shields.io/github/release/softonic/laravel-intelligent-scraper.svg?style=flat-square)](https://github.com/softonic/laravel-intelligent-scraper/releases)
[![Software License](https://img.shields.io/badge/license-Apache%202.0-blue.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/travis/softonic/laravel-intelligent-scraper/master.svg?style=flat-square)](https://travis-ci.org/softonic/laravel-intelligent-scraper)
[![Total Downloads](https://img.shields.io/packagist/dt/softonic/laravel-intelligent-scraper.svg?style=flat-square)](https://packagist.org/packages/softonic/laravel-intelligent-scraper)

This packages offers a scraping solution that doesn't require to know the web HTML structure and it is autoconfigured
when some change is detected in the HTML structure. This allows you to continue scraping without manual intervention
during a long time.

## Installation

To install, use composer:

```bash
composer require softonic/laravel-intelligent-scraper
```

To publish the scraper config, you can use
```bash
php artisan vendor:publish --provider="Softonic\LaravelIntelligentScraper\ScraperProvider" --tag=config
```

## Configuration

There are two different options for the initial setup. The package can be 
[configured using datasets](#configuration-based-in-dataset) or 
[configured using Xpath](#configuration-based-in-xpath). Both ways produces the same result but 
depending on your Xpath knowledge you could prefer one or other.

### Configuration based in dataset

The first step is to know which data do you want to obtain from a page, so you must go to the page and choose all
the texts, images, metas, etc... that do you want to scrap and label them, so you can tell the scraper what do you want.

An example from microsoft store could be:
```php
<?php
use Softonic\LaravelIntelligentScraper\Scraper\Models\ScrapedDataset;

ScrapedDataset::create([
    'url'  => 'https://test.c/p/my-objective',
    'type' => 'Item-definition-1',
    'data' => [
        'title'     => 'My title',
        'body'      => 'This is the body content I want to get',
        'images'    => [
            'https://test.c/images/1.jpg',
            'https://test.c/images/2.jpg',
            'https://test.c/images/3.jpg',
        ],
    ],
]);
```

In this example we can see that we want different fields that we labeled arbitrarily. Some of theme have multiple
values, so we can scrap lists of items from pages.

With this single dataset we will be able to train our scraper and be able to scrap any page with the same structure.
Due to the pages usually have different structures depending on different variables, you should add different datasets
trying to cover maximum page variations possible. The scraper WILL NOT BE ABLE to scrap page variations not incorporated
in the dataset.

Once we did the job, all is ready to work. You should not care about updates always you have enought data in the dataset
to cover all the new modifications on the page, so the scraper will recalculate the modifications on the fly. You can 
check [how it works](how-it-works.md) to know much about the internals.

We will check more deeply how we can create a new dataset and what options are available in the next section.

#### Dataset creation

The dataset is composed by `url` and `data`. 
* The `url` part is simple, you just need to indicate the url from where you obtained the data.
* The `type` part gives a item name to the current dataset. This allows you to define multiple types.
* The `data` part is where you indicate what data and assign the label that you want to get. 
The data could be a list of items or a single item.

A basic example could be:
```php
<?php
use Softonic\LaravelIntelligentScraper\Scraper\Models\ScrapedDataset;

ScrapedDataset::create([
    'url'  => 'https://test.c/p/my-objective',
    'type' => 'Item-definition-1',
    'data' => [
        'title'     => 'My title',
        'body'      => 'This is the body content I want to get',
        'images'    => [
            'https://test.c/images/1.jpg',
            'https://test.c/images/2.jpg',
            'https://test.c/images/3.jpg',
        ],
    ],
]);
```

In this dataset we want that the text `My title` to be labeled as title and we also have a list of images that we want 
to be labeled as images. With this we have the flexibility to pick items one by one or in lists.

Sometimes we want to label some text that it is not clean in the HTML because it could include insivible characters like
`\r\n`. To avoid to deal with that, the dataset allows you to add regular expressions.

Example with `body` field as regexp:

```php
<?php
use Softonic\LaravelIntelligentScraper\Scraper\Models\ScrapedDataset;

ScrapedDataset::create([
    'url'  => 'https://test.c/p/my-objective',
    'data' => [
        'title'     => 'My title',
        'body'      => regexp('/^Body starts here, but it is do long that.*$/si'),
        'images'    => [
            'https://test.c/images/1.jpg',
            'https://test.c/images/2.jpg',
            'https://test.c/images/3.jpg',
        ],
    ],
]);
```

With this change we will ensure that we detect the `body` even if it has hidden characters. 

**IMPORTANT** The scraper tries to find the text in all the tags including children, so if you define a regular
expression without limit, like for example `/.*Body starts.*/` you will find the text in `<html>` element due to that 
text is inside some child element of `<html>`. So define regexp carefully.

### Configuration based in Xpath

After you collected all the Xpath from the HTML, you just need to create the configuration models. They looks like:
```php
<?php
use Softonic\LaravelIntelligentScraper\Scraper\Models\Configuration;

Configuration::create([
    'name' => 'title',
    'type' => 'Item-definition-1',
    'xpaths' => '//*[@id=title]',
]);

Configuration::create([
    'name' => 'category',
    'type' => 'Item-definition-1',
    'xpaths' => ['//*[@id=cat]', '//*[@id=long-cat]'],
]);
```

In the definition, you should give a name to the field to be scraped and identify it as a type. The xpaths field could
contain a string or an array of strings. This is because the HTML can contain different variations depending on the
specific page, you you can write a list of Xpath that will be checked in order giving the first result found.

## Usage

After configure the scraper, you just need to execute it like:
```php
<?php 
$scraper = resolve(\Softonic\LaravelIntelligentScraper\Scraper\Scraper::class);
$data = $scraper->getData('https://test.c/p/my-objective', 'Item-definition-1');

/**
 * Item-definition-1 defined as 3 fields tagged as: title, body and images 
 */
echo var_export($data);
/**
 * Output:
 * [
 *      'title' => ['My title'].
 *      'body' => ['This is the body content I want to get'],
 *      'images' => [
 *          'https://test.c/images/1.jpg',
 *          'https://test.c/images/2.jpg',
 *          'https://test.c/images/3.jpg',
 *      ],
 * ]
 */

```

All the output fields are arrays that can contain one or more results.

## Testing

`softonic/laravel-intelligent-scraper` has a [PHPUnit](https://phpunit.de) test suite and a coding style compliance test suite using [PHP CS Fixer](http://cs.sensiolabs.org/).

To run the tests, run the following command from the project folder.

``` bash
$ docker-compose run test
```

To run interactively using [PsySH](http://psysh.org/):
``` bash
$ docker-compose run psysh
```

## How it works?

The scraper is auto configurable, but needs an initial dataset or add a configuration. 
The dataset tells the configurator which data do you want and how to label it.

![Scrape process](./docs/images/diagram.png "Scrape process")

To be reconfigurable and conserve the dataset freshness the scraper store the latest data scraped.

```
# Powered by https://code2flow.com/app
function calculate configuration {
  if(!Has dataset?) {
    goto fail;
  }
  Extract configuration using dataset;
  if(!Has extracted configuration?) {
    goto fail;
  }
}

Scrape url 'https://test.c/p/my-onjective' using 'Item-definition-1';
try {
  load configuration;
}
catch(Missing config) {
  call calculate configuration;
}

extract data using configuration;
// It could be produced by old configuration
if(Error extracting data) {
  call calculate configuration;
}
extract data using configuration;
// It could be produced because the dataset does not have all the page variations
if(Error extracting data) {
  goto fail;
}

goto success

fail:
No scraped data;
return;
success:
Scraped data;
```

## License

The Apache 2.0 license. Please see [LICENSE](LICENSE) for more information.

[PSR-2]: http://www.php-fig.org/psr/psr-2/
[PSR-4]: http://www.php-fig.org/psr/psr-4/