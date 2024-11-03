---
layout: post
title: "I Waited For 10 Billion Cycles and All I Got Was This Loading Screen"
tags: software-philosophy
---

Modern hardware is unbelievably fast. The M1 Max that I'm writing this article
on runs at 3.2GHz. That is [3.2 _BILLION_ clock cycles per second][1]. Yet,
Microsoft Teams takes 3 seconds to open a link, and I refuse to believe it takes
9.6 BILLION clock cycles to open a link. Obviously, that's an
over-simplification, but the point stands: how is it that hardware gets faster,
but the applications we use only get slower?

[1]: https://www.cpubenchmark.net/cpu.php?cpu=Apple+M1+Max+10+Core+3200+MHz&id=4585

"That's just because we're doing so much more." These are the words of a
[late-stage capitalism enjoyer][23]. Let me explain.

[23]: https://www.youtube.com/watch?v=4EmstuO0Em8

An excellent example of the power of modern computing is video games. I can
simulate massive 3D environments, complete with physics and ray-traced
lighting, and play in real-time with my friends in other states or even
countries, all while pushing 124 million pixels per second[^1] on reasonably
attainable consumer hardware.

[^1]: 1080p at 60 FPS = 1920 × 1080 × 60 = 124,416,000

Looking at it in the other direction, people have managed to [run DOOM on just
about anything that can compute][2]: calculators, iPods, cameras. Incredibly
low-power, practically disposable devices still have enough computing power to
run a state-of-the-art game from 1993. This isn't particularly surprising,
given it's been three decades, but it shows how far we've come.

[2]: https://www.reddit.com/r/itrunsdoom

### "Web is bad"

Web is cool. In fact, web is so cool that it's in its own class of backwards
compatibility, cross-platform-ness, and approachability. Web also has an
event-driven model that makes UIs a lot easier to write. Of course, these
conveniences also come at a cost: speed. It's inherently difficult to execute a
language as flexible as JS, and it's likely the first thing you blame when
comparing a fast, native application against a sluggish, web-based one. Of
course the interpreted, dynamically-typed language is going to use more memory
and run slower.

Depending on what you're doing, JS might deserve the blame. For instance,
[webpack][3] has to parse thousands of files, build an AST, and do some fancy
CPU-intensive things I don't fully understand, so it makes sense that [esbuild
(Go)][4] and [swc (Rust)][5] are much faster than webpack at that.

[3]: https://github.com/webpack/webpack
[4]: https://github.com/evanw/esbuild
[5]: https://github.com/swc-project/swc

