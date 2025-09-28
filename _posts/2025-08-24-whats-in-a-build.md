---
layout: post
title: "What's in a build?"
description: "What would you say you do here?"
date: 2025-08-24
tags: [build]
---

I've worked as a build engineer, either on a formal build team or as part of a
more general "infra" team, for basically the entirety of my professional career
(beginning in 2018). Although when I'm asked what I _do_ exactly at any deeper
level of detail than just "software engineer" I've never had a answer that felt
like it really captures the essence of what I think about day-to-day. This post
is intended to serve as that full explanation of what I think my job is.

<!--more-->

## The "in-between" Team

Every build team I've talked to, heard of, or worked on has ended up being
responsible for a different set of things. I've worked on projects that could
rightly be classified as each of: CI, Developer Experience ("DevEx"), Release
Management, Engineering Productivity ("EngProd"), Toolchains, and traditional
Infrastructure. Sometimes there were dedicated teams for those domains that I
worked with (and learned from). Sometimes there was just A Guy[^guy]. It is a
staple of the build team experience to find out there used to be A Guy, but they
don't work here anymore. In lieu of a dedicated Build team is often the case
that the Build work is done by a smattering of (unlucky?) individuals.

I've had the privilege of working on teams with some of the strongest generalist
engineers that I've ever met[^mike] which has heavily shaped (warped?) my
perspective on how build teams operate: I regularly describe my build teams as
the "team of last resort", in that if a thing needs doing and no other team seems
to really be responsible for it, it falls to the build team.

But what do we _do_ exactly?

### Enablement

At the heart of Build is doing the work to make it possible for other
engineering teams to do their work: Our domain begins when you save the contents
of your text editor to disk and ends with the production of some usable output
artifact or collection thereof. This includes the obvious things like providing
build automation (e.g. Make, Gradle, Bazel), toolchains (C++ compilers, JDK
versions, Python interpreters), and developer tooling (linters, formatters,
debuggers). On top of the obvious things Build work is **full** of weird,
one-off requests that will probably never come up again[^weird]. These are hard
to describe categorically other than by their uniqueness and somehow being in
the critical path of the development of the product.

To pull from my own experience the following are all real requests made to
Build teams I've worked on:

- We have a monorepo, except for this one project[^snort]. Can you pull that
  entire project into a subfolder of the monorepo? Also that import needs to
  preserve the 100+ commit Git history of that project. Also don't break CI or
  disrupt the release we're cutting soon.
