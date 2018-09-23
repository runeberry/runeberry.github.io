---
layout: blog-post
title: Site redesign
---

I spent most of today playing around with the site, so here's what I accomplished:

* Custom URL in place - [runeberry.com](https://runeberry.com) is now streaming _directly_ to your browser!
* SSL certificate is in place, so your credit card info is safe with me.
  * Actually, [GitHub Pages manages this for free](https://blog.github.com/2018-05-01-github-pages-custom-domains-https/) and it's super-easy. But it's worthy of a bullet point for the hours that I spent getting an SSL cert, generating a CSR, fiddling with redirects, and then realizing I didn't need to do any of that.
* Moved the blog posts to a `blog/` subdirectory, and added a lil' navbar to the top of the post template.
* Tweaked the default theme for jekyll ([minima](https://github.com/jekyll/minima)) to be an equivalent dark theme. I can't expect anyone to read anything without a dark theme, right?

So what's next? Actually working on the [game engine]({% post_url 2018-09-14-Ecosystem %}) that I started talking about like a week ago, rather than A/B testing the color palette of my blog. Hopefully more on that tomorrow!