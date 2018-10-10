---
layout: blog-post
title: Open source
---
A couple of years ago, I threw together a quick plugin for [Wox](https://github.com/Wox-launcher/Wox), a Spotlight search equivalent for Windows, that would let me search the [Old School RuneScape wiki](https://oldschool.runescape.wiki/) from my desktop. I found myself frequently doing this:

1. Click away from the game and onto the browser
2. Open a new tab in Chrome
3. Search for something OSRS-related via Google
4. Look for the wiki result (usually first, but no always)
5. Click to open the page

It doesn't sound like much, but with the plugin I'm able to shorten the process down to this:

1. From anywhere, type Alt+Space+"osw"+my search
2. Hit Enter if it's the first result
3. Press down a couple of times and hit Enter if it's not the first result

The whole process goes from, say, 10-20 seconds per search to about 2-3 seconds per search. Not huge, but when you're doing these kinds of searches several times per day, it adds up. It was a nice little plugin that I used a lot, but I never advertised it (except for [right here right now](https://github.com/dolphinspired/Wox.Plugin.RuneScapeWiki)). Although the plugin is publicly available, I still don't know if anyone is using it other than me.

Just recently, the wiki team [decided to move their wikis](https://runescape.wiki/w/Forum:Leaving_Wikia) (RS and OSRS) to new sites, with excellent reasons outlined in the linked post. Even though I don't play the game currently (I won't say "anymore", because nobody ever really quits), I wanted to keep my plugin up to date. If for no other purpose than peace of mind, so that there's not completely broken software out there with my name on it.

I soon discovered that the update was a little more involved than just switching URLs. The original wiki was powered by Wikia, and the new one by MediaWiki, each of which has wildly different APIs. At first, I couldn't figure out how to translate my search over to the MediaWiki API, so I resorted to just scraping the search results page and grabbing links, titles, and snippets off the page. It worked well enough, but I knew it was more prone to breakage than I would like. I reached out to the [wiki admins](https://weirdgloop.org/) just to share the plugin, and their response inadvertently helped point me in the right direction. The API I needed was there, I just totally missed it - if anyone else is lost on this, you want [action=query](https://www.mediawiki.org/wiki/API:Query), which seems so obvious in retrospect.

Since I'm playing around in the code anyway, I'm trying to add a few features. One thing that I have roughly working now is that thumbnails will load next to each article, rather than the same generic icon for each search result. That's lead to some interesting experiments with image caching and running background Tasks to download the images.

One tangiential thing it's got me thinking about is open source. I personally enjoy Wox and use it every day, but the state of the software looks... well, not so great. I wanted to share my plugin with a broader audience (namely, RuneScape players) by pointing out that Wox also has Twitch and YouTube plugins. However, I downloaded each of these plugins to try them out and they are _straight up broken_ - and these are the [two most popular plugins](http://www.wox.one/plugin) for Wox, by far. The YouTube one never returns any results and the Twitch one doesn't even install properly.

Since this is all open-sourced, I took a crack at [debugging and fixing the Twitch plugin](https://github.com/Eligioo/WoxTwitch/pull/2). The owner of the repo stated in the only issue on the repo that he wasn't working on Wox anymore, but would merge in any pull request to fix it, so maybe soon we can bring this one back to life. The YouTube one, however, [doesn't look as promising](https://github.com/suteudragos/YoutubeQuery). The code hasn't been touched in 2 years, none of the issues have been responded to, and the author isn't very active on GitHub. I thought about pulling it down to investigate it, but what are the odds that a successful fix would even get merged? I wouldn't bet money on it.

The problem with any plugin fix, though, is that the original author still needs to upload the fixed plugin to the Wox website in order for existing users to get the fix. This process isn't documented anywhere, but it boils down to adding all the parts (.dll binaries or .py scripts, plugin.json, and images) to a `.zip` archive and changing the extension to `.wox`. That's why I included a pre-packaged `.wox` file with that PR linked above - I figure making the process easier might increase the chance that a fix actually gets published.

So the plugin scene for Wox doesn't look stellar, but what about Wox itself? I was sad to find that it's not looking so hot, either. The last release was in February, and there [haven't been any significant changes](https://github.com/Wox-launcher/Wox/commits/master) since then. The lead maintainer (I assume?) posted a note citing a [2-month leave of absence](https://github.com/Wox-launcher/Wox/commit/8a0f80181e92c9e71a687d27df53d6cd9649b633) which was only withdrawn [after 10 months](https://github.com/Wox-launcher/Wox/commit/553a6e8ff6f1bd8e31c7c06d1cac4c73ba0af3f1), also in February. It's been radio silence since then. Not to mention there are over 500 issues and 13 pull requests piling up. Documentation is still very sparse - at least, what's available in English isn't very helpful.

Anyway, maybe this is little more than a forensic exercise to figure out what happened. Is the project dead? Who pronounces it so? At least one developer in the issues is sounding the death knell and suggesting everyone move to a new project. If we move on to the [next big thing](https://github.com/oliverschwendener/ueli), how do we help ensure the same thing doesn't happen again? Or do we expect it to happen, and just roll with it? What makes a successful open-sourced project? Now _that_ sounds like something I need to google and read up on.

Wox had a pretty big spike in popularity at some point, possibly around the release of Windows 10 or one of its infamous breaking updates. But it never attained the critical mass it needed to become self-sustaining. Would a larger community of administrators have helped it stay afloat? Would better documentation have helped draw in more developers to make a more attractive plugin ecosystem? Maybe a larger audience than just Windows users? Whatever could have been or could yet be, I think I'll be saying goodbye to this particular project soon - of course, after quietly leaving a working and stable contribution in my wake.