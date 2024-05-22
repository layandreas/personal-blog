+++
author = "Andreas Lay"
title = "Hackerfluff: Building a Hackernews Reader with Flutter"
date = "2024-05-22"
description = "As a learning experience I built & released a Hackernews client app"
tags = ["mobile", "flutter"]
categories = ["Frontend"]
ShowToc = true
TocOpen = true
+++


## A Hackernews Client

In an attempt to learn some frontend / mobile development I've decided to give
[Flutter](https://flutter.dev/) a try. Flutter is cross-platform framework
developed by Google which lets us build & deploy for web (iOS & Android),
web and desktop from the same code base.

<!-- {{< appstore >}} -->

I'm an avid [Hacker News](https://news.ycombinator.com/) reader, they provide an
[API to query stories and comments](https://github.com/HackerNews/API), so I
decided to build a Hackernews client app.


## Web Version

Embedded below is the web-version of my app:

{{< iframe src="https://layandreas.github.io/hackerfluff-web/" frameborder="0" >}}

**Note:** This this is just a demo which might not be stable. For example this
web build is using an [experimental sqlite web implementation](https://pub.dev/packages/sqflite_common_ffi_web)
to store bookmarks and the history of seen comments. So far I've noticed some crashes
on Safari so I'd recommend using Chrome for this web demo.


## Why Flutter?

When it comes to cross-platform development there are currently two 
production-ready alternatives: Flutter and React Native.

My reasons for going with Flutter are simple:

- People who worked with it generally liked the framework and the development experience

- Although I know *a little* HTML, Javascript, CSS & React I'm not deep into to 
  Javascript ecosystem. I also don't particularly like CSS and my desire to
  really learn it isn't there

- Flutter comes with a rich set of customisable widgets. I'm happy not having to
  design widgets myself or having to choose from dozens of component libraries

- I own an iPhone and would like to use my app on it. However I don't feel
  inclined into investing heavily into Apple's walled garden ecosystem. With
  Flutter at least I'm not forced to use Xcode and am free to deploy an
  identical app on most platforms


## Getting Started with Flutter

For me the best way to get started with Flutter was the
[official get started guide](https://docs.flutter.dev/get-started/install),
including the associated [codelab](https://codelabs.developers.google.com/codelabs/flutter-codelab-first#0).

A few notes:

- Dart - the [language Flutter is using](https://dart.dev/guides) - looks quite 
  boilerplate heavy. Don't let that intimidate you: You won't have to write
  most of that yourself, if you're using `VS Code` then the official Dart and
  Flutter extensions will help you out with writing less. However I still think
  all that boilerplate can get in the way of learning as it can be hard to find
  the relevant logic in a code snippet, *even within tutorial code and toy examples*

- The official documentation is quite good. But I take some issues with it: 

  - For one, a lot of useful information is hidden in videos
  [like these](https://www.youtube.com/watch?v=WhVXkCFPmK4&embeds_referring_euri=https%3A%2F%2Fapi.flutter.dev%2F&source_ve_path=MjM4NTE&feature=emb_title).
  The videos aren't bad, but that information could often be transferred more
  efficiently as written text

  - Some information is outdated. For state management for example the docs refer to the
    [provider package](https://pub.dev/packages/provider), however after a little
    research you will find that [Riverpod](https://riverpod.dev/de/docs/introduction/getting_started)
    is its successor and reommended by its maintainer

- You won't need an external component library as Flutter ships with a lot of
  pre-built widgets. I'd recommend to start with the
  [material components](https://docs.flutter.dev/ui/widgets/material).
  They're quite customisable in case you want to deviate from the default
  material design. There is a also an
  [iOS-styled widget set](https://docs.flutter.dev/ui/widgets/cupertino)
  available, however I'd recommend the material design one as it appears to be
  more feature-rich and better maintained by the Flutter team.


## Reasons not to use Flutter

**Google might abandon it**

While I don't believe is going to happen anytime soon, my feeling is that any
product within Google currently has or will have to prove its value to the
company in `$$$`. While I don't know numbers I imagine that from a business
perspective it's hard to justify investing in Flutter.

**The web experience isn't great**

Performance of Flutter app on the web isn't that great so the user experience could be bad.
For a primarily web-based app it's probably best to used other state-of-the-art web tech
and go with something like [nextjs](https://nextjs.org/). However web performance
might be improved with the
[Flutter web WASM target](https://docs.flutter.dev/platform-integration/web/wasm).

**There are a lot of open issues on GitHub**

At the time of writing the 
[Flutter repository has 5000+ open issues](https://github.com/flutter/flutter/issues). 
I myself had to [open an issues as I experienced frame rate drops on iOS](https://github.com/flutter/flutter/issues/141913). 
That said I received a quick response to my issue - it was a known one - 
and I was able to fix it with the information given in the issue's thread. However
all that debugging is still taking time & effort that could go into building stuff.

**Releasing on the AppStore is not a very pleasant experience**

App reviews, guidelines and a not very user friendly
[app store connect homepage](https://appstoreconnect.apple.com/) make the process of
releasing an app to the iOS time consuming not quite hassle free.
Just writing some HTML & Javascript, dropping it on some cloud storage bucket
and your code can run anywhere is unbeatable from a developer point of view.

