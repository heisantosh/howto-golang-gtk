# Using Go to build GTK+ applications

This is my experience so far in the journey of building GTK+ apps using the Go programming language. I'll be sharing mostly Go specific or Go related things here.

I'm going to use my project at [`https://github.com/mubitosh/qrshare`](https://github.com/mubitosh/qrshare) as the reference. It is an app for sharing file using QR Code. It is available in the AppCenter with the name `QR Share`.

Below is the project layout.
Pruned output of `tree -L 4` in the project root directory- 

```bash
.
├── data
│   ├── com.github.mubitosh.qrshare.appdata.xml
│   ├── com.github.mubitosh.qrshare.contract
│   ├── com.github.mubitosh.qrshare.desktop
│   ├── icons
│   │   ├── 128
│   │   │   └── com.github.mubitosh.qrshare.png
│   │   ├── 32
│   │   │   └── com.github.mubitosh.qrshare.png
│   │   ├── 48
│   │   │   └── com.github.mubitosh.qrshare.png
│   │   └── 64
│   │       └── com.github.mubitosh.qrshare.png
│   ├── meson.build
│   ├── screenshot-app.png
│   └── screenshot-qr-window.png
├── debian
│   ├── changelog
│   ├── compat
│   ├── control
│   ├── copyright
│   ├── rules
│   └── source
│       └── format
├── LICENSE
├── meson
│   └── build.sh
├── meson.build
├── po
│   ├── com.github.mubitosh.qrshare.pot
│   └── de.po
├── README.md
├── src
│   └── github.com
│       └── mubitosh
│           └── qrshare
└── vendor
    └── src
        └── github.com
            ├── boombuler
            ├── gosexy
            └── gotk3
```


## Go bindings for GTK+

GTK+ is written in C. As such bindings exist for various languages including Go.