In the case of general web applications though, nope; web isn't the reason your
glorified IRC chat application runs at 5 FPS. An example that was floating
around [Twitter][7] a couple weeks ago is [McMaster-Carr's catalog][6]. Through
aggressive preloading and server-side rendering, they're able to make the
website feel incredibly snappy to use and navigate, while using decades-old
technologies (ASP.NET, jQuery). Bandwagon jumpers are in shambles.

[6]: https://mcmaster.com
[7]: https://twitter.com/KennethCassel/status/1847034096062710087

If you're a React hater, this may seem like a great argument against React and
other "modern" JS frameworks, but that's entirely[^2] the wrong takeaway. Check
out [NextFaster][8] ([repo][9]) for an equally-or-faster copy of the McMaster
catalog built with Next.js and hosted on Vercel.

[^2]:
    There is definitely an argument about "the tower of complexity" and the
    infinite abstractions we created just to put boxes on a screen, but it's not
    a valid performance argument. You can write performant applications with
    almost anything.

[8]: https://next-faster.vercel.app
[9]: https://github.com/ethanniser/NextFaster

Probably the epitome of web technologies is [Figma][10]. A full-blown design
tool running at 60 FPS with real-time multiplayer is kind of insane when you
think about it. To be fair, it's not entirely JS; a large portion of the app
runs in WebAssembly and WebGL. Could they squeeze more performance out of a
native app? Probably. But that would also mean giving up the convenience of
being a website you can just... go to. Instead, they [went all-in on
optimization][11] and proved that browser can absolutely rip.

[10]: https://www.figma.com
[11]: https://www.figma.com/blog/quality-and-performance/

Side Note: Massive shout-out to V8, SpiderMonkey, and JavaScriptCore devs for
making all of this possible. It's because of your arcane work that the rest of
us are able to build anything at all.

### "Electron is bad"

_Note: I am using "Electron" as a stand-in for Electron, CEF, WKWebView, Edge
WebView2 and any other web-to-native wrapper._

[Electron][12] made it possible to make web applications and deploy them as
desktop applications. The appeal of this is undeniable. Now you can hire a
single skill set, build a single application a single time, then have "desktop"
apps for every architecture and architecture and OS under the sun (yes, even
Linux).

The trade-off? The only reliable way to
run web applications is a web browser, so Electron just... shipped a web
browser. What could go wrong? ¯\\\_(ツ)\_/¯

[12]: https://www.electronjs.org/

Turns out, a lot of things could go wrong. Over the last decade, it's become
socially acceptable for application downloads to be over 500 MB, and for them to
use egregious amounts of RAM and CPU, slowly killing your battery life in the
process. And for all the resources they suck back, it's not even like they feel
good to use. So many apps leak their non-nativity through strange scrolling and
selection behavior, appearance, and navigation handling, not to mention the
dismal sluggishness that comes with changing screens. Discord and Teams are
great examples of this Electron-ification. They both have solid mobile apps that
look and feel good to use, but desktop just gets a re-boxed website. _Why?_

It's not really fair to blame Electron for this. Electron simply said, "here's a
way to put your web app in a window".

The real responsibility lies with the companies building these apps. It's
definitely possible to build good Electron apps - Slack, Obsidian, and Notion
are prime examples. You just have to care.

A section on Electron wouldn't be complete without mentioning [Brackets][16],
[Atom][17], and more famously, [VS Code][18]. It's honestly impressive that VS
Code runs as well as it does, considering that editing large files in JS is
approximately like shooting yourself in the foot and running a marathon: a point
was proved, but I'm not sure which one.

[16]: https://brackets.io/
[17]: https://github.com/atom/atom
[18]: https://code.visualstudio.com/

It's undeniable that a large portion of VS Code's success should be attributed
to the plugin ecosystem around it, and that's in no small part because of the
approachability of the web stack. Having recently written a [(meme) plugin for
SWC][13], I can confidently say that native plugin DX isn't anywhere close to
the über-fast iteration cycles of the web.

[13]: https://www.github.com/preyneyv/swc-plugin-use-prompt

### "Native is bad"

All this web technology disparaging may have you believe that I love using
native apps. Most native apps feel great to use ([Postico][21], [Zed][22]) and I
often find myself reaching for them whenever possible.

Having switched to [Zed][22] a couple months ago, I don't think I can go back to
VS Code. Everything (and I mean everything) happens _instantly_. You may think
VS Code is "fast enough", but it's like the speed of sound versus the speed of
light. You can just barely perceive it, but it really makes all the difference.
How does Zed do it? By writing their [own GPU framework from scratch][15] and
writing the entire editor in Rust. When you actually use modern hardware, it's
_really_ fast. Who would've thunk?

[15]: https://www.gpui.rs
[21]: https://eggerapps.at/postico2/
[22]: https://zed.dev/

One thing I'm keen to play around with more is [React Native][19] for desktop.
RN's DOM-like rendering model appeals to my web-dev sensibilities, and I think
it strikes the balance between great DX and great performance, especially with
the [New Architecture][20]. It is presently unclear to me why RN for desktop
isn't particularly common, because it seems like the ideal balance on paper
(though it may just come down to first-party support).

[19]: https://reactnative.dev/
[20]: https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here

Unfortunately, all is not well in native-land. Adobe products are
known to crash exceptionally often. The search menu on Windows 11 is comically
bad. Activity Monitor on macOS takes 5 seconds to populate a list of running
programs.

### So we're doomed, then?

Clearly, these issues aren't localized to a specific programming language,
operating system, or industry. It's a broader trend to build things faster and
worse. Delivery over quality.

#### Tragedy of the Commons

Things are particularly worse in recent years because modern computers are so
much faster than what the average person needs. If you scroll YouTube Shorts on
a desktop browser, it starts leaking RAM (over 2 GB!) until it eventually hits a
resource limit. That kind of stuff wouldn't fly in times of yore, where
computers only had 2 GB of RAM total. The seemingly ample computing resources of
modern machines have become an excuse to build worse things.

#### The "Good Enough" Psy-Op

Clearly, I care a lot about things feeling good to use, but from talking to
various people, I realized it's not as common as you might think. Most people
are "okay" with waiting 500ms for a right-click context menu to appear, and they
look past the janky reflow that happens when you resize a window. It's either a
hold-over from a bygone era of waiting 15 seconds for an image to load or a
learned resignation to poor user experience. We can do better. We have the
technology.

#### "Ship It" Culture

Build things, and build things fast. This article isn't an indictment of rapid
iteration on new ideas, but there's a massive difference between treating
performance as tech debt and ignoring performance entirely. In so many of the
things we use day-to-day, it feels like performance isn't even an afterthought,
and it's more like they wait for hardware to catch up. That's a valid approach
if you're doing something bleeding-edge, but if your job is to put text on a
screen, I find that hard to agree with.

### Performative performance

From reading this article, you may infer that my recommendation is to optimize
everything down to the CPU instruction. While that is impressive in its own
right, it is entirely intractable for most software (unless you're Chris Sawyer
making RollerCoaster Tycoon). This article also isn't about performance for
the sake of performance. It is simply the realization that large-scale modern
software seems to care very little about how it feels to use, because you'll use
it anyway. That's a future I don't want.

If you are a software developer, I encourage you to build things with precision
and quality. I view software as an art, and I want to be able to appreciate its
craftsmanship.

Stop doing less with more.

---