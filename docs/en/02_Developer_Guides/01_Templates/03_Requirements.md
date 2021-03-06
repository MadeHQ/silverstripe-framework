title: Requirements
summary: How to include and require other assets in your templates such as javascript and CSS files.

# Requirements

The requirements class takes care of including CSS and JavaScript into your applications. This is preferred to hard 
coding any references in the `<head>` tag of your template, as it enables a more flexible handling through the 
[api:Requirements] class.

The examples below are using certain folder naming conventions (CSS files in `css/`, JavaScript files in `javascript/`).
SilverStripe core modules like `cms` use a different naming convention (CSS and JavaScript files in `client/src/`).
The `Requirements` class can work with arbitrary file paths.

## Template Requirements API

**<my-module-dir>/templates/SomeTemplate.ss**

	:::ss
	<% require css("<my-module-dir>/css/some_file.css") %>
	<% require themedCSS("some_themed_file") %>
	<% require javascript("<my-module-dir>/javascript/some_file.js") %>

<div class="alert" markdown="1">
Requiring assets from the template is restricted compared to the PHP API.
</div>

## PHP Requirements API

It is common practice to include most Requirements either in the *init()*-method of your [controller](../controllers/), or
as close to rendering as possible (e.g. in [api:FormField]).

	:::php
	<?php

	class MyCustomController extends Controller {

        public function init() {
            parent::init();
		
            Requirements::javascript("<my-module-dir>/javascript/some_file.js");
            Requirements::css("<my-module-dir>/css/some_file.css");
        }
    }

### CSS Files

	:::php
    Requirements::css($path, $media);

If you're using the CSS method a second argument can be used. This argument defines the 'media' attribute of the 
`<link>` element, so you can define 'screen' or 'print' for example.

	:::php
    Requirements::css("<my-module-dir>/css/some_file.css", "screen,projection");

### Javascript Files

	:::php
    Requirements::javascript($path, $options);

A variant on the inclusion of custom javascript is the inclusion of *templated* javascript.  Here, you keep your
JavaScript in a separate file and instead load, via search and replace, several PHP-generated variables into that code.

	:::php
    $vars = array(
        "MemberID" => Member::currentUserID(),
    );

	Requirements::javascriptTemplate("<my-module-dir>/javascript/some_file.js", $vars);

In this example, `some_file.js` is expected to contain a replaceable variable expressed as `MemberID`.

If you are using front-end script combination mechanisms, you can optionally declare
that your included files provide these scripts. This will ensure that subsequent
Requirement calls that rely on those included scripts will not double include those
files.

    :::php
    Requirements::javascript('<my-module-dir>/javascript/dist/bundle.js', ['provides' => [
        '<my-module-dir>/javascript/jquery.js'
        '<my-module-dir>/javascript/src/main.js',
        '<my-module-dir>/javascript/src/functions.js'
    ]]);
    Requirements::javascript('<my-module-dir>/javascript/jquery.js'); // Will will skip this file

You can also use the second argumet to add the 'async' and/or 'defer attributes to the script tag generated:

	:::php
    Requirements::javascript(
    	"<my-module-dir>/javascript/some_file.js", 
    	array(
    		"async" => true,
    		"defer" => true,
    	)
    );

### Custom Inline CSS or Javascript

You can also quote custom script directly. This may seem a bit ugly, but is useful when you need to transfer some kind
of 'configuration' from the database in a raw format.  You'll need to use the `heredoc` syntax to quote JS and CSS, 
this is generally speaking the best way to do these things - it clearly marks the copy as belonging to a different
language.

	:::php
	Requirements::customScript(<<<JS
	  alert("hi there");
	JS
	);

	Requirements::customCSS(<<<CSS
	  .tree li.$className {
	    background-image: url($icon);
	  }
	CSS
	);

## Combining Files

You can concatenate several CSS or javascript files into a single dynamically generated file. This increases performance
by reducing HTTP requests.

	:::php
	Requirements::combine_files(
		'foobar.js',
		array(
			'<my-module-dir>/javascript/foo.js',
			'<my-module-dir>/javascript/bar.js',
		)
	);

