title: The Minimum Viable Product
date: 2022-01-14
tags: tech

This image has been circulating on LinkedIn as a tongue and cheek
example of a miminum viable product. Look at the googly eyes, and if
they're moving, you're probably in an Earthquake.

<img src="earthquake-detection-kit.jpeg" width="272" height="363">

Of course, at least one of the resposnes to this meme on LinkedIn was
a description of how it's not an MVP. To be an MVP, it would need 24/7
monitoring or a video camera with a motion alarm. This is presumably
to avoid missing earthquakes that happen off hours or when you're away
from the detector. The trouble with this is that it assumes a product
requirement that may not exist. If you don't care about earthquakes
that happen when you're away, you don't need any of the extra
monitoring. The product is viable as it is. To know whether or not
something is a minimum viable product, you first must understand what
that product is required to do to be viable.

Assuming this detector actually works, the one time I've been in an
[earthquake](https://en.wikipedia.org/wiki/2011_Virginia_earthquake),
it might have been useful to have around. Being a transplant to the
Northeast from Texas, I had no idea in practice what an earthquake was
like or even to expect that it might be a possibility. When it
happened, none of us knew what was going on, and I remember my
co-workers and I walking over to a wall of exterior windows. It's hard
to imagine a worse response to an earthquake, but if we'd realized
what was happening, we'd have realized what not to do. A wall mounted
detector marked 'Earthquake' might well have helped us avoid the
mistake. This experience makes me sympathetic to the notion that this
detection kit has its uses as designed. It can be viable as designed
because it satisfies the requirement of immediate real-time detection.

This viability, though, is contingent on the fact that there was no
need to know about earthquakes that occurred off-hours. Add that
requirement in, and more capability is needed. Understanding the
viablity of a product therefore requires understanding how it will be
used and what the expectations of its users. The power of the MVP is
that it forces you to develop that understanding. Getting to an MVP is
less about the product and more about the requirements that drive the
creation of that product.

In a field like technology, where practicioners are often attracted to
the technology itself, this distinction can be easy to
miss. Personally, I came into this field because I like building
things. It's fun. There's a great personal satisfaction that derives
from envisioning a system, and then building it out. The tension in
the concept of the MVP for this mindset is that it forces you to
consider whether or not it's useful to build the thing at all.  The
notion of the MVP inherently pulls you away from the act of building
and forces you to to consider that there may be no value in the thing
you aim to build.

Take this back to the earthquake detection kit, and make the
assumption that you don't really need 24/7 monitoring. Real time
earthquake detection in the same room is sufficient. It follows from
this that automatic detection via a video camera doesn't need to be
built. Something that may have been exciting for a practitioner to
build no longer has any justification.  The deeper understanding of
the requirements can carry with it disincentive, if you're just
looking to build something.

One of my first consulting engagments was with a New York bank
building out a power trading system. This was in 2004, and the market
for power trading was re-emerging after collapsing in the wake of the
Enron disaster. For a trading organization like my client, it was
useful to trade power to be able to create hedges, and for the
traders, they had to trade to be able to make money. The lack of a
trading system was therefore blocking both personal and business
goals, and needed to be addressed quickly.

Contrary to our advice as consultants, our client had decided to build
out their trading platform from scratch in Java. The architecture was
client-server with a Oracle back end, a Swing front end, and some
broadcase messaging across the clients to keep the clients up to
date. Initial scope was limited to the very simplest sorts of
deals. As consulatants, our role was to support them in design and
test as their in-house developers did the actual build. I came into
the role with more than enough experience to have directly contributed
to the software, but my role had focused on writing design documents
and testing somebody else's code. It was frustrating, but very good
experience.

This project was my first real exposure to the notion of the 'training
issue'. Coming from a background of packaged software development, my
instincts were competely aligned around building software that helps
avoid user error. In mass market software, there aren't opportunities
for formal training, so the software itself has to fill in the
gap. The lack of ability to train users to avoid problems makes it
worthwhile to spend more effort to eliminate problem areas in the
software. Taken back to the idea of an MVP, the requirements for a
commercial software package to be viable for its user base can be
considerably higher than for in-house software.

However, this trading platform was different than what I was used
to. It was in house software written for a user base known in advance
that numbered in the dozens. With a user community that well known and
small, it became feasable to train them all ahead of time, or as
issues were found. A crashing, high severity bug that might block a
mass market software release might easily be addressed by just
training users how to avoid it.  In this scenario, it was often
cheaper and faster to train users around the rough spots in the
code. (There were still bugs that might block a release. Anything that
would allow an audit check or trading limit to be bypassed is a good
example. The bar for a bug to block a release was just higher, not
eliminated entirely.)

As a software purist, this was initially anathema. The notion of the
training issue, and consequent acceptance of certain bugs, was a
difficult pill to swallow from the mindset of someone trying to
produce high quality software. However, as a consultant trying to help
get a business moving quickly it made perfect sense. Remember, the
goal was to get the company able to hedge in the power market, not
just write software. The software was a required cost of doing
business, and nothing else. An e-mail with a workaround can be cheaper
and faster than a full fix. How do you think a business reacts when
you tell them that they can't operate just yet, because there's a
field validation error or something. I'm not suggesting this is a good
way to build perfect software, but this is about *Minimum Viability*
and not perfection. Just like you may not need 24/7 earthquake
detection, you may also not need full fixes for all your bugs.

Note that this is not fundementally a negative message. The better
understanding of requirements you can achieve by pursuing a MVP can
ultimately lower the cost of your build. In the case of a trading
organization, it can get your traders doing their job more quickly. In
the case of an earthquake detector, it means you can afford more than
just one. Lowering the cost of your product can enable it to be used
sooner and in more ways than otherwise.

The punchline to the trading desk story is that most of the way
through the first phase of the build, our client changed
direction. They switched from a full custom build to a deployment of a
lightly customized commercial trading platform. They had done this
after realizing the costs of their initial choice, and that these
costs did not pay off in terms of useful required functionality.  The
system they settled on wasn't perfect, and there wasn't a big Java
build out, but it got them to market more quickly, particularly in
subsequent phases of the deployment. This is an extreme example of how
a focus on requirements can both eliminate the need to write a lot of
code and achieve business goals more quickly at the same time.

The concept of an MVP has power because it focuses your attention on
the actual requirements you're trying to meet. With that clearer
focus, you can achieve lower costs by reducing your scope. This in
turn implies you can afford to do more of real value with the limited
resources you have available.  It's not as much about doing less, as
it is about doingo more of value with the resources you have at
hand. That's a powerful thing, and something to keep in mind as you
decide what you really must build.
