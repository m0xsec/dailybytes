---
title: "Webmentions"
date: 2021-12-03T15:44:48-08:00
draft: false
hideSummary: false
showToc: true
tags: ["indieweb", "indieauth", "PGP", "webmention"]
categories: ["blog"]
cover:
    image: "/posts/webmentions/images/cover.png" # image path/url
    alt: "webmention logo" # alt text
    caption: "Webmentions, a fundamental component of IndieWeb" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

I've been looking at adopting more and more [IndieWeb principles](https://indieweb.org/principles) as I get my blog setup. Today I took the time getting Webmentions setup, and wow it was a lot of fun! I wanted to write a bit about the process I went through to get it all setup and what I plan to do next. Here we go!

<!-- more -->

# IndieAuth

{{< notice note>}}
IndieAuth is in the process of [transitioning](https://indieweb.org/IndieAuth-brainstorming#naming_confusion) to IndieLogin. 
{{< /notice >}}

The first part of this process was to get [IndieAuth](https://indieauth.com/) working. I won't go into too much detail about what IndieAuth is or how it works, but basically it is an IndieWeb approach to OAuth that allows you to identify yourself by your [domain name](https://indieweb.org/RelMeAuth). Third-party OAuth providers are used to authenticate your identity, or you can use a PGP key.

This works by placing the `rel="me"` attribute on profile links, such as Twitter or Github.

```html
<a href="https://twitter.com/USERNAME" rel="me">@USERNAME on Twitter</a>
<a href="https://github.com/USERNAME" rel="me">Github</a>
```

Something important to note is that in order for this to work, the profile that you are linking to must have a link back to your website. This is pretty easy to do on Twitter and Github as you just put your website in your profile bio. I discovered that Keybase also uses `rel="me"` links on your profile, however IndieAuth does not support identity authorization with Keybase since they aren't an OAuth provider.

If you're like me, you might be a little put off by having to use a service like Twitter or Github just to get IndieAuth working. After all, the whole point is to be able to login as "you". This is where using a PGP comes in! You use a similar `rel` attribute on a link to your public PGP key, though it looks like this:

```html
<link rel="pgpkey" href="/key.pub">
```

If you use this method, you will be asked to sign a randomly generated string using your PGP key, then provide the signed string.


{{< notice info >}}
There is currently an open [issue](https://github.com/aaronpk/indielogin.com/issues/44) for IndieLogin not supporting ed25519 PGP keys.
{{< /notice >}}


# Receiving Webmentions

I decided to use [Webmention.io](https://webmention.io/) as my webmentions endpoint. It was made by [Aaron Parecki](https://aaronparecki.com/), much like a lot of core IndieWeb services (thanks btw, Aaron!). I considered rolling my own implementation using Go or something, though for now this works very well. 

Setting it up is easy once you have IndieAuth working. All you have to do is advertise your webmention endpoint, which looks something like:

```html
<link rel="webmention" href="https://webmention.io/example.com/webmention" />
```

This will handle all incoming webmentions and will display them on a nice dashboard on their website. Though webmentions are way better when you display them on your website!

# Displaying Webmentions

 When looking into how to do this, I came across [fluffy](http://beesbuzz.biz/)’s incredible [webmentions.js](https://github.com/PlaidWeb/webmention.js) library. Getting it setup was also very straight forward:

1. Include the JS library somewhere on your website:
   
   ```html
   <script src="/js/webmention.min.js" async></script>
   ```

2. Place the following wherever you wish to render your webmentions:
   
   ```html
   <div id="webmentions"></div>
   ```

You can also add some custom CSS to adjust how everything renders. Check out the [Github Repo](https://github.com/PlaidWeb/webmention.js) for information on how to do that.

# What Next?

The next thing I would like to do is to automate sending webmentions when I write a new blog post. Right now I have to send them out manually which is a bit tedious. I am thinking I will write some sort of bash script that runs as a [Git Hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) whenever I push changes to my blog's repo. I may also leverage [Telegraph](https://telegraph.p3k.io/) in someway.

When I get that figured out I will be sure to share!


`<3 m0x`