# Contributing

This guide have some instructions and tips on how to create a new Tachiyomi extension. Please **read it carefully** if you're a new contributor or don't have any experience on the required languages and knowledges.

This guide is not definitive and it's being updated over time. If you find any issue on it, feel free to report it through a [Meta Issue](https://github.com/tachiyomiorg/tachiyomi-extensions/issues/new?assignees=&labels=Meta+request&template=request_meta.yml) or fixing it directly by submitting a Pull Request.

## Table of Contents

1. [Prerequisites](#prerequisites)
   1. [Tools](#tools)
   2. [Cloning the repository](#cloning-the-repository)
2. [Getting help](#getting-help)
3. [Writing an extension](#writing-an-extension)
   1. [Setting up a new Gradle module](#setting-up-a-new-gradle-module)
   2. [Core dependencies](#core-dependencies)
   3. [Extension main class](#extension-main-class)
   4. [Extension call flow](#extension-call-flow)
   5. [Misc notes](#misc-notes)
   6. [Advanced extension features](#advanced-extension-features)
4. [Multi-source themes](#multi-source-themes)
   1. [The directory structure](#the-directory-structure)
   2. [Development workflow](#development-workflow)
   3. [Scaffolding overrides](#scaffolding-overrides)
   4. [Additional Notes](#additional-notes)
5. [Running](#running)

## Prerequisites

Before you start, please note that the ability to use following technologies is **required** and that existing contributors will not actively teach them to you.

- [Kotlin](https://kotlinlang.org/)
- Web scraping
    - [HTML](https://developer.mozilla.org/en-US/docs/Web/HTML)
    - [CSS selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors)
    - [OkHttp](https://square.github.io/okhttp/)
    - [JSoup](https://jsoup.org/)

### Tools

- Emulator or phone with developer options enabled and a recent version of Tachiyomi installed
- [Icon Generator](https://as280093.github.io/AndroidAssetStudio/icons-launcher.html)

### Cloning the repository

Some alternative steps can be followed to ignore "repo" branch and skip unrelated sources, which will make it faster to pull, navigate and build. This will also reduce disk usage and network traffic.

## Getting help

- Join [the Discord server](https://discord.gg/tachiyomi) for online help and to ask questions while developing your extension. When doing so, please ask it in the `#programming` channel.
- There are some features and tricks that are not explored in this document. Refer to existing extension code for examples.

## Writing an extension

The quickest way to get started is to copy an existing extension's folder structure and renaming it as needed. We also recommend reading through a few existing extensions' code before you start.

### Setting up a new Gradle module

Each extension should reside in `src/<lang>/<mysourcename>`. Use `all` as `<lang>` if your target source supports multiple languages or if it could support multiple sources.

The `<lang>` used in the folder inside `src` should be the major `language` part. For example, if you will be creating a `pt-BR` source, use `<lang>` here as `pt` only. Inside the source class, use the full locale string instead.

#### Extension file structure

The simplest extension structure looks like this:

```console
$ tree src/<lang>/<mysourcename>/
src/<lang>/<mysourcename>/
├── AndroidManifest.xml
├── build.gradle
├── res
│   ├── mipmap-hdpi
│   │   └── ic_launcher.png
│   ├── mipmap-mdpi
│   │   └── ic_launcher.png
│   ├── mipmap-xhdpi
│   │   └── ic_launcher.png
│   ├── mipmap-xxhdpi
│   │   └── ic_launcher.png
│   ├── mipmap-xxxhdpi
│   │   └── ic_launcher.png
│   └── web_hi_res_512.png
└── src
    └── eu
        └── kanade
            └── tachiyomi
                └── extension
                    └── <lang>
                        └── <mysourcename>
                            └── <MySourceName>.kt

13 directories, 9 files
```

#### AndroidManifest.xml
A minimal [Android manifest file](https://developer.android.com/guide/topics/manifest/manifest-intro) is needed for Android to recognize a extension when it's compiled into an APK file. You can also add intent filters inside this file (see [URL intent filter](#url-intent-filter) for more information).

#### build.gradle
Make sure that your new extension's `build.gradle` file follows the following structure:

```gradle
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'

ext {
    extName = '<My source name>'
    pkgNameSuffix = '<lang>.<mysourcename>'
    extClass = '.<MySourceName>'
    extVersionCode = 1
    isNsfw = true
}

apply from: "$rootDir/common.gradle"
```

| Field | Description |
| ----- | ----------- |
| `extName` | The name of the extension. |
| `pkgNameSuffix` | A unique suffix added to `eu.kanade.tachiyomi.extension`. The language and the site name should be enough. Remember your extension code implementation must be placed in this package. |
| `extClass` | Points to the class that implements `Source`. You can use a relative path starting with a dot (the package name is the base path). This is used to find and instantiate the source(s). |
| `extVersionCode` | The extension version code. This must be a positive integer and incremented with any change to the code. |
| `libVersion` | (Optional, defaults to `1.4`) The version of the [extensions library](https://github.com/tachiyomiorg/extensions-lib) used. |
| `isNsfw` | (Optional, defaults to `false`) Flag to indicate that a source contains NSFW content. |

The extension's version name is generated automatically by concatenating `libVersion` and `extVersionCode`. With the example used above, the version would be `1.4.1`.

### Core dependencies

#### Extension API

Extensions rely on [extensions-lib](https://github.com/tachiyomiorg/extensions-lib), which provides some interfaces and stubs from the [app](https://github.com/tachiyomiorg/tachiyomi) for compilation purposes. The actual implementations can be found [here](https://github.com/tachiyomiorg/tachiyomi/tree/master/app/src/main/java/eu/kanade/tachiyomi/source). Referencing the actual implementation will help with understanding extensions' call flow.

#### DataImage library

[`lib-dataimage`](https://github.com/tachiyomiorg/tachiyomi-extensions/tree/master/lib/dataimage) is a library for handling [base 64 encoded image data](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs) using an [OkHttp interceptor](https://square.github.io/okhttp/interceptors/).

```gradle
dependencies {
    implementation(project(':lib-dataimage'))
}
```

#### Additional dependencies

If you find yourself needing additional functionality, you can add more dependencies to your `build.gradle` file.
Many of [the dependencies](https://github.com/tachiyomiorg/tachiyomi/blob/master/app/build.gradle.kts) from the main Tachiyomi app are exposed to extensions by default.

> Note that several dependencies are already exposed to all extensions via Gradle version catalog.
> To view which are available view `libs.versions.toml` under the `gradle` folder

Notice that we're using `compileOnly` instead of `implementation` if the app already contains it. You could use `implementation` instead for a new dependency, or you prefer not to rely on whatever the main app has at the expense of app size.

Note that using `compileOnly` restricts you to versions that must be compatible with those used in [the latest stable version of Tachiyomi](https://github.com/tachiyomiorg/tachiyomi/releases/latest).

### Extension main class

The class which is referenced and defined by `extClass` in `build.gradle`. This class should implement either `SourceFactory` or extend one of the `Source` implementations: `HttpSource` or `ParsedHttpSource`.

| Class | Description |
| ----- | ----------- |
|`SourceFactory`| Used to expose multiple `Source`s. Use this in case of a source that supports multiple languages or mirrors of the same website. For similar websites use [theme sources](#multi-source-themes). |
| `HttpSource`| For online source, where requests are made using HTTP. |
| `ParsedHttpSource`| Similar to `HttpSource`, but has methods useful for scraping pages. |

#### Main class key variables

| Field | Description |
| ----- | ----------- |
| `name` | Name displayed in the "Sources" tab in Tachiyomi. |
| `baseUrl` | Base URL of the source without any trailing slashes. |
| `lang` | An ISO 639-1 compliant language code (two letters in lower case in most cases, but can also include the country/dialect part by using a simple dash character). |
| `id` | Identifier of your source, automatically set in `HttpSource`. It should only be manually overriden if you need to copy an existing autogenerated ID. |

### Extension call flow

#### Popular Manga

a.k.a. the Browse source entry point in the app (invoked by tapping on the source name).

- The app calls `fetchPopularManga` which should return a `MangasPage` containing the first batch of found `SManga` entries.
    - This method supports pagination. When user scrolls the manga list and more results must be fetched, the app calls it again with increasing `page` values (starting with `page=1`). This continues while `MangasPage.hasNextPage` is passed as `true` and `MangasPage.mangas` is not empty.
- To show the list properly, the app needs `url`, `title` and `thumbnail_url`. You **must** set them here. The rest of the fields could be filled later (refer to Manga Details below).
    - You should set `thumbnail_url` if is available, if not, `fetchMangaDetails` will be **immediately** called (this will increase network calls heavily and should be avoided).

#### Latest Manga

a.k.a. the Latest source entry point in the app (invoked by tapping on the "Latest" button beside the source name).

- Enabled if `supportsLatest` is `true` for a source
- Similar to popular manga, but should be fetching the latest entries from a source.

#### Manga Search

- When the user searches inside the app, `fetchSearchManga` will be called and the rest of the flow is similar to what happens with `fetchPopularManga`.
    - If search functionality is not available, return `Observable.just(MangasPage(emptyList(), false))`
- `getFilterList` will be called to get all filters and filter types.

##### Filters

The search flow have support to filters that can be added to a `FilterList` inside the `getFilterList` method. When the user changes the filters' state, they will be passed to the `searchRequest`, and they can be iterated to create the request (by getting the `filter.state` value, where the type varies depending on the `Filter` used). You can check the filter types available [here](https://github.com/tachiyomiorg/tachiyomi/blob/master/app/src/main/java/eu/kanade/tachiyomi/source/model/Filter.kt) and in the table below.

| Filter | State type | Description |
| ------ | ---------- | ----------- |
| `Filter.Header` | None | A simple header. Useful for separating sections in the list or showing any note or warning to the user. |
| `Filter.Separator` | None | A line separator. Useful for visual distinction between sections. |
| `Filter.Select<V>` | `Int` | A select control, similar to HTML's `<select>`. Only one item can be selected, and the state is the index of the selected one. |
| `Filter.Text` | `String` | A text control, similar to HTML's `<input type="text">`. |
| `Filter.CheckBox` | `Boolean` | A checkbox control, similar to HTML's `<input type="checkbox">`. The state is `true` if it's checked. |
| `Filter.TriState` | `Int` | A enhanced checkbox control that supports an excluding state. The state can be compared with `STATE_IGNORE`, `STATE_INCLUDE` and `STATE_EXCLUDE` constants of the class. |
| `Filter.Group<V>` | `List<V>` | A group of filters (preferentially of the same type). The state will be a `List` with all the states. |
| `Filter.Sort` | `Selection` | A control for sorting, with support for the ordering. The state indicates which item index is selected and if the sorting is `ascending`. |

All control filters can have a default state set. It's usually recommended if the source have filters to make the initial state match the popular manga list, so when the user open the filter sheet, the state is equal and represents the current manga showing.

The `Filter` classes can also be extended, so you can create new custom filters like the `UriPartFilter`:

```kotlin
open class UriPartFilter(displayName: String, private val vals: Array<Pair<String, String>>) :
    Filter.Select<String>(displayName, vals.map { it.first }.toTypedArray()) {
    fun toUriPart() = vals[state].second
}
```

#### Manga Details

- When user taps on a manga, `fetchMangaDetails` and `fetchChapterList` will be called and the results will be cached.
    - A `SManga` entry is identified by it's `url`.
- `fetchMangaDetails` is called to update a manga's details from when it was initialized earlier.
    - `SManga.initialized` tells the app if it should call `fetchMangaDetails`. If you are overriding `fetchMangaDetails`, make sure to pass it as `true`.
    - `SManga.genre` is a string containing list of all genres separated with `", "`.
    - `SManga.status` is an "enum" value. Refer to [the values in the `SManga` companion object](https://github.com/tachiyomiorg/extensions-lib/blob/master/library/src/main/java/eu/kanade/tachiyomi/source/model/SManga.kt#L24).
    - During a backup, only `url` and `title` are stored. To restore the rest of the manga data, the app calls `fetchMangaDetails`, so all fields should be (re)filled in if possible.
    - If a `SManga` is cached, `fetchMangaDetails` will be only called when the user does a manual update (Swipe-to-Refresh).
- `fetchChapterList` is called to display the chapter list.
    - **The list should be sorted descending by the source order**.
- `getMangaUrl` is called when the user taps "Open in WebView".
  - If the source uses an API to fetch the data, consider overriding this method to return the manga absolute URL in the website instead.
  - It defaults to the URL provided to the request in `mangaDetailsRequest`.

#### Chapter

- After a chapter list for the manga is fetched and the app is going to cache the data, `prepareNewChapter` will be called.
- `SChapter.date_upload` is the [UNIX Epoch time](https://en.wikipedia.org/wiki/Unix_time) **expressed in milliseconds**.
    - If you don't pass `SChapter.date_upload` and leave it zero, the app will use the default date instead, but it's recommended to always fill it if it's available.
    - To get the time in milliseconds from a date string, you can use a `SimpleDateFormat` like in the example below.

      ```kotlin
      private fun parseDate(dateStr: String): Long {
          return runCatching { DATE_FORMATTER.parse(dateStr)?.time }
              .getOrNull() ?: 0L
      }

      companion object {
          private val DATE_FORMATTER by lazy {
              SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.ENGLISH)
          }
      }
      ```

      Make sure you make the `SimpleDateFormat` a class constant or variable so it doesn't get recreated for every chapter. If you need to parse or format dates in manga description, create another instance since `SimpleDateFormat` is not thread-safe.
    - If the parsing have any problem, make sure to return `0L` so the app will use the default date instead.
    - The app will overwrite dates of existing old chapters **UNLESS** `0L` is returned.
    - The default date has [changed](https://github.com/tachiyomiorg/tachiyomi/pull/7197) in preview ≥ r4442 or stable > 0.13.4.
      - In older versions, the default date is always the fetch date.
      - In newer versions, this is the same if every (new) chapter has `0L` returned.
      - However, if the source only provides the upload date of the latest chapter, you can now set it to the latest chapter and leave other chapters default. The app will automatically set it (instead of fetch date) to every new chapter and leave old chapters' dates untouched.
- `getChapterUrl` is called when the user taps "Open in WebView" in the reader.
  - If the source uses an API to fetch the data, consider overriding this method to return the chapter absolute URL in the website instead.
  - It defaults to the URL provided to the request in `pageListRequest`.

#### Chapter Pages

- When user opens a chapter, `fetchPageList` will be called and it will return a list of `Page`s.
- While a chapter is open in the reader or is being downloaded, `fetchImageUrl` will be called to get URLs for each page of the manga if the `Page.imageUrl` is empty.
- If the source provides all the `Page.imageUrl`'s directly, you can fill them and let the `Page.url` empty, so the app will skip the `fetchImageUrl` source and call directly `fetchImage`.
- The `Page.url` and `Page.imageUrl` attributes **should be set as an absolute URL**.
- Chapter pages numbers start from `0`.
- The list of `Page`s should be returned already sorted, the `index` field is ignored.

### Misc notes

- Sometimes you may find no use for some inherited methods. If so just override them and throw exceptions: `throw UnsupportedOperationException("Not used.")`
- You probably will find `getUrlWithoutDomain` useful when parsing the target source URLs. Keep in mind there's a current issue with spaces in the URL though, so if you use it, replace all spaces with URL encoded characters (like `%20`).
- If possible try to stick to the general workflow from `HttpSource`/`ParsedHttpSource`; breaking them may cause you more headache than necessary.
- By implementing `ConfigurableSource` you can add settings to your source, which is backed by [`SharedPreferences`](https://developer.android.com/reference/android/content/SharedPreferences).

### Advanced Extension features

#### URL intent filter

Extensions can define URL intent filters by defining it inside a custom `AndroidManifest.xml` file.
For an example, refer to [the NHentai module's `AndroidManifest.xml` file](https://github.com/tachiyomiorg/tachiyomi-extensions/blob/master/src/all/nhentai/AndroidManifest.xml) and [its corresponding `NHUrlActivity` handler](https://github.com/tachiyomiorg/tachiyomi-extensions/blob/master/src/all/nhentai/src/eu/kanade/tachiyomi/extension/all/nhentai/NHUrlActivity.kt).

To test if the URL intent filter is working as expected, you can try opening the website in a browser and navigating to the endpoint that was added as a filter or clicking a hyperlink. Alternatively, you can use the `adb` command below.

```console
$ adb shell am start -d "<your-link>" -a android.intent.action.VIEW
```

#### Update strategy

There is some cases where titles in a source will always only have the same chapter list (i.e. immutable), and don't need to be included in a global update of the app because of that, saving a lot of requests and preventing causing unnecessary damage to the source servers. To change the update strategy of a `SManga`, use the `update_strategy` field. You can find below a description of the current possible values.

- `UpdateStrategy.ALWAYS_UPDATE`: Titles marked as always update will be included in the library update if they aren't excluded by additional restrictions.
- `UpdateStrategy.ONLY_FETCH_ONCE`: Titles marked as only fetch once will be automatically skipped during library updates. Useful for cases where the series is previously known to be finished and have only a single chapter, for example.

If not set, it defaults to `ALWAYS_UPDATE`.

#### Renaming existing sources

There is some cases where existing sources changes their name on the website. To correctly reflect these changes in the extension, you need to explicity set the `id` to the same old value, otherwise it will get changed by the new `name` value and users will be forced to migrate back to the source.

To get the current `id` value before the name change, you can search the source name in the [repository JSON file](https://github.com/tachiyomiorg/tachiyomi-extensions/blob/repo/index.json) by looking into the `sources` attribute of the extension. When you have the `id` copied, you can override it in the source:

```kotlin
override val id: Long = <the-id>
```

Then the class name and the `name` attribute value can be changed. Also don't forget to update the extension name and class name in the individual Gradle file if it is not a multisrc extension.

**Important:** the package name **needs** to be the same (even if it has the old name), otherwise users will not receive the extension update when it gets published in the repository. If you're changing the name of a multisrc source, you can manually set it in the generator class of the theme by using `pkgName = "oldpackagename"`.

The `id` also needs to be explicity set to the old value if you're changing the `lang` attribute.

## Multi-source themes
The `multisrc` module houses source code for generating extensions for cases where multiple source sites use the same site generator tool(usually a CMS) for bootsraping their website and this makes them similar enough to prompt code reuse through inheritance/composition; which from now on we will use the general **theme** term to refer to.

This module contains the *default implementation* for each theme and definitions for each source that builds upon that default implementation and also it's overrides upon that default implementation, all of this becomes a set of source code which then is used to generate individual extensions from.

### The directory structure
```console
$ tree multisrc
multisrc
├── build.gradle.kts
├── overrides
│   └── <themepkg>
│       ├── default
│       │   ├── additional.gradle
│       │   └── res
│       │       ├── mipmap-hdpi
│       │       │   └── ic_launcher.png
│       │       ├── mipmap-mdpi
│       │       │   └── ic_launcher.png
│       │       ├── mipmap-xhdpi
│       │       │   └── ic_launcher.png
│       │       ├── mipmap-xxhdpi
│       │       │   └── ic_launcher.png
│       │       ├── mipmap-xxxhdpi
│       │       │   └── ic_launcher.png
│       │       └── web_hi_res_512.png
│       └── <sourcepkg>
│           ├── additional.gradle
│           ├── AndroidManifest.xml
│           ├── res
│           │   ├── mipmap-hdpi
│           │   │   └── ic_launcher.png
│           │   ├── mipmap-mdpi
│           │   │   └── ic_launcher.png
│           │   ├── mipmap-xhdpi
│           │   │   └── ic_launcher.png
│           │   ├── mipmap-xxhdpi
│           │   │   └── ic_launcher.png
│           │   ├── mipmap-xxxhdpi
│           │   │   └── ic_launcher.png
│           │   └── web_hi_res_512.png
│           └── src
│               └── <SourceName>.kt
└── src
    └── main
        ├── AndroidManifest.xml
        └── java
            ├── eu
            │   └── kanade
            │       └── tachiyomi
            │           └── multisrc
            │               └── <themepkg>
            │                   ├── <ThemeName>Generator.kt
            │                   └── <ThemeName>.kt
            └── generator
                ├── GeneratorMain.kt
                ├── IntelijConfigurationGeneratorMain.kt
                └── ThemeSourceGenerator.kt
```

- `multisrc/src/main/java/eu/kanade/tachiyomi/multisrc/<themepkg>/<Theme>.kt` defines the the theme's default implementation.
- `multisrc/src/main/java/eu/kanade/tachiyomi/multisrc/<theme>/<Theme>Generator.kt` defines the the theme's generator class, this is similar to a `SourceFactory` class.
- `multisrc/overrides/<themepkg>/default/res` is the theme's default icons, if a source doesn't have overrides for `res`, then default icons will be used.
- `multisrc/overrides/<themepkg>/default/additional.gradle` defines additional gradle code, this will be copied at the end of all generated sources from this theme.
- `multisrc/overrides/<themepkg>/<sourcepkg>` contains overrides for a source that is defined inside the `<Theme>Generator.kt` class.
- `multisrc/overrides/<themepkg>/<sourcepkg>/src` contains source overrides.
- `multisrc/overrides/<themepkg>/<sourcepkg>/res` contains override for icons.
- `multisrc/overrides/<themepkg>/<sourcepkg>/additional.gradle` defines additional gradle code, this will be copied at the end of the generated gradle file below the theme's `additional.gradle`.
- `multisrc/overrides/<themepkg>/<sourcepkg>/AndroidManifest.xml` is copied as an override to the default `AndroidManifest.xml` generation if it exists.

> **Note**
>
> Files ending with `Gen.kt` (i.e. `multisrc/src/main/java/eu/kanade/tachiyomi/multisrc/<theme>/XxxGen.kt`)
> are considered helper files and won't be copied to generated sources.

### Scaffolding overrides
You can use this python script to generate scaffolds for source overrides. Put it inside `multisrc/overrides/<themepkg>/` as `scaffold.py`.
```python
import os, sys
from pathlib import Path

theme = Path(os.getcwd()).parts[-1]

print(f"Detected theme: {theme}")

if len(sys.argv) < 3:
    print("Must be called with a class name and lang, for Example 'python scaffold.py LeviatanScans en'")
    exit(-1)

source = sys.argv[1]
package = source.lower()
lang = sys.argv[2]

print(f"working on {source} with lang {lang}")

os.makedirs(f"{package}/src")
os.makedirs(f"{package}/res")

with open(f"{package}/src/{source}.kt", "w") as f:
    f.write(f"package eu.kanade.tachiyomi.extension.{lang}.{package}\n\n")
```

### Additional Notes
- Generated sources extension version code is calculated as `baseVersionCode + overrideVersionCode + multisrcLibraryVersion`.
    - Currently `multisrcLibraryVersion` is `0`
    - When a new source is added, it doesn't need to set `overrideVersionCode` as it's default is `0`.
    - For each time a source changes in a way that should the version increase, `overrideVersionCode` should be increased by one.
    - When a theme's default implementation changes, `baseVersionCode` should be increased, the initial value should be `1`.
    - For example, for a new theme with a new source, extention version code will be `0 + 0 + 1 = 1`.
- `IntelijConfigurationGeneratorMainKt` should be run on creating or removing a multisrc theme.
    - On removing a theme, you can manually remove the corresponding configuration in the `.run` folder instead.
    - Be careful if you're using sparse checkout. If other configurations are accidentally removed, `git add` the file you want and `git restore` the others. Another choice is to allow `/multisrc/src/main/java/eu/kanade/tachiyomi/multisrc/*` before running the generator.
