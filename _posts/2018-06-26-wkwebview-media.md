---
layout: post
title: "Autoplaying video in WKWebView on iOS 10+"
---

I was looking into why videos wouldn't autoplay inside a WKWebView and couldn't find a comprehensive up-to-date answer online. In an attempt to write such an answer I wrote this post, describing what I just learned and how I fixed the issue.

Here's how you define a HTML video element that starts playing as soon as it comes into view, without becoming fullscreen:

```html
<video width="320" height="240" autoplay playsinline muted>
  <source src="movie.mp4" type="video/mp4">
</video>
```

This is exactly the way you would implement a much more efficient GIF replacement. A great explanation of the `autoplay`, `playsinline` and `muted` attributes and how Safari handles them can be found in [a webkit.org blogpost](https://webkit.org/blog/6784/new-video-policies-for-ios/).

The HTML snippet has the desired behavior in Mobile Safari, but when loading the same page inside a WKWebView, the video will not play. The default configuration of WKWebView is more restrictive and we need to explicitly allow this.

There's two things we need to do: 
- Set `allowsInlineMediaPlayback` to `true` to allow media to play inline. If you don't do this, a fullscreen player will take over the screen as soon as the video starts playing, which should be immediately.
- Set `mediaTypesRequiringUserActionForPlayback` to a value that excludes video, e.g. `.audio` or `[]`. If you don't do this, the user will need to tap on the play button in the video to start it. Since we do not declare the `control` attribute on the video element, the play button is not visible and the video is basically unplayable.

In code, this looks as follows:

```swift
let configuration = WKWebViewConfiguration()
configuration.allowsInlineMediaPlayback = true
configuration.mediaTypesRequiringUserActionForPlayback = .audio
let webView = WKWebView(frame: .zero, configuration: configuration)
```

Make sure to change the configuration before passing it to WKWebView. I first tried to change the configuration through the webview (`webView.configuration.allowsInlineMediaPlayback = true`), but that silently failed.
