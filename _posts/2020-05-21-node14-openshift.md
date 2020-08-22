---
layout: post
title:  "Use Node.js 14 on Red Hat OpenShift"
date:   2020-05-21
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2020/05/21/use-node-js-14-on-red-hat-openshift/
author: lholmquist
---

On April 21st, <a href="https://developers.redhat.com/blog/category/node-js/">Node.js</a> released its latest major version with <a href="https://nodejs.org/en/blog/release/v14.0.0/">Node.js 14</a>. Because this is an even-numbered release, it will become a Long Term Support (LTS) release in October 2020. This release brings a host of improvements and features, such as improved diagnostics, a V8 upgrade, an experimental Async Local Storage API, hardened the streams APIs, and more.

While Red Hat will release a <a href="https://developers.redhat.com/blog/category/ubi/">Universal Base Image (UBI)</a> for Node.js 14 in the coming months for <a href="https://developers.redhat.com/openshift">Red Hat OpenShift</a> and <a href="https://developers.redhat.com/topics/linux/">Red Hat Enterprise Linux</a>, this article helps you get started today. If you're interested in more about Node.js 14's improvements and new features, check out the article listed at the end. <!--more-->

Let's use a sample application that is based on the official <a href="https://nodejs.org/fr/docs/guides/nodejs-docker-webapp/#dockerizing-a-node-js-web-app"><em>How to Dockerize a Node.js Application</em> Nodejs.org docs</a>. This is a simple Express.js application with a Dockerfile using the latest upstream community Node.js 14 image.
<h2>How to deploy</h2>
First, use the <code>oc new-app</code> command with a Git repo that has a Dockerfile in it:

`$ oc new-app https://github.com/nodeshift-starters/basic-node-app-dockerized`

To access your application, you need to expose it using this simple command:

`$ oc expose svc/basic-node-app-dockerized`

Or, you can use the <a href="https://www.npmjs.com/package/nodeshift">Nodeshift module</a> to deploy a local directory. Assuming that you cloned the project we used earlier, you can run this command:

`$ npx nodeshift --build.strategy=Docker --expose`

<h3>Wrap up</h3>
As you can see, using Node.js 14 on Red Hat OpenShift today is pretty simple. To learn more about the improvements and features in Node.js 14, check out the official <a href="https://medium.com/@nodejs/node-js-version-14-available-now-8170d384567e">Node.js blog post</a>.
