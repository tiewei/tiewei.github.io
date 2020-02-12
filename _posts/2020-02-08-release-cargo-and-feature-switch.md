---
layout: post
title: "release cargo and feature switch"
---

# release cargo and feature switch

## Backgroud

A software release is about deliver a set of features to customer at a certain date. There are two important factors here, one is a set of features, the other is the date. In tranditional model, we set a fixed date as "release date", and then counting backwards to decide "code freeze date", planning each sprint deliver milestone (progress). This model requires each feature developer have a good estimation of how much time the feature developing needs, and also the features are relatively standalone, only met these two conditions, the release date might be able to be kept. In real world though, sofeware development effort estimation rarely worked well, ["hofstadter's law"](https://en.wikipedia.org/wiki/Hofstadter%27s_law) and ["90/90 rule"](https://en.wikipedia.org/wiki/Ninety-ninety_rule) always won. In addition, the "independent features" are always little part of the feature set, especially in monolithic project.

Agile devleoping think another way, in its [manifesto](https://agilemanifesto.org/principles.html), "deliver working software frequently" and "working software is the primary measure of progress" are two key principles about release. It's easy to understand, if more features are set within a release, more developing time and testing will need to be done, and it's harder and takes longer to make a release, then people tend to make fewer releases and it in turn means there would be even more features in one release. That leads to a vicious cycle. If we change the way we do release, make it smaller and easier, we can release more often, and it helps feature velocity and reduce the drama before the release date.

In most agile practice, it mentions investment tools like test automation, CICD, and more recently microservice was invented to arrange application as a collection of services, reduce the dependencies between teams/codes, so that feature developing can be better decoupled, helped improve the feature velocity. However it doesn't provide much guidance on how to balance the release date and features, which usually will be heard between product and developer.

## Decoupled Release

### DNV Hypothesis

We might have notice in agile practice, in one release, we can only get two out from these three gurantees: 1. fixed release date(**D**), 2. fixed number of features(**N**), 3. feature velocity (**V**) (deliver as many features as possible)

Unrigorous proof:

1. In order to provide fixed release date and features, usually feature would be delivered slower overall in afraid of impacting changes cross releases or microservices ([Hyrum's Law Online](http://www.hyrumslaw.com/))
2. In order to provide fixed release date and ship as many features within a long time period(velocity), the feature promised within a single release usually can't be fixed, otherwise any bugs or regression will delay the date
3. In order to provide fixed set of features and able to ship as many  features as possible within time, the date is very hard to promise for each release (fall back to tranditional model)

We can find examples for all three cases:

1. To make fixed date and features, developer prefers to take "safe" changes that can fit into one release, and it's hard to introduce big feature that could cause impact for other features. e.g. upgrading helm2 to helm3
2. To make fixed date and want adding impacting features and refactors, we can't promise that impacting feature will in which release. e.g. switch internal driver to tinker
3. To make fixed features and as many features as possible, the date slips often.

### Decoupled Release

In most cases, we would prefer keeping a short and fixed release date, in order to be able to release more frequent. And between fixed set feature within a release and features velocity, it depends on the coupling between features. If it's a security requirement has to be fixed in next release, we prefer make it as fixed features, or if it's a multi-depedent feature (e.g. k8s upgrading), we might not able to promise if that would be ready in the next release. Usually the features have to be included within a release are those might cause the software "stopped working", things like bugs that causes crash, data loss or leaks. In short, a release prefers a fixed fequency, without promise what will be include within the release unless they are absolutely required.

Then our question becomes how  to reduce the chance of introducing a "stop working" bugs. It usually introduced by deep-coupling features - things like a foundational refactor, backward incompatible changes, or change depends on multiple other features, such changes usually huge, involves multiple components, hard to review and tests.  In order to improve it, we need to make sure the features in releases can be decoupled, and easy to review and test.

A decoupled release that has fixed frequency, without promising the features within each release, and offers decoupled features can be review and tested easily. It provides way for us to release on time, with known set of bug fixes and features, and has ability to introduce large change progressively.

## Release cargo and feature switch

### Concept

In order to put decoupled release work in practice, we can introduce release cargo and feature switch.

If we think a decoupled release is like a train system, it rans between two stations, one side is developers, the other side is users, or whatever next step it is in the delivery cycle. The train always arrives and departures on time. The cargoes on the train are the features will be shipped from developers to users. Developers mission is to put as much features (**V**) onto the train before it leaves (**D**), while keep the training deliver at least as much as cargos it was last round(regression). We build tools like CICD to make it as fast as possible to put cargos on to the train, so that the major time costing is to prepare the cargoes.

For small cargoes(feature) or tinkers (bug fixes), it's easy, we just load the cargo onto the train when it comes. However when it comes to large cargos, like some large features can't be completed within a release cycle, or an engine repair, like some refactors or fundamental changes, it becomes tricky. If we put cargo onto the train only when they are ready, the moment putting it onto the train, we might not know how much impact it will cost to adjust spaces and engine horsepower, things like merging a large feature branch, preparing acceptance testing plan, updating user manual and so on. Unfortunately these situations are pretty common for an infrastructure related platform, for instance, a top to button TLS enablement, a major dependent upgrading (k8s version), or converting pattern from synchronized operation to asynchronized one.

What if we put uncompleted features onto the train early? It might sound strange to ship uncompleted features, but if we could put a "feature switch" to help us hide such feature from user (not only just from UI, but also disabled), it could bring us many benefits: it would help us review codes related to the feature early, exposes integration problems early and reduces the size of feature branches and unmerged codes that hide potential issues or technical debt, the feature switch turn on and off a underdeveloping features can help us tests, integrate the feature internally but not visible for users till they are completely accepted. Most importantly, for even we found the last minute issue before release, we can always turn such feature off and defer the effort fixing it later. This approach has been widely used in web developing for A/B testing and  can be used in infrastructure platforms too.

### Practice

In practice, due to the difference of infrastructure platform and a website, it might be different.

An infrastructure platform has some key factors

1. heavy to upgrade or rollback changes
2. have multi-layers dependencies
3. exposed api needs to be backward compatible
4. if it fails, it impacts all applications running on the platform

In order to adopt the release cargo and feature switch, we need to change the way we developing features:

1. When start developing a big feature, the first thing to do is spliting a large feature to a set of dependent small parts
2. for each small part, we need to start with adding a feature switch in config file
3. find the package/module that implements the small feature, try to abstract the changing pieces as interface
4. keep all existing implementation of such feature, run regression tests to make sure existing feature still work
5. add an switch function based on the config flag, keep defaults to old implemenation
6. implement the new feature in different package/module, piece by piece, send review every time a small reviewable/testable codes are added
7. review and test the feature in CI with special flag that have such feature turned on
8. repeat 2-7 until all feature parts are review and tested
9. push to production with both code paths and switch new feature on as default, ship for customer beta testing
10. if anything goes wrong, turn off such feature and repeat step 9, otherwise annource deprecating of old feature
11. remove feature switch, old code path at next major release that deprecats it.

In above steps, whenever a relase train arrives, ship the code as it currently is.

### Case study

We actually did such practice when adding a new external network driver support for an IaaS platform.

1. at first, we planed to have the network driver interface, the internal driver was the first implementation for quick start
2. we started new external network driver project, while regular release of IaaS platform are still going, for about 6 weeks
3. when we do integration, we reimplemented the interface with new driver, and added a series of feature switches, a "netDriver" switch to decide to use internal or external driver
4. while keeping the release with internal driver, we setup CI to have external driver enabled to test the result, and a "decoupledFeatures" switch to allow us adding cni, nodeconfig, ingress, lb one step at a time
5. after all testing done and switch CI to use external driver by default, we released the first version using external driver to end user while keeping the internal driver codes and CI running
6. after another 2 releases, we removed intneral driver completely

In this pattern, we didn't ever blocked release due to the big switch, all the features were review and integrated progressively. And this pattern should be able to be adopt for k8s upgrading or anything that might be too big enough to be fit into a fixed release time frame.

## Summary

In infrastructure platform developing, it's common to have heavy features that hard to be completed within one release time. So we either be able to keep the release pace and adding as many features as we could in one release, or keep the feature set and productivity but slips release date. In most cases we want to practice the "decoupled release", that relaxed features from release schedule, and decouples features dependencies if possible. And release cargo and feature switch could be one method that can be used in infrastructure developing, would help to improve the release timing, large feature developing, discover the integration problem early and encourage setup CI for all features added, all of these, would help a software developing team deliver a better product on time.

