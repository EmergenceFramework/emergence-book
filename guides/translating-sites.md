# Localization

## Install Localization Files on Server

First ensure the locales you'd like to create localization files for are installed on your server.

1. Check which locals are currently supported `locale -a`
2. Add the locales you want \(for example`es`\):

   `sudo locale-gen es_US`

   `sudo locale-gen es_US.UTF-8`

3. Run this update command

   `sudo update-locale`

4. Restart 'Web' in the Emergence Portal \(:9083\)

## Marking translatable strings

To flag text in your dwoo or php files as translatable, simply wrap the text in a gettext\(\) or shortcut function.

See full details here: [http://emr.ge/docs/localization/marking](http://emr.ge/docs/localization/marking)

## Generating .pot Files

Emergence can generate a .pot file by scanning your entire code base and pulling out all translatable strings. This file will then be consumed by Poedit where they'll be transalted and exported into .po files. To generate a .pot file, go to the site-admin portal of your website and click 'Generate .pot file'.

## Generating .po Files

Open the .pot file generated in the previous step in [Poedit](https://poedit.net/). Once opened, create a new translation in the language of your choice. This is where you'll actually need to translate all the strings one by one. Once finished, save the .po file out to site.po \(which you'll upload in the next step\).

## Uploading .po Files

You'll now want to upload the site.po files generated in the previous step into each emergence site you'd like to support. Upload each file using the following pattern, where es stands for Spanish.

`/locales/es_US.utf8/site.po`

The /locales directory is at the same level as your php-classes directory. If it's not already present, you'll be able to find it in the \_parent directory.

**Note:** to allow users easy access to switch back to English, you'll want to add a en\_US file that contains all the bare English words. This is important because in the last step, we'll generate a select box from the supported languages and if English isn't present it won't be available.

### Verify Files

To verify you've added the files to the correct location, make sure you see all the expected languages by calling the following function:

`Emergence\Locale::getAvailableLocales()`

## Turning Localization On

To enable localization the `Emergence\Locale::loadRequestedLocale();` needs to be called on every request. This is best done in an event handler on the Site classes' before script execute function. When this function it called, it will take the site.po file provided and generate a .mo file on the fly. This file will be saved in `/emergence/sites/HANDLE/site-data/locales/` directory.

`/event-handlers/Site/beforeScriptExecute/localization.php`

```text
<?php

Emergence\Locale::loadRequestedLocale();
```

## Switching Locale

### Specify via query string parameter with each request

By appending`?locale=es_US`to a request URI you can select the locale es\_US.utf8 for a single response.

### Set via client-side cookie

By setting a cookie called`locale`to`es_US`you can switch the locale for all responses during the life of the cookie. This is the recommended technique for implementing a user-selectable locale. You can set the cookie either client-side with JavaScript or server side with a response header.

### Send Accept-Language header with requests

If the browser or client application provides an Accept-Language header, Emergence will attempt to pick the closest available locale.

### Set default language site-wide

`/php-config/Emergence/Locale.config.d/default.php`

```text
Emergence\Locale::$default = 'en_US.utf8';
```
