+++
title = "My Rust 2020 wish list: playing nice"
date = 2019-11-03
+++
This year I decided to jot down some thoughts for the [Rust 2020 call for blog posts](https://blog.rust-lang.org/2019/10/29/A-call-for-blogs-2020.html).

The major caveat here is that I'm just a language enthusiast. I'm not using Rust professionally, and my skill in using the language is somewhat mediocre. We are a pretty diverse bunch, and other people and teams will have different wishes and expectations.

I think the general theme here will be integration or "playing well", on different levels. A lot of these here are not language changes, and most of them already exist in some form. Probably none of them are new ideas.

## Debugging experience

Rust got a head start here because of its similarity to C and C++, and being able to use the debugging symbols implementation from LLVM. With a bit of patience and skill you can make `gdb` or `lldb` work decently enough with Rust programs. I think, however, that we could and should do better. We shouldn't aim to match the `gdb` experience for C++ programs, but rather for C# or Java-like integration.

A first step here would be to improve the existing debug visualizers, and make sure they play nice with the three major debuggers we have &mdash; `gdb`, `lldb` and `vsdbg`. In the long run, I hope we will find a way to call existing `Debug` or `Display` implementations. This is probably very hard, since the formatting code might have been dropped by the linker from the final executable. Is there a way to leverage DWARF for it? I don't know.

Relevant issues:

- <https://github.com/rust-lang/rust/issues/50005>
- <https://github.com/rust-lang/rust/issues/36503>
- <https://github.com/rust-lang/rust/issues/40460>

## Linking speed

This doesn't apply to all projects, but a lot of users would be happier if `cargo` would use the LLVM linker (`lld`) by default. In some cases, linking is by far the slowest step in a build, more so in an incremental one. Using `lld` can dramatically cut down on link times, especially for large projects with a lot of debug information.

My understanding of the current status is that:

- on MacOS it's not worth using `lld` because it's buggy and unmaintained
- on Linux with GCC 9 or later it's easy to use `lld` by adding `-C link-arg=-fuse-ld=lld` to `RUSTFLAGS` or your `.cargo/config` file
- on Windows it should work, but there's a large chance that `cargo` will pick up the GCC packaged by the Rust toolchain, even if you have GCC 9 installed. I'm not sure what's the current situation with Visual Studio.

All of this info doesn't seem to be documented, but can be found floating around GitHub and other forums. Is there any hope of seeing `lld` as the default, at least on Linux?

Visual Studio also has incremental linking. Can we get something like that?

Relevant issue: <https://github.com/rust-lang/rust/issues/39915>.

## System integration

Right now, `cargo` stores the crates.io registry under `~/.cargo/registry`, and maybe also under `~/.cargo/git`. I'm not sure myself. I'd very much like if `cargo` was a better team player here. There are clear-cut guidelines for storing configuration and cached data, and `cargo` is not following them. This makes it painful for people using backup programs, roaming user profiles, or who just want to free up some disk space.

This was quite heavily discussed and even implemented, but eventually fizzled out:

- <https://github.com/rust-lang/rfcs/pull/1615>
- <https://github.com/rust-lang/rust/issues/12725>
- <https://github.com/rust-lang/cargo/issues/1734>
- <https://github.com/rust-lang/cargo/issues/1976>
- <https://github.com/rust-lang/cargo/pull/5183>

Notable here is the strong opposition from someone closely associated to the language. I believe it's in the interest of Rust to play nice with the platform expectations, instead of being its own special snowflake. There's nothing special to `~/.cargo/git` here, no deeper meaning, no potential for data loss if it removed. It's just a volatile directory, and should not be elevated to something that the users need or care to look at.

Mildly relevant: <https://github.com/rust-lang/rfcs/pull/2789>.

## Language features

I hope there will be progress on features like `impl Trait`, `async fn` in traits, const generics, streams, generators and perhaps specialization. Probably everyone wants these, so there's not much to say.

Maybe the next year the `chalk` and `polonius` work will finally bear some fruit, and perhaps there will be some resolution to the parallel compiler saga. Speaking of which, can `salsa` replace or supplement the `rustc` query system?

## Bringing crates into `std`

This is something that comes up from time to time. I think we should try to identify common patterns, especially in unsafe code, and offer good implementations in `libstd`.

Relevant discussion: <https://internals.rust-lang.org/t/calculating-which-3rd-party-crates-are-good-candidates-for-std-inclusion-via-left-pad-index/11129>

## Compiling system libraries

This is quite specific and is already requested, but I'd like to see `cargo` be able to build `libstd` instead of using a bundled version. This would be very helpful for embedded projects, and will improve code generation in some setups like projects that use abort on panic.

Relevant issue: <https://github.com/rust-lang/cargo/issues/4959>

## Popular `cargo` plugins

We've seen `cargo-vendor` get integrated into `cargo`, but there are a couple of other popular plugins that should follow the same route: `cargo-edit`, `cargo-outdated` and `cargo-tree` come to mind.

Relevant issue: <https://github.com/rust-lang/cargo/issues/4309>

## Paper cuts

I'm sure that everybody has some paper cuts they would like to see fixed. The worst I can think of right now are probably the weird interaction between `cargo` workspaces and features, and `html_root_url` which so many people forget to update that it's common to add a reminder about it in the manifest.

## The future of RLS

I was glad to watch `rust-analyzer` make great strides in the last year, from the nifty assists that were implemented, to `macro_rules` support and pretty decent method resolution. Pending some `chalk` improvements, it might become almost as precise as RLS, but with much better responsiveness.

The one thing I'm worried about is visible in the [contributors list](https://github.com/rust-analyzer/rust-analyzer/graphs/contributors). Not to discount the work of many people there, but the whole project is basically moving forward only through the effort of a couple of people.

In theory, `rust-analyzer` is an experiment to see if we can improve on RLS. I think we're quickly approaching a point where we need to answer that question. Is it fit to replace RLS or not? In my opinion (and that of many nightly toolchain users, I'm sure) the answer is a resounding yes. But I would like to see some official statement here. If it's deemed to be the way forward, maybe RLS should be put into maintenance mode, with those resources being allocated to `rust-analyzer` instead.