<div class="alert" markdown='1'>
To make debugging easier in your local environment, combined files is disabled when running your application in `dev`
mode. You can re-enable dev combination by setting `Requirements_Backend.combine_in_dev` to true.
</div>

### Configuring combined file storage

In some situations or server configurations, it may be necessary to customise the behaviour of generated javascript
files in order to ensure that current files are served in requests.

By default, files will be generated on demand in the format `assets/_combinedfiles/name-<hash>.js`,
where `<hash>` represents the hash of the source files used to generate that content. The default flysystem backend,
as used by the `[api:AssetStore]` backend, is used for this storage, but it can be substituted for any
other backend.

You can also use any of the below options in order to tweak this behaviour:

 * `Requirements.disable_flush_combined` - By default all combined files are deleted on flush.
   If combined files are stored in source control, and thus updated manually, you might want to
   turn this on to disable this behaviour.
 * `Requirements_Backend.combine_hash_querystring` - By default the `<hash>` of the source files is appended to
   the end of the combined file (prior to the file extension). If combined files are versioned in source control,
   or running in a distributed environment (such as one where the newest version of a file may not always be
   immediately available) then it may sometimes be necessary to disable this. When this is set to true, the hash
   will instead be appended via a querystring parameter to enable cache busting, but not in the
   filename itself. I.e. `assets/_combinedfiles/name.js?m=<hash>`
 * `Requirements_Backend.default_combined_files_folder` - This defaults to `_combinedfiles`, and is the folder
   within the configured asset backend that combined files will be stored in. If using a backend shared with
   other systems, it is usually necessary to distinguish combined files from other assets.
 * `Requirements_Backend.combine_in_dev` - By default combined files will not be combined except in test
   or live environments. Turning this on will allow for pre-combining of files in development mode.

In some cases it may be necessary to create a new storage backend for combined files, if the default location
is not appropriate. Normally a single backend is used for all site assets, so a number of objects must be
replaced. For instance, the below will set a new set of dependencies to write to `mysite/javascript/combined`


    :::yaml
    ---
    Name: myrequirements
    ---
    SilverStripe\View\Requirements:
      disable_flush_combined: true
    SilverStripe\View\Requirements_Backend:
      combine_in_dev: true
      combine_hash_querystring: true
      default_combined_files_folder: 'combined'
    SilverStripe\Core\Injector\Injector:
      MySiteAdapter:
        class: 'SilverStripe\Filesystem\Flysystem\AssetAdapter'
        constructor:
          Root: ./mysite/javascript 
      # Define the default filesystem
      MySiteBackend:
        class: 'League\Flysystem\Filesystem'
        constructor:
          Adapter: '%$MySiteAdapter'
        calls:
          PublicURLPlugin: [ addPlugin, [ %$FlysystemUrlPlugin ] ]
      # Requirements config
      MySiteAssetHandler:
        class: SilverStripe\Filesystem\Storage\FlysystemGeneratedAssetHandler
        properties:
          Filesystem: '%$MySiteBackend'
      SilverStripe\View\Requirements_Backend:
        properties:
          AssetHandler: '%$MySiteAssetHandler'

In the above configuration, automatic expiry of generated files has been disabled, and it is necessary for
the developer to maintain these files manually. This may be useful in environments where assets must
be pre-cached, where scripts must be served alongside static files, or where no framework php request is
guaranteed. Alternatively, files may be served from instances other than the one which generated the
page response, and file synchronisation might not occur fast enough to propagate combined files to
mirrored filesystems.

In any case, care should be taken to determine the mechanism appropriate for your development
and production environments.

### Combined CSS Files

You can also combine CSS files into a media-specific stylesheets as you would with the `Requirements::css` call - use
the third paramter of the `combine_files` function:

```php
$loader = ThemeResourceLoader::instance();
$themes = SSViewer::get_themes();

$printStylesheets = array(
	$loader->findThemedCSS('print_HomePage.css', $themes),
	$loader->findThemedCSS('print_Page.css', $themes)
);

Requirements::combine_files('print.css', $printStylesheets, 'print');
```