There are couple of Go bindings for GTK+. The ones I found were -
 * [https://github.com/gotk3/gotk3](https://github.com/gotk3/gotk3)
 * [https://github.com/mattn/go-gtk](https://github.com/mattn/go-gtk)

`gotk3` is GTK+3 binding while `go-gtk` GTK+2 binding.

I'm interested in using GTK+3, so I'm using `gotk3`.

## Build tools

We have popular ones like CMake and meson. I decided to go with meson as it I find it more comfortable. 

### Meson

Unfortunately, there is no meson project profiles built in for Go. But that's fine. Below is the top level `meson.build` from my project [https://github.com/mubitosh/qrshare](https://github.com/mubitosh/qrshare)

```
project('com.github.mubitosh.qrshare')

run_target('build',
	command: 'meson/build.sh')

subdir('data')
```

and `meson/build.sh`

```bash
#!/bin/bash

appid=com.github.mubitosh.qrshare

# Build the app
gb build -tags gtk_3_18 all
mv bin/qrshare-gtk_3_18 bin/${appid}

# Generate translations for the app
for po in $(ls po/*.po)
do
    name=$(basename $po)
    lang=${name%.*}
    mo=po/locale/${lang}/LC_MESSAGES/${appid}.mo
    mkdir -p po/locale/${lang}/LC_MESSAGES/
    msgfmt -c -v -o $mo $po
done
```

Generally, a language name is passed as the second argument to `project()`. Since meson doesn't recognise Go, only the project name is passed. Before meson version `0.40`, a second argument was mandatory. As of this writing I'm using meson `0.42` and passing a single argument also works fine.

For actually compiling the Go files I used a shell script to do the compilation. It is then provided as a target int the build file. So when `ninja build` is invoked the compilation script will be executed.

### Managing Go dependencies with [GB](https://getgb.io/)

I have tried 3 options-
 * usual `go get`
 * the brand new [`dep`](https://github.com/golang/dep)
 * [`gb`](https://getgb.io/)

The issue with go get was - while building in AppCenter it was unable to access internet. This means the required Go packages are not downloaded. So the app couldn't be built.
Later on, in the gitter appcenter channel `btkostner` mentioned that now network can be accessed during build time now. So this might work now but I haven't tried to  onfirm.

I was really looking forward to using `dep`. In this case there were two limitations - go 1.7 or above was required, `dep` needs network access to download dependencies. As of this writing go 1.6 is available in the default repo. Also there is no package for `dep` to install from the ubuntu repo. Option is to either include the `dep` binary in the project or do a `go get` to install. Both options doesn't look great.

`gb` is the one that worked out really well. I highly recommend to go through the documentation and little examples in the website https://getgb.io. Basically, with gb there is a copy of the dependency inside the project itself. This eliminates the need to download packages during build time. Also there is a package available for `gb` in elementary/Ubuntu repos. So, it can be specified as a build dependency in the debian/control file.

```
Source: com.github.mubitosh.qrshare
Section: x11
Priority: extra
Maintainer: Santosh Heigrujam <santosh.hei@gmail.com>
Build-Depends: debhelper (>= 9),
               libgtk-3-dev,
               meson,
               golang-go,
               gb
Standards-Version: 3.9.3

Package: com.github.mubitosh.qrshare
Architecture: any
Depends: ${misc:Depends}, ${shlibs:Depends}
Description: Share files using QR code
 Share by using the option in the right click menu. Scan the QR code image to get the shared file.
```

Now I just have to add `gb build -tags gtk_3_18 all` in the `meson/build.sh` script for `gb` to build the project.

## Translations

### A brief overview

For those unfamiliar with enabling app translations in GTK+ apps, here is a brief overview.

There are three types of files that comes into picture while working on i18n of the app
 * .pot
 * .po
 * .mo

`.pot` file is like a template. It contains all the strings to be translated and an empty place for the resulting new string.

Below is a part of `com.github.mubitosh.qrshare.pot` file from my `qrshare` app.

```
#: src/github.com/mubitosh/qrshare/main_window.go:90
msgid   "Select a file to share"
msgstr  ""
```

`.po` files are a copy of `.pot` file and specific to each language. It contains the actual translated strings.

Below is the corresponding entry in `de.po` file for German language.

```
#: src/github.com/mubitosh/qrshare/main_window.go:90
msgid "Select a file to share"
msgstr "Wählen Sie eine zu freigebende Datei aus"

```

`.mo` files are compiled from `.po` files and are meant to be read by the computer. When the app is installed the `.mo` files are also installed at the location `/usr/share/locale/${lang}/LC_MESSAGES/`, for the various languages `${lang}` that the app supports. The app uses the `.mo` files for translation while running with supported languages.

To get the `.pot`, `.po`, `.mo` files, there are tools we can use. The general steps are -
 1. Use a library/package in the app to enable translation
 2. Extract strings from the source files and create a .pot file
 3. Copy .pot file and create a .po file
 4. Update .po file with the translations
 5. Compile .mo files from .po files
 6. App detects .mo files and uses the for the corresponding language setting

 We'll take a look at the tools I used for my Go project afterwards.

### 1. Go package for enabling translation

To enable translations strings in the app are passed to some library/package function. For example, for `Vala` apps we can see something like `_("Hello world!")`. `_()` is a macro for using `gettext()` function.

For my Go project I'm using [`github.com/gosexy/gettext`](https://github.com/gosexy/gettext), which is a binding for `GNU gettext`.

Below is the content of `https://github.com/mubitosh/qrshare/blob/master/src/github.com/mubitosh/qrshare/i18n.go`

```go
package main

import (
	"path/filepath"

	"github.com/gosexy/gettext"
)

func initI18n() {
	gettext.SetLocale(gettext.LC_ALL, "")
	gettext.BindTextdomain(appID, filepath.Join("/usr/share", "locale"))
	gettext.BindTextdomainCodeset(appID, "UTF-8")
	gettext.Textdomain(appID)
}

// T returns the value of gettext.Gettext.
// It is a shorthand of using gettext.Gettext function.
func T(s string) string {
	return gettext.Gettext(s)
}
```

And an example usage in `https://github.com/mubitosh/qrshare/blob/master/src/github.com/mubitosh/qrshare/main_window.go#L10`

Here I'm using `T()` function as a short form of `gettext.Gettext()` function.

```go
...

// mainWindowNew returns Granite Welcome screen style window.
func mainWindowNew(qrshare *QrShare) *gtk.ApplicationWindow {
	titleLabel, _ := gtk.LabelNew(T("Share a file with QR Share"))
	styleCtx, _ := titleLabel.GetStyleContext()
	styleCtx.AddClass("h1")
	titleLabel.SetJustify(gtk.JUSTIFY_CENTER)
	titleLabel.SetHExpand(true)

	subtitleLabel, _ := gtk.LabelNew(T("Use any of the options below to share\nScan the QR code to download the file"))
	styleCtx, _ = subtitleLabel.GetStyleContext()
	styleCtx.AddClass("h2")
	styleCtx.AddClass("dim-label")

...
```

### 2. Extracting strings into .pot file

For generating the `.pot` file, I used the tool `go-xgettext`. It can be installed as-

`go get github.com/gosexy/gettext/go-xgettext`

At the project root directory, I used the below command to get the `.pot` file-

`go-xgettext -o po/com.github.mubitosh.qrshare.pot --package-name=com.github.mubitosh.qrshare -k=T src/github.com/mubitosh/qrshare/*.go`

`T` is the shorthand I used for function `gettext.Gettext()`

### 3,4. Creating .po file for German language

`msginit` can be used to create `de.po` from `com.github.mubitosh.qrshare.pot`. This prompts for the email address to be used to identify the translator.

```bash
msginit -l de -o de.po -i com.github.mubitosh.qrshsare.pot
```

Now, the translations can be simply updated in `de.po`. A catch here - the content type in `de.po` should `UTF-8` as:

`"Content-Type: text/plain; charset=UTF-8\n"`

### 5. Creating the .mo file

In my `meson/build.sh` script I have the following lines - 

```bash
# Generate translations for the app
for po in $(ls po/*.po)
do
    name=$(basename $po)
    lang=${name%.*}
    mo=po/locale/${lang}/LC_MESSAGES/${appid}.mo
    mkdir -p po/locale/${lang}/LC_MESSAGES/
    msgfmt -c -v -o $mo $po
done
```

This looks for the `.po` files in the `po` directory in compiles them into `.mo` files using the tool `msgfmt`. And the `.mo` files are placed into the expected layout under `/usr/share` directory. While installing the app we just copy the `locale` directory into under `/usr/share`.

## Integration with debian

To run the script `meson/build.sh` the custom target `build` is invoked - line 18 in `debian/rules`.

Since I'm not using any integrated tools, the following files need to be copied into the correct locations-
 1. the app binary `com.github.mubitosh.qrshare`
 2. the locale directory containing the `.mo` files

This is done in the `debian/rules` file at lines numbers 25, 26 respectively

```bash
     1	#!/usr/bin/make -f
     2	# -*- makefile -*-
     3	# Sample debian/rules that uses debhelper.
     4	# This file was originally written by Joey Hess and Craig Small.
     5	# As a special exception, when this file is copied by dh-make into a
     6	# dh-make output file, you may use that output file without restriction.
     7	# This special exception was added by Craig Small in version 0.37 of dh-make.
       
     8	# Uncomment this to turn on verbose mode.
     9	#export DH_VERBOSE=1
       
    10	%:
    11		dh $@
       
    12	override_dh_auto_clean:
    13		rm -rf debian/build
       
    14	override_dh_auto_configure:
    15		mkdir -p debian/build
    16		cd debian/build && meson --prefix=/usr ../..
       
    17	override_dh_auto_build:
    18		cd debian/build && ninja -v && ninja build
    19		
    20	override_dh_auto_test:
    21		cd debian/build && ninja test
       
    22	override_dh_auto_install:
    23		cd debian/build && DESTDIR=${CURDIR}/debian/com.github.mubitosh.qrshare ninja install
    24		mkdir -p debian/com.github.mubitosh.qrshare/usr/bin
    25		cp bin/com.github.mubitosh.qrshare debian/com.github.mubitosh.qrshare/usr/bin/
    26		cp -R po/locale debian/com.github.mubitosh.qrshare/usr/share/

```

## Downsides

* The resulting app binary is a big.
* Have to stick with the version of Go available in the Ubuntu repo (1.6 for elementary Loki). Though `btkostner` mentioned go 1.9 is available in the AppCenter build environment.
* May have to write/generate bindings with C only libraries or with some GTK+ features like G Resources.

## Upsides :)

* I get to use Go.
* All the popular Go packages can be used, especially the network related ones
* A new experience building GTK+ apps with Go finding out what works and what doesn't work. And sharing those experiences.