I believe that when `rust-analyzer` got started, there were some quite strong differences of opinion related to the long-term suitability of the RLS design. Hopefully we're over them by now, and we can make a decision that goes towards the best interest of the users.

## The RFC process

It's probably well-understood that the RFC process has some limitations that make it scale poorly. Perhaps a better approach here, which I'm sure was proposed before, would be to:

- move most of the RFC discussions to separate repositories, using issues and pull requests
- keep a long-running summary of these discussion with pros and cons in something like a HackMD document
- try to prevent team members from delaying RFCs by months or years with concerns or other things that never happen

## The communication platform

The Rust project seems to have settled on Discord as a replacement for IRC. I've never used that platform but, as a lot of others have pointed out, something like Matrix might be more in line with the values of the community.

## Community (part I)

Speaking of community, I'm pretty disappointed in how the new website project was handled. I'm not going to repeat the complaints that others have had about the contents, lack of translations, design, and the site being more or less broken in not so subtle ways. A lot of these were fixed in the last year.

But I think the launch of the new site was rushed out unnecessarily, and became a friction point that could have been avoided by saying "okay, a lot of people are crying out about this change we're making, maybe we should stop and re-think this for a while". Instead, we've seen helpful pull requests being closed without thanks or reason. Is that the impression we want to leave to new contributors? _If I had that experience with a project, I would be quite reluctant to get involved any further._

