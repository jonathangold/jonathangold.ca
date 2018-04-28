---
title: "AWS:Ec2 in Mgmt"
date: 2018-04-28T00:11:43-04:00
draft: false
---

Before dealing with the nitty-gritty of the <a href="http://docs.aws.amazon.com/sdk-for-go/api/service/ec2/">EC2 API for Golang</a> (and boy is it gritty,) it will help to understand the basics of <a href="https://github.com/purpleidea/mgmt">mgmt</a> and how resources are implemented. Mgmt is an open-source event-based config management solution written entirely in Golang, based on a <a href="https://en.wikipedia.org/wiki/Directed_acyclic_graph">Directed Acyclic Graph</a> model. It is the brain-child of <a href="https://twitter.com/purpleidea">@purpleidea</a>, who maintains the project and provides mentoring to new contributors (myself included.) Mgmt's resource modules each define methods to watch, verify and change the state of any resources the user may need to maintain, as well as any relationships to other resources where appropriate. This is the story of my battle to write <a href="https://github.com/purpleidea/mgmt/blob/master/resources/aws_ec2.go">one of those modules</a>.
<blockquote class="twitter-tweet" data-cards="hidden" data-lang="en">
<p dir="ltr" lang="en">DAE think that the <a href="https://twitter.com/awscloud?ref_src=twsrc%5Etfw">@awscloud</a> EC2 <a href="https://twitter.com/hashtag/golang?src=hash&amp;ref_src=twsrc%5Etfw">#golang</a> API is gross?<a href="https://t.co/gdDFRyJJ1t">https://t.co/gdDFRyJJ1t</a></p>
Happy to provide constructive criticism to aws engineers.

— James Just James (@purpleidea) <a href="https://twitter.com/purpleidea/status/917287869340880896?ref_src=twsrc%5Etfw">October 9, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

With that out of the way, let's dive into the implementation. There are two crucial methods that <em>must</em> be implemented in order to get a resource working in mgmt; <a href="https://github.com/purpleidea/mgmt/blob/master/docs/resource-guide.md#watch">Watch()</a> and <a href="https://github.com/purpleidea/mgmt/blob/master/docs/resource-guide.md#checkapply">CheckApply()</a>. Both methods do what they say. Watch() monitors the resource and sends an event when it detects that the state has changed, and CheckApply() checks the state and, if necessary, applies the actions required to bring the resource into the desired state.

One thing I feel compelled to point out is that, like all of Amazon's APIs, the AWS SDK for Golang is auto-generated, as is its documentation. While this undoubtedly saves Amazon's developers countless hours of work, it makes programming against the API significantly more frustrating than it ought to be.

<img src="img/it-crowd-maurice-moss-frustration.gif" alt="" width="500" height="289" />

I began my implementation with the CheckApply() method, as that tends to be the quickest way to a working proof-of-concept. The check portion of the method needs to query the API for any instance that matches the name specified by the user. If it finds one, it then needs to check its state against the definition. If the state matches, we are done. That was easy.

If the resource is not in the correct state, we continue to the apply portion of the method. Here's where the real work gets done. Apply defines the actions needed to bring the resource into the desired state. It consists of the logic required to create, start, stop and terminate EC2 instances. Once I wrapped my head around the API (which was no small feat,) this code was straight-forward to write, and was fairly close to the examples available in the documentation.

Once the CheckApply() method was finished, I could actually run it using mgmt's poll meta-parameter. Poll essentially bypasses the Watch() method, and just repeats CheckApply() on an interval defined by the user. It worked! There were a few bugs work out in the logic, but once they were addressed, I could be certain that no matter what actions I took, mgmt made sure that the resources I defined would always return to the correct state.

<img src="img/great-success-gif.gif" alt="" width="476" height="360" />

The next step was to implement Watch(). This would prove to be more difficult. Until I impliment an http server to receive events from AWS CloudWatch we have to rely on the APIs WaitUntilInstance... methods to alert us that the state may have changed. I say may, because these methods return after about ten minutes, whether or not the state has changed. It would be really helpful if the API returned an error, which we could use to trigger the method to retry. When the wait method returns, we actually have to check whether the state actually changed, or if it just timed out, by rechecking the state. If the state really has changed, we send an event to the engine.

There are some issues with this approach, including two subtle races. The first consists of notifying the engine that we're running only when the watching actually begins (rather than when we ask it to.) The second is sending events only when we are sure the watching has restarted after detecting a change. Luckily <a href="https://twitter.com/purpleidea">@purpleidea</a> recently demonstrated a <a href="https://github.com/purpleidea/mgmt/blob/master/examples/longpoll/redirect-server.go">race-free HTTP server</a> implementation that I may use in a future patch to leverage <a href="https://aws.amazon.com/cloudwatch/">AWS CloudWatch</a> events in Watch().

Between navigating the auto-generated API documentation and puzzling together the logic required to watch the resources reliably, writing this module was a challenge. If the API were a little more sane, I might have saved a little of my own sanity. I couldn't have done it without <a href="https://twitter.com/purpleidea">@purpleidea</a>'s awesome mentoring and the community he's built around mgmt. Come join us on freenode at #mgmtconfig and on <a href="https://github.com/purpleidea/mgmt">GitHub</a>.
