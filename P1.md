Moving gRPC to the Cloud Native Computing Foundation (CNCF) and ASLv2
-------------------------------------------------------
* Author(s): Varun Talwar, Chris Aniszczyk
* Approver: a11r
* Status: Draft
* Implemented in: N/A
* Last updated: 2017-02-01
* Discussion at: https://groups.google.com/forum/#!msg/grpc-io/AWCJlR-MA9k/N-EKJtQPAwAJ

## Abstract

* Move the gRPC project/community under the auspicies of the Cloud Native Computing Foundation (CNCF)
* Move the gRPC license to Apache License v2.0
* Host gRPC tracks/event at CloudNativeCon/KubeCon events in late March and early December

## Rationale

The Cloud Native Computing Foundation (https://cncf.io) is the current neutral home of the Kubernetes project and four additional projects in the cloud native technology space ([fluentd](http://www.fluentd.org/), [linkerd](https://linkerd.io/), [prometheus](https://prometheus.io/), [opentracing](http://opentracing.io/)). The CNCF acts as a neutral home for gRPC and increases the willingness of developers from other companies and independent developers to collaborate, contribute, and become committers. There's a plethora of benefits available for projects under the CNCF which is discussed here (https://www.cncf.io/projects) but some that may be relevant to the gRPC community are:

* Our world-class events team will create a track or custom conference for your project at our CloudNativeCon/KubeCon events around the world, bringing together developers and users. 2017 events will take place in at least North America, Europe, China, and Japan.
* You get priority access to the $20 million CNCF Community Cluster (https://github.com/cncf/cluster), a 1,000 server deployment of state-of-the-art Intel servers housed at the Supernap Switch facility in Las Vegas. We encourage you to do large-scale integration testing before releases, as well as testing experimental approaches and the scalability impact of pull requests.
* You will have control over a substantial annual budget (currently ~$20 K) to improve your project documentation.
* We have travel funding available for your non-corporate-backed developers and to increase attendance of women and other underrepresented minorities.
* We will connect you to our worldwide network of Cloud Native meetup groups (http://meetups.cncf.io) and ambassadors to raise awareness of your project. We will also help sponsor meetup groups dedicated to your project so food and beverages can be provided.
* You will have access to full-time CNCF staff who are eager to assist your project in myriad ways and help make it successful.

The important aspect is that your existing committers still control your project, and we just ask that you have a neutral, unbiased process for resolving conflicts and deciding on new committers and moving existing ones to emeritus status.

For an explanation of why the CNCF prefers the Apache Public License 2.0, see this blog post:
https://www.cncf.io/blog/2017/02/01/cncf-recommends-aslv2

## Proposal

We propose to give gRPC and its existing projects (https://github.com/grpc and https://github.com/grpc-ecosystem) a new home at CNCF. The suggested process to make this happen can be done in four steps:

* Have this gRFC proposal open for 10 days and serve as a notice and hub for any community questions around the move.
* Move gRPC to the Apache License v2.0 on Feb 15th
* File the official gRPC proposal as a CNCF project and have the TOC call for a vote
* Assuming the vote is accepted, announce the move to the public and throw a party!

This process should take 2-3 weeks to complete depending on community feedback and voting.

### Related Items:

* https://github.com/cncf/toc/pull/23
* https://www.cncf.io/blog/2017/02/01/cncf-recommends-aslv2
* https://www.cncf.io/announcement/2016/03/10/cloud-native-computing-foundation-accepts-kubernetes-as-first-hosted-project-technical-oversight-committee-elected
