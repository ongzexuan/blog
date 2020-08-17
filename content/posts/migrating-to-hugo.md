---
title: "Migrating from Ghost to Hugo"
date: 2020-08-17T00:49:17-04:00
draft: false
feature_image: /migrating-to-hugo/feature_image.png
summary: "Why not Git commit my blog?"
---

I don't really do front end, websites are kind of beyond me. I like to make things that spit out plaintext (like... APIs). My blog has been running on a Ghost installation on a Digital Ocean droplet for the last two years. It's probably the most lightweight blog CMS that I've seen so far (I never want to see a Wordpress site again). One feature that it doesn't have is a way to manage the content like I manage code on a regular basis. I suppose I could version control the markdown documents, and have some mechanism for managing the images, and ensuring the links don't get broken, but then there'd be two sources of truth to quote a [friend of mine](https://yish.sg/post/hello-world/). I was thus introduced to an alternative called [Hugo](https://gohugo.io/) where I could do just that, committing my blog content to GitHub.

Since it was going to be a set of static pages, I figured I could also migrate my blog from Digital Ocean to GitHub pages, and save 5 bucks a month.

### Unpaid Plug for Digital Ocean

Since we are saying bon voyage to the Digital Ocean, I must say that DO has been great as far as hosting is concerned. Where else can you get a reasonably reliable VPS for 5 bucks a month that doesn't suck or sound shady? In a quest to outdo that price bar I have rented random servers in Texas from Low End Box, doing well at around 3.5 bucks, but then I always run out of RAM when running Python scripts on it, so productivity gets poured into reducing memory footprint, and scheduling jobs so they don't consume all the RAM and crash all of Python.

Reflecting on the price, at this point in my life, 5 bucks a month is pretty affordable. Back in high school, I remember frustrating over choosing multiple options whose difference in price per month was in the order of 50 cents. I am happy and grateful that I can shell out 5 bucks a month for a server today. GitHub also offers hosting of static pages for free. What a time to be alive.

I guess that wasn't so much of a plug.

## Content Management Workflow

With Hugo, the workflow for managing content is greatly simplified. First I create a new markdown document in the `posts` folder, and then I most probably create a new corresponding folder with the same name in the `static` directory to host images. That's pretty much it actually.

I have two separate GitHub repos setup for this, the [first repo](https://github.com/ongzexuan/blog) holds the entire source code of the blog. The [second repo](https://github.com/ongzexuan/ongzexuan.github.io), the actual GitHub page, holds the rendered pages (the `public` folder of Hugo). To push content therefore, just requires committing to both repos, which can be done with a simple script I borrowed from Hugo.

## Theme

I quite liked the original Ghost theme I had (Casper 2). It was simple and provided the right balance for the strange mix of life choices I have (there's the code, the plants, the food, the occasional photo). A nice person ported the [Casper 3 theme over to Hugo](https://github.com/jonathanjanssens/hugo-casper3), so I just manually ported over the small number of posts I had. I was in particular annoyed at the fact that the theme was in black, and played around with the CSS settings before realizing that I had dark mode enabled.

There were a number of things that didn't work out of the box and were not particularly well documented.

#### Images

Adding images can be done using a shortcode as follows (compress the handlebars):

```
{ { < figure src="/malmo-maze-solver/moving_small.gif" caption="Bot repeatedly falling into the lava." > } }
```


#### Github Gists

Adding gists is a bit more complicated. First add a shortcode under `layouts > shortcodes > gist.html`. In that file specify the JS.

```
<script type="text/javascript" src="http://gist.github.com/{{ .Get 0 }}.js"></script>
```

Then you can add gists using the shortcode with a gist id:

```
{ {<gist 07415be703d5576f9433ddb0730f7766>} }
```

The gist CSS layout does not play well with dark mode. It looks really ugly. I overwrote that with some custom CSS I borrowed from a source I lost, which I put under the `static` folder and import in my site header:

{{<gist c77799e2f1f54569c034a9ceeca85e3e>}}

#### Front Matter

Post metadata, or what Hugo calls front matter, are not particularly well documented. The most useful front matter are, after some digging:

- `feature_image` - for addding a featured image to the post head, and on the index page.
- `summary` - adds a summary that overrides the badly generated default summary, to be displayed on the index page.
- `tags` - adds a list of tags, but the default CSS allows for only the first tag to be displayed. I can live with that for now.

To add featured images to

#### Author Page

The theme doesn't seem to have a default author page that works. To be honest there's only one author here so really I just need another page that's not a post. I can't seem to find a way to do that so I created the author taxonomy (another Hugo term). This required me to add my author metadata (i.e. my metadata) under `content > authors > zexuano > _index.md`. I also had to create a design for this page under `layouts > authors > list.html`. You may also need to create an empty folder in `authors` with your author name.

#### Other Annoying Fixes

Some features were not provided out of the box so I hardcoded some items into the theme itself. This includes adding a favicon to the page, changing the site title to a static one instead of a dynamically generated one per post, and adding icons for post authors. If you'd like to know how I fixed something, you can take a look directly at the [source code of the blog](https://github.com/ongzexuan/blog) and do a diff, or just ask me.

#### Before and After

There are a number of small but nice changes to the layout which I am quite pleased with.

{{<figure src="/migrating-to-hugo/old.png" caption="Old layout, with obnoxiously sized title, a non-recognizable image icon, and a date that's kind of in a strange place, with the summary enlarged for some reason." >}}

{{<figure src="/migrating-to-hugo/new.png" caption="New layout, with obnoxiously sized title, text in the nav bar, nice looking date with an author icon, and no enlarged summary." >}}

## Closing Thoughts

I believe this is as close as I'm going to get in reducing the inertia of writing new posts caused by friction between typing something and pushing it out there. The changes I've made relative to the old Ghost instance also make it much easier to write book reviews, a task for which has been left in the back burner for some time. Expect to see some in the future!

