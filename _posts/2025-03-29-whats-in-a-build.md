---
layout: post
title: "What's in a build?"
description: "What would you say you do here?"
date: 2025-03-29
tags: [build]
---

I've worked as a build engineer, either on a formal build team or as part of a
more general "infra" team, for basically the entirety of my professional career
(beginning in 2018). Although when I'm asked what I *do* exactly at any deeper
level of detail than just "software engineer" I've never had a answer that felt
like it really captures the essence of what I think about day-to-day. This post
is intended to serve as that full explanation of what I think my job is.

## The "in-between" Team

Every build team I've talked to, heard of, or worked on has ended up being
responsible for a different set of things. I've worked on projects that could
rightly be classified as each of: CI, Developer Experience ("DevEx"), Release
Management, Engineering Productivity ("EngProd"), Toolchains, and
Infrastructure. Sometimes there were dedicated teams for those domains that I
worked with (and learned from). Sometimes there was just A Guy[^guy]. It is a staple
of the build team experience to find out there used to be A Guy, but they don't
work here anymore.

I've had the privilege of working on teams with some of the strongest generalist
engineers that I've ever met[^mike] which has heavily shaped (warped?) my
perspective on how build teams operate: I regularly describe my build teams as
the "team of last resort", in that if a thing needs doing and no other team seems
to really be responsible for it, it falls to the build team.

But what do we *do* exactly?

### Enablement

Build work is full of weird, one-off tasks that will never come up again[^weird].

[^guy]: Intended as a gender-neutral title.

[^mike]: Shout-out to Mike if he ever reads this. Thanks for teaching me so much.

[^weird]: At least, not again at the same employer.