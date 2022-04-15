# Twill Transformers

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE.md)
[![Build Status][ico-travis]][link-travis]
[![Coverage Status][ico-scrutinizer]][link-scrutinizer]
[![Quality Score][ico-code-quality]][link-code-quality]
[![Total Downloads][ico-downloads]][link-downloads]

This package allows you to create transformers to generate view data for your Twill app. It contains a base Transformer 
class and a series of traits, allowing you not only to transform model data, but also generate all blocks, from Twill's block editor and preview data.

## Reasoning

The main class of this package was extracted from the work we did for a client where we decided to use Storybook and Twig templates 
to build the front end. The idea is to free the back end developer from writing front-end code. For this to happen, the whole data
generation is automated, starting from the controller `view()` call.

## Install

### Via Composer

```bash
composer require area17/twill-transformers
```

### Publish the config file

```bash
php artisan vendor:publish --provider="A17\TwillTransformers\ServiceProvider"
```

## Usage

For those using the same approach (Storybook + Twig), this is what you have to do to make it all happen:

### Create your base Transformer

Extend the class Transformer and create your basic page structure. All JSON generated by the transformer will have this structure. 

```php
namespace App\Transformers;

use A17\TwillTransformers\Transformer as TwillTransformer;

abstract class Transformer extends TwillTransformer
{
    /**
     * @return array|null
     */
    public function transform()
    {
        return $this->sanitize([
            'template_name' => $this->makeTemplateName(),

            'header' => $this->transformHeader($this->data),

            $this->makePageKey() => $this->transformData($this->data),

            'seo' => $this->transformSeo($this->data),

            'footer' => $this->transformFooter($this->data),
        ]);
    }
}
```

### Use required traits

Add and use this one on your base controller:

```php
use A17\TwillTransformers\ControllerTrait;
```

And this one on your base repository:

```php
use A17\TwillTransformers\RepositoryTrait;
```

### Create your first Transformer

Note that data to be transformed is self-contained inside the transformer object. So `$this` holds everything it's responsible for transforming, and as it's usually a Laravel Model descendent, it also has access to everything we usually do with models, accessors, mutators, relationships, presenters, everything. 

```php
namespace App\Transformers;

class LandingPage extends Transformer
{
    public function transform()
    {
        return [
            'hero' => [
                'title' => $this->title,

                'text' => $this->text,

                'image' => $this->transformMedia(),
            ],

            'blocks' => $this->transformBlocks(),

            'related' => $this->transformRelatedArticles(),
        ];
    }
}
```

### Create block transformers  

Note that this is only required if your blocks are not compatible with the FE data needs. In order to reuse Twill blocks, BE and FE must agree on the structure and naming of all blocks. But, if you still have data transformation to be done on a block, you can create blocks as this: 

```php
namespace App\Transformers\Block;

use App\Transformers\Block;

class ArtistPortrait extends Block
{
    public function transform()
    {
        return [
            'component' => 'portrait',

            'data' => [
                'name' => $this->name,

                'text' => $this->main_info,

                'button' => [
                    'more_label' => ___('Lire plus'),
                    'less_label' => ___('Lire moins'),
                ],

                'extra_text' => $this->additional_info,

                'image' => $this->transformMedia(),
            ],
        ];
    }
}
```

### Reuse blocks as components  

If you have a block transformer and needs to reuse it to generate data on your App transformers, or even other block transformers, you can basically call them this way:

```php
public function transformArtistPortraits($portraits)
{
    return collect($portraits)->map(function ($portrait) {
        return $this->transformBlockArtistPortrait($portrait);
    });
}
```

If the `transform` method call starts with `Block`, like in `transformBlockArtistPortrait()`, it basically will try to instantiate the named block transformer class to be used.

This is the code for the block being called above:

```php
namespace App\Transformers\Block;

use App\Transformers\Block;

class ArtistPortrait extends Block
{
    public function transform()
    {
        return [
            'component' => 'portrait',

            'data' => [
                'name' => $this->name,

                'text' => $this->main_info,

                'button' => [
                    'more_label' => ___('Lire plus'),
                    'less_label' => ___('Lire moins'),
                ],

                'extra_text' => $this->additional_info,

                'image' => $this->transformMedia(),
            ],
        ];
    }
}
```

Note that the data (a model, an array, an object) passed to your transformer (block or app transformer) can be accessed using `$this` from inside your block:

So, when we call a transformer like this: 

```php
$this->transformBlockArtistPortrait($portrait);
```

Inside the transformer, we can render images just by doing:

```php
$this->transformMedia()
```

Or get the name of a portrait person this way:

```php
$this->name
```

### Rendering the front-end

This is all you have to do to send JSON data to your front-end:

```php
@extends('front.layouts.app')

@section('contents')
    @include("templates.{$template_name}")
@endsection
```

And if you need to take a look at the generated data, you just have to add `?output=json` the URL.

### Rendering previews on Twill

Previews are included, they are basically a `side effect` of this new approach, so you just have to configure your preview path on `twill.php` file:
 
```php
'views_path' => 'admin._site.front',
```

And create this Blade template to your render previews for everything:  

```php
@extends('front.layouts.app')

@php
    $data = _transform($item);
@endphp

@section('contents')
    @include("templates.{$data['template_name']}", $data)
@endsection
```

### Twill's block editor

First, you need to have a block component for every block you render, even if it's only extending a base component from your app. 

Then you just need to create this view, that will handle and render all blocks in the editor:

```php
@php
    $transformed = _transform($block);

    $type = Str::kebab(Str::camel($transformed['type']));
@endphp

@include("components.block.{$type}.block-{$type}", $transformed)
```

## Blade Transformers

You can define the transformer directly inside a Blade template:

```blade
@extends('layouts.app')

@transformer(\App\Transformers\Post)

...
```

On your base Transformer add a `blade()` static radmethod to handle the data transformation:

```php
public static function blade($transformer, $data): array
{
    if (app()->bound(BladeTransformer::class)) {
        return app(BladeTransformer::class)->transform($transformer, $data);
    }

    return [];
}
```

Then on each Transformer called from Blade, you can define a `transformStorybookData()` method to render fake data in case there is no data to be transformed available. This can be used, for instance, when rendering Storybook stories via [Blast](https://github.com/area17/blast).  

```php
<?php

namespace App\Transformers;

class Posts extends Transformer
{
    public function transform(): array
    {
        return [
            'title' => $this->title,
        ];
    }

    public function transformStorybookData(): array
    {
        return [
            'title' => 'Fake Title to Be Displayed Inside Storybook Only',
        ];
    }
}
```

If you are building a Storybook story using Blast, your story should extend the application layout 

```blade
@extends('layouts.app')

@transformer(\App\Transformers\Post)

@storybook([
    'layout' => 'fullscreen',
    'status' => 'dev',
    'args' => []
])

@section('content')
    <div class="container">
        ...
    </div>
@endsection
```

And your layout should check if it's running in Blast and only render the story component:  

```php
@if (runningInBlast(get_defined_vars()))
    @yield('content')
@else
    <!DOCTYPE html>
    <html lang="{{ str_replace('_', '-', app()->getLocale()) }}" data-active-nav="@yield('active_nav')">
        <head>
            ...
        </head>
        
        <body>
            ...
        </body>
    </html>
@endif
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Testing

```bash
$ composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) and [CODE_OF_CONDUCT](CODE_OF_CONDUCT.md) for details.

## Security

If you discover any security-related issues, please email antonio@area17.com instead of using the issue tracker.

## Credits

- [AREA 17](https://github.com/area17)
- [Antonio Ribeiro][link-author]
- [All Contributors][link-contributors]

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

[ico-version]: https://img.shields.io/packagist/v/area17/twill-transformers.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square
[ico-travis]: https://img.shields.io/travis/area17/twill-transformers/master.svg?style=flat-square
[ico-scrutinizer]: https://img.shields.io/scrutinizer/coverage/g/area17/twill-transformers.svg?style=flat-square
[ico-code-quality]: https://img.shields.io/scrutinizer/g/area17/twill-transformers.svg?style=flat-square
[ico-downloads]: https://img.shields.io/packagist/dt/area17/twill-transformers.svg?style=flat-square

[link-packagist]: https://packagist.org/packages/area17/twill-transformers
[link-travis]: https://travis-ci.org/area17/twill-transformers
[link-scrutinizer]: https://scrutinizer-ci.com/g/area17/twill-transformers/code-structure
[link-code-quality]: https://scrutinizer-ci.com/g/area17/twill-transformers
[link-downloads]: https://packagist.org/packages/area17/twill-transformers
[link-author]: https://github.com/antonioribeiro
[link-contributors]: ../../contributors
