Webapp Langpacks Spec
=====================

This document outlines the design of the language package support for webapps 
and instant webapps.  The design revolves around special webapps 
which can register themselves to provide localization resources for multiple 
apps and multiple languages at once.

There are two independent parts of the language package system:

 - language negotiation, which requires the localization library to be able to 
   query the platform for languages available for the current app, both 
   bundled with the app and installed by the user as langpacks,

 - localization resources IO, which fetches the resources either from the 
   current app or a user-installed langpack.

This proposal relies on the app installation process to set Gecko prefs which 
store the list of languages available for the given app, and the app protocol 
handler to redirect l10n resource IO to langpacks, if necessary.

This proposal relies on new native API methods which provide a way to query the 
platform for any additional languages available for the current app as well as 
to fetch resources provided by other apps for the current app.

(In the following examples we use the Email app to illustrate the design.)


App's HTML
----------

In HTML, language resources are linked like this:

    <link 
      rel="localization"
      href="/locales/email.{locale}.properties">

The information about the available and the default languages is stored in 
`meta` elements:

    <meta name="defaultLanguage" content="en-US">
    <meta name="availableLanguages" content="en-US:2.2-1, de:2.2-1">


mozApps.getAdditionalLanguages()
--------------------------------

When a document is opened the l10n.js library is tasked with negotiating the 
correct language to display the UI in to the user.  It uses 
`navigator.languages` to get the list of user's preferred langs, the 
`mozApps.getAdditionalLanguages` API and the `meta` elements to get the list of 
languages available for the current app, as well as the default language.  The 
result of the negotiation is a list of supported locales.

The information about extra languages available for the current app is stored 
in a Gecko pref and can be accessed via the `mozApps` API.

    mozApps.getAdditionalLanguages(manifestURI);

This method returns a `DOMRequest` which resolves to the JSON-parsed value of 
the `'webapps.languages.' + appId` Gecko pref.

The information about the bundled and the default languages is stored in `meta` 
elements and can be accessed synchronously.

An example code in l10n.js using this API could look like this:

    navigator.mozApps.getAdditionalLanguages(manifestURI).then(
      function(extraLangs) {
        /*
         * Use <meta name="availableLanguages"> to get the list of 
         * languages bundled with the app. Append extraLangs to it.   
         * Negotiate supported languages using navigator.languages, the 
         * list of all available languages including those coming from 
         * langpacks and the value of <meta name="defaultLanguage">.
         */
      });

When a langpack is installed or removed, an `availablelanguageschange` event is 
dispatched on the `document` with the list of extra languages provided as 
a `languages` event detail.


Resource IO
-----------

Once the l10n.js library has negotiated the list of supported languages, it can 
resolve the URL templates from `link` elements and fetch the resources in the 
correct language.

First, l10n.js resolves the URI of the resource to fetch:

    /locales/email.{locale}.properties

…becomes:

    /locales/email.de.properties

Since l10n.js also knows which languages come from the app and which come from 
langpacks (from the step before), it can decide which method of IO to use:

  - For languages bundled with the app, the resources are fetched using regular 
    XHR requests.

  - For languages provided by langpacks, the resources are fetched via the 
    `mozApps.getResource()` native API.


mozApps.getResource()
---------------------

This API provides a way for userland code to request resources provided for 
a specific app by some other app.

    mozApps.getResource(manifestURI, resourcePath);

It is not a method of the `App` object to allow instant webapps (which are not 
installed) to benefit from langpacks.


Langpack App Manifest
---------------------

Langpacks are webapp with a special 'langpack' role.  They're hidden from the 
Homescreen and can't be launched.  They act as bundles of resources which 
override requests to certain origins.  A single langpack can contain resources 
for multiple apps and multiple languages.  An example of a langpack manifest 
with the origin `my-langpack.gaiamobile.org` looks like this:

    "role": "langpack",
    "languages-provided": {
      "app://calendar.gaiamobile.org/manifest.webapp": {
        "basepath": "/calendar",
        "languages": {
          "de": "2.2-4"
        }
      },
      "app://email.gaiamobile.org/manifest.webapp": {
        "basepath": "/email",
        "languages": {
          "de": "2.2-4",
          "pl": "2.2-7"
        }
      }
    }

A special `languages-provided` field defines languages and their versions for 
app origins.  This information is used for the language negotiation.

  1. Can we skip the protocol part of the manifest URI in the keys of the 
     `languages_provided` object?


Langpack Installation
---------------------

TBD.
    

Language Negotiation and Resource IO, with Langpacks
----------------------------------------------------

`mozApps.getAdditionalLanguages()` for a given app now returns the newer 
version of `de`, as well as adds `pl` to the list of available languages.  The 
l10n.js library can negotiate the UI language using user's preferred languages 
stored in `navigator.languages`.

Once the negotiation has completed, specific resources can be fetched.  Again, 
`{locale}` is replaced by l10n.js with the result of the negotiation.

    /locales/email.{locale}.properties

…becomes:

    /locales/email.pl.properties

Since 'pl' is a langpack language (one of the languages returned by 
`mozApps.getAvailableLanguages()`), l10n.js will use the native API to get the 
resource:

    mozApps.getResource(
      'app://email.gaiamobile.org/manifest.webapp',
      '/locales/email.pl.properties');


Langpack Uninstallation
-----------------------

TBD.


App Upgrade & Uninstallation
----------------------------

TBD.

Requirements
----------------------------

Platform Team:
 - Write support for Langpacks API
    - getAdditionalLanguages
    - getResource
 - Write custom logic for language pack installation
 - Write logic for storing and retrieving resources in chrome’s IDB
 - Write logic for firing additionallanguageschange event

Marketplace Team:
 - Write support for ‘language pack’ addon type
 - Provide REST API for retrieving list of available language packs

UX Team:
 - Design interaction flow for discovering, installing and selecting additional languages

Gaia Team:
 - Extend Settings to support the flow designed by UX

L10n Team:
 - Update l10n.js library to support new Langpack API on runtime and buildtime
