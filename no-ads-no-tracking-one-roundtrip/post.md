When I set out to create this blog several years ago, I had a couple of goals in mind in terms of user experience.

These goals were:

* No Ads, ever
* No Tracking, ever
* Minimal Footprint 

The first and second are rather self-explanatory - users who read my blog should not be exposed to advertising and/or tracking services. The web nowadays is crawling with, in my opinion, unethical and often illegal tracking mechanisms. There has been EU wide regulation in this regard (GDPR), but the general trend of tracking as much as possible doesn't seem to have shifted so far.

This lack of a shift is due to a huge part of the web running an advertising-based business model, which relies on tracking users across different sites, collecting as much information as possible in order to improve ad targeting.

This unending hunger for users' personal data, paired with a business model reliant upon targeted advertising makes for a dangerous, self-reinforcing cycle.

The third part, Minimal Footprint in this case means that, in terms of performance, the whole blog should, in terms of byte size, be as small as possible.

And just to be clear, I'm not talking about less than 1MB here, I'm talking about fitting the whole frontpage within 10KB. This is rather specific, but the more general goal would be to make the page as small as possible, reducing all unnecessary stuff, improving every reader's experience in the process.

We'll talk about the Why and the How of making this happen for a website. This will not be an especially technical, or exhaustive description of what to do, but more a write-up of the reasoning and some direction for people interested in going down the same road.

## Why

First off, let's talk about why these goals were made and why they're important to me. I'm very critical of the role advertising plays in modern society and especially when it comes to the web. It's undeniable, that advertising has been the motor behind the internet's rapid growth and enabled many sites to exist and flourish at a time when there were very few other business models available.

However, the amount of ads one is blasted with nowadays while browsing the web is crazy. There's a reason several browsers have already started blocking ads and trackers by default. The advertising industry, and this is related to the topic of "No Tracking", relies on collecting as much data as possible to target ads. This mass collection of personal data is dangerous and very problematic when a government agency trying to protect citizens does it, but is completely unacceptable when it's done for profit by private companies.

Additionally, there is by now plenty of evidence for the adverse effect advertising has on people's well-being, so much so, that several cities such as Sao Paolo banned outdoor advertising completely.

So, with these things in mind and the fact that I luckily don't depend on generating any income with my blog, I don't think it would be justifiable to harass my readers with advertising.

The next goal on the list is to NOT use trackers, or track users in any way. Data privacy is a human right. Unfortunately, in lockstep to monetizing the web using advertising, collecting and selling people's data for profit has been a lucrative business model for some time. Many people don't seem to question why a service which costs millions to maintain would be free to use.

This is one of the fundamental underlying problems of monetization on the web in my opinion - people are not willing to pay for basic services and don't seem to have a good understanding what creating and maintaining these services cost. This, paired with some ignorance regarding data privacy in the form of `I have nothing to hide`, leads to a lot of data sharing, both knowingly and even more unknowingly.

The point about the `minimal footprint` is less "controversial" ;). The posts I create are mostly text, code and the odd image here, or there. This means that there is really no reason for the whole website to be bigger than a few kb and it should be possible for the front page to fit into the first couple TCP roundtrips (after TLS). This makes the reading and browsing experience for every visitor better and additionally makes the site usable in places with bad/unstable internet connections.

An added benefit is, that even for users with good internet connections, the page loads blazingly fast and doesn't need a lot of bandwidth, or energy to render (relevant for mobile devices).

## The How

Well, not including any ads, or trackers is actually pretty easy - just don't do it. ;) In fact, my blog doesn't include a single line of JavaScript, which helps with page size as well. The only "tracking" information I'im interested in, is how many people read a post relative to others.

And I also don't care as much about the exact number, but rather about what people like to read / find on the web related to my other posts. For this purpose, the server log of whereever the blog is hosted should be sufficient.

Getting down the request size is a bit more challenging, but also not particularly difficult. The first step is to reduce the amount of images on the frontpage to a minimum. A small logo and icons, optimally svg, are fine, but everything above that will make it hard to stay under the limit.

No JavaScript also helps, but might be hard to achieve depending on the content you create. For me, I write primarily posts with code-snippets and very rarely an image, so I don't need any JS on the site. Should you need JavaScript for the kind of content you're creating, make sure it's a minified and minimal version of what you need. Try not to include third-party scripts, if you include JavaScript it should be small enough to inline it in the page.

In any case, everything you send down the wire should be compressed (e.g. gzipped), which will save you a lot of page size. Inlining assets such as small images and CSS might also help, as it will lead to less requests being necessary to load the page.

## Conclusion

In this post I attempted to explain my reasoning  behind creating a blog, which loads the minimal amount of data from the network, isn't plastered with ads and respects the privacy of it's readers.

The web is a beautiful thing and has enriched my life in so many ways, I believe that the current situation with unnecesarrily bloated websites, which try to extract as much personal and behavioral data from users as possible in order to blast them with "better" advertising is not a sustainable model.