Speaking of which, some of us are probably [still waiting](https://internals.rust-lang.org/t/website-retrospective/9556/7) for an honest post-mortem on it. The point here is not to make heads roll, but to acknowledge that some mistakes were made, figure out what they were and what to do to prevent them from happening again. I'd like to see some accountability and transparency on this front, more than others.

To wit, there is this blurb on the the [Community team](https://www.rust-lang.org/governance/teams/community) page:

> Coordination and supporting events, content creation, running the RustBridge program, and conducting the survey.

I agree that those are important things to care about, but the community is more than that. We should acknowledge the regular contributors, the crate authors, the enthusiasts that help people learn the language on the various forums, even those that submit a drive-by pull request once in a blue moon.

We do have the Rust Survey once a year, but no formal feedback channel otherwise. Sure, some people will post on Discourse or Reddit, and some project members might notice and answer them, but I think there is some room for improvement here.

## Roadmap

I think I would like to see a rough and very high-level list or board of things that the team is actively working on, or planning to. Maybe the roadmap for 2020 could have a section of "we will probably want these in the future, but cannot work on them now, maybe the next year". Such a list could be updated during the year, as priorities and resources change.

## Community (part II)

There is a very deep rift in the community in the shape of async run-times and related projects. Much has been said about the two major implementations, yet to me it seems more and more that they are more similar than different. I think it's apparent here that the reasons for it are more social than technical. Time will probably make them converge more and more on the technical side, but the current situation is not ideal.

There is much to say here about various aspects, from naming to timing, and generally the reasoning behind the projects started by the now-disbanded Async-WG (nb. `async-std` is not an Async-WG project, even though the same people are involved). But these were already discussed a lot in different places, and I don't want to add anything here. Perhaps it is more important to ask:

1. is the _status quo_ of two very similar, but slightly incompatible run-times in the best interest of the users?
2. if yes, how do the two projects position themselves, which users are they catering to? It's worth mentioning here that the goals of the two projects are quite similar according to their developers &mdash; offering async versions of the functionality present in the standard library. One of the two projects actually uses this as its tagline, which tends to confuse beginners.
3. if no, what can be done to close this rift? The answer here is clearly not technical, and it will require giving up some control and some ego, and actively trying to work with the other side. This is hard to do, harder than any technical challenge that the two projects might encounter.

I also want to say that there is a certain level of misinformation and FUD floating around, most likely unintentional. We should work on that, because it can get toxic in the long run. In addition, I was very disappointed to see a comment on a well-intended [pull request](https://github.com/withoutboats/romio/pull/106) that was quite aggressive towards the competing project. It's now deleted, but it went like:

> After 2 hours, this issue got 17 thumbs up. After 5 hours, it has 20. I recommend closing this issue as spam.

(although initially it said "obvious trolling" in Russian, or something similar)

That's an awful way to speak about the competing project, which actually spawned both `romio` and the one you're working on. And I don't think it's in the best interest of `async-std` if its developers take this kind of position.

## Transparency

Finally, I'm not sure how well the working group structure has panned out. There are quite a lot of WGs, but I'm not sure how much progress they have been making.

I think the plan was to have regular progress meetings and keep minutes for each WG, but my impression is that these happened for a month or two, then they stopped.

Maybe the recently-announced [Inside Rust](https://blog.rust-lang.org/inside-rust/index.html) blog will be a simple solution to this problem.

## The future

As [somebody else](https://tim.mcnamara.nz/post/188733729327/rust-2020-lets-embrace-the-eternal-september) has mentioned, I think our greater challenges have yet to come, brought by new production users (let's not forget that the greatest mass of software developers is not visible in the Stack Overflow surveys) and maybe some internal tensions. I still think that Rust is a quite nice local maximum of programming languages, but we must steel ourselves and prepare for new challenges, from inexperience users, to companies tying to get more control over the project, to competition for other languages like C# that are inching close and closer towards Rust and Go.

## Conclusion

Even though I spoke more about things that aren't quite working, we've made a nice amount of progress this year. I'm not going to make a list here, as I'm sure others will be much better at it.

And while I've been somewhat negative on a couple of points, I still think we're doing pretty well overall. I'm only worried that social factors can prove to be a problem in the long run, and we &mdash; both the project members and the community &mdash; should focus a little on _resiliency_ in the long run. This means playing nice towards others, and being prepared for new users, more competition, and the potential for more conflicts.
