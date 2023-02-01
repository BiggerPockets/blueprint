---
author: Aaron Breckenridge
title: Introducing the Rails safe_stylesheet_pack_tag helper
description: When it works in development, it should work in production. But this isn't the case when you're using the Rails asset pipeline or webpacker. In this article, we introduce a helper that prevents asset issues from blowing up production.
excerpt: 
tags: rails engineering
---

Here at BiggerPockets Engineering, we pride ourselves on finding new and interesting ways to cause production outages at inopportune times. We recently discovered a new way to break production that we'd love to share with you. This bug comes courtesy of `stylesheet_pack_tag`'s different behavior in production vs development.

# A bit of background

Like many modern Rails applications, our application compiles different asset packs via webpacker. Webpacker is handy for compiling CSS and JS assets from NPM into packs that can be compiled by browsers. We came across a fantastic new bug after introducing our first CSS asset via an NPM package. Specifically: We added the Remix Icons package to a Webpacker JS pack. Here's how that's done:

```sh
# in your shell
yarn add remixicon
```

```js
/* in app/javascript/packs/application.js */
import 'remixicons/fonts/remixicon.css'
```

Once added to the application pack, the CSS can be included in a layout file via a call to `stylesheet_pack_tag`:

```erb
<!-- in app/views/layouts/application.html.erb -->
<%= stylesheet_pack_tag "application" %>
```

This works just fine in development environments: the icons show up. When compiling for production, this would normally result in a CSS stylesheet in the `public/packs` directory with a name like `application.signature.css`, whch is also picked up by the `stylesheet_pack_tag`. Unfortunately that's not how it works in production. Instead, webpacker was configured to split out vendor-suppied assets from non-vendor supplied assets, so the CSS ended up with a name like `application~vendor.signature.css` in production.

This brings us to the unexpected behavior that you may have experienced if you have spend any amound of time maintaining a large-scale Rails production application:

**If Rails cannot find an asset in production, it raises an exception.**