By default, all requirements files are flushed (deleted) when ?flush querystring parameter is set.
This can be disabled by setting the `Requirements.disable_flush_combined` config to `true`.

<div class="alert" markdown='1'>
When combining CSS files, take care of relative urls, as these will not be re-written to match
the destination location of the resulting combined CSS.
</div>

### Combined JS Files

You can also add the 'async' and/or 'defer' attributes to combined Javascript files as you would with the 
`Requirements::javascript` call - use the third paramter of the `combine_files` function:

```php
$loader = ThemeResourceLoader::instance();
$themes = SSViewer::get_themes();

$scripts = array(
    $loader->findThemedJavascript('some_script.js', $themes),
    $loader->findThemedJavascript('some_other_script.js', $themes)
);

Requirements::combine_files('scripts.js', $scripts, array('async' => true, 'defer' => true));
```

### Minification of CSS and JS files

You can minify combined Javascript and CSS files at runtime using an implementation of the
`SilverStripe\View\Requirements_Minifier` interface.

```php
namespace MyProject;

use SilverStripe\View\Requirements_Minifier;

class MyMinifier implements Requirements_Minifier
{
    /**
     * Minify the given content
     *
     * @param string $content
     * @param string $type Either js or css
     * @param string $filename Name of file to display in case of error
     * @return string minified content
     */	
	public function minify ($content, $type, $fileName)
	{
		// Minify $content;

		return $minifiedContent;
	}
}
```

Then, inject this service in `Requirements_Backend`.

```yaml
SilverStripe\Core\Injector\Injector:
  SilverStripe\View\Requirements_Backend:
    properties:
      Minifier: %$MyProject\MyMinifier
```

<div class="alert" markdown='1'>
While the framework does afford you the option of minification at runtime, we recommend using one of many frontend build
tools to do this for you, e.g. [Webpack](https://webpack.github.io/), [Gulp](http://gulpjs.com/), or [Grunt](https://gruntjs.com/).
</div>


## Clearing assets

	:::php
	Requirements::clear();

Clears all defined requirements. You can also clear specific requirements.

	:::php
	Requirements::clear(THIRDPARTY_DIR.'/prototype.js');

<div class="alert" markdown="1">
Depending on where you call this command, a Requirement might be *re-included* afterwards.
</div>

## Blocking

Requirements can also be explicitly blocked from inclusion, which is useful to avoid conflicting JavaScript logic or 
CSS rules. These blocking rules are independent of where the `block()` call is made. It applies both for already 
included requirements, and ones included after the `block()` call.

One common example is to block the core `jquery.js` added by various form fields and core controllers, and use a newer 
version in a custom location. This assumes you have tested your application with the newer version.

	:::php
	Requirements::block(ADMIN_THIRDPARTY_DIR . '/jquery/jquery.js');

<div class="alert" markdown="1">
The CMS also uses the `Requirements` system, and its operation can be affected by `block()` calls. Avoid this by 
limiting the scope of your blocking operations, e.g. in `init()` of your controller.
</div>

## Inclusion Order

Requirements acts like a stack, where everything is rendered sequentially in the order it was included. There is no way
to change inclusion-order, other than using *Requirements::clear* and rebuilding the whole set of requirements. 

<div class="alert" markdown="1">
Inclusion order is both relevant for CSS and Javascript files in terms of dependencies, inheritance and overlays - be 
careful when messing with the order of requirements.
</div>

## Javascript placement

By default, SilverStripe includes all Javascript files at the bottom of the page body, unless there's another script 
already loaded, then, it's inserted before the first `<script>` tag. If this causes problems, it can be configured.

	:::php
	Requirements::set_force_js_to_bottom(true);

`Requirements.force_js_to_bottom`, will force SilverStripe to write the Javascript to the bottom of the page body, even 
if there is an earlier script tag.

If the Javascript files are preferred to be placed in the `<head>` tag rather than in the `<body>` tag,
`Requirements.write_js_to_body` should be set to false.

	:::php
	Requirements::set_write_js_to_body(false);


## API Documentation

 * [api:Requirements]
 * [CMS Architecture and Build Tooling](/developer_guides/customising_the_admin_interface/cms_architecture)