- We sell our product to the government and they are going to start requiring us
  to provide an exhaustive
  [SBOM](https://www.cisa.gov/topics/cyber-threats-and-advisories/sbom/sbomresourceslibrary).
  We need to able to automatically produce those for every release of our
  product.
- We develop a safety critical system and originally targeted an x86 platform,
  but we now need to target an aarch64 platform for unit cost reasons. What will
  it take to cross-compile our full software stack and to verify the behavior to
  comparable levels to what we have today?

Often there are things that the engineers I support would want but don't ask for
because they aren't aware of them or don't realize they are possible to do at
all. Being able to offer a simple and sometimes almost surgical intervention for
a thorny problem is a particular delight for me, so I view it as part of my job
to be deeply familiar with the tools we rely on in order to be able to do so.
Some good examples in this realm are lesser-known but powerful toolchain
features for instrumentation or optimization (e.g. LTO,
[PGO](https://aaupov.github.io/blog/2023/07/09/pgo),
[BOLT](https://github.com/llvm/llvm-project/blob/main/bolt/README.md)),
quirks/"features" of Python's import system & packaging, or just being the only
person willing to go make changes to linker flags.

### Scaling Costs

With the addition of each new engineer the complexity and cognitive overhead of
contributing to a project grows. Unless these creeping forces are actively
mitigated the productivity of each individual inevitably declines. If we view
this as a continuous optimization problem, then as projects grow in size there
is some critical point where adding an additional engineer to work directly on
the product results in less marginal "progress" than adding an engineer to work
on reducing the existing overhead. Hence: a Build team is born.

#### Coordination Problems

Most problems in software have more than one acceptable solution and wasting
work solving the same problem more than once is a cardinal sin of project
management. So all we have to do is pick one of those solutions and stick with
it, right? Even better, maybe someone else has already had the same problem as
you and came up with a reasonable approach that you'd be happy to reuse. All you
have to do is figure out how to integrate the code they wrote with what you're
currently working on! Wait, why doesn't their library import cleanly? Is that
`PYTHONPATH` hack required? Should we ask them to publish
[wheels](https://stackoverflow.com/questions/38547651/what-are-wheels-used-for-and-why-are-they-helpful-in-the-context-of-installing)?
Wait, why doesn't it build on my machine? Is that linker flag a GNU extension?
Maybe we can just vendor their library into our project and fix up the import
errors ourselves? Oh, then it turns out there was a security fix that our
vendored copy didn't receive. Maybe you can just land your fixes for the import
errors upstream? But now your PR is getting held up in code review and you're
getting comments explaining how your curly braces are wrong and linking to their
style guide and you roll your eyes and make the changes because you're just
trying to solve this problem and don't understand why this whole process has to
be so difficult.

Beyond the technical decisions made by a team within their domain there are
countless choices made about how teams interact with one another. These customs
& norms have more often than not have grown organically over the lifetime of the
project rather than originating from any sort of legible decision-making
process. And whether or not they're written down anywhere or automatically
enforced by tooling these cultural conventions absolutely exist; Sometimes the
only way you find out about them is by someone angrily telling you that you've
done things the wrong way.

The Build team sits at a unique position to influence & set policy on such
matters. Import conventions, package layout, deployment artifacts: Any number of
reasonable choices exist that have little value over one another and it is
primarily costly when teams *differ*. As these issues are usually secondary
considerations to the real problem an engineer is trying to solve they are often
happy to be able to copy from or simply follow blessed examples of the "right
way" to do things. The soft power of controlling the "default" option or
deliberately making a desired choice the easiest path to take leads to a policy
being adopted with no authoritative mandate or expenditure of political capital.

If you've never had a frustrating experience like the one described in this
section, consider sending a thank you note to your Build people. :)

#### Tighter Loops

The pace of software development is constrained by the length of the feedback
loops present in the system. Edit-Compile-Run-Debug. PR->Merge->Release->Deploy.
Review comments, make changes, re-upload. Hard problems take multiple attempts
and multiple iterations to figure out, so an increase in the time it takes to
receive a signal on our in-progress work has a multiplicative impact on the time
it takes for that work to proceed to the next stage of its lifecycle[^flow].

Speeding up these processes and tightening these loops is possibly the most
obvious and most visible work a Build team can do. Sometimes it's just a matter
of [making the compiler faster](https://llvm.org/docs/HowToBuildWithPGO.html),
but once all the low-hanging optimization has been squeezed out of your tools we
have to start thinking about the feedback loops themselves and figure out how to
get that information to developers sooner: If a critical property must be upheld
and violations are only found after deployment resulting in a change made a week
ago being reverted, can we add a direct check for that property in a test suite
in CI? If developers have to upload their changes to get the results of that
test suite populated on the PR, can we make it so they can easily run those same
tests locally? If that's already possible, do the runtime environments and
behavior of those test suites match between the local developer flow and the CI
suite so that developers actually *trust* the local results? Could this property
be upheld by a linter or static analysis tool rather than requiring a test suite
to be run (or a human to make a comment on a PR)? Can we make it so those lints
& checks are shown to the developer in their editor, before they even hit
"Save"?

More thoughts about the role of Build and what it means to be good at the job to come in a subsequent post.

---

[^guy]: Intended as a gender-neutral title.

[^mike]: Shout-out to Mike if he ever reads this. Thanks for teaching me so much.

[^weird]: At least, not again at the same employer.

[^snort]: I had an interview once where the person I was speaking to described their situation as "a collection of monorepos". I did not successfully contain my snort.

[^flow]: If you like to think about this sort of thing I *highly* recommend [The Principles of Product Development Flow](https://www.goodreads.com/book/show/6278270-the-principles-of-product-development-flow).
