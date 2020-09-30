---
layout: post
title:  "Deploying serverless Node.js applications on Red Hat OpenShift, Part 1"
date:   2020-09-15
categories: Openshift Nodeshift Serverless Knative
cannonical_url: https://developers.redhat.com/blog/2020/09/15/deploying-serverless-node-js-applications-on-red-hat-openshift-part-1/
author: lholmquist
---

<a href="https://developers.redhat.com/topics/serverless-architecture">Red Hat OpenShift Serverless</a> recently became GA, and with it came new options for application deployment. This article introduces one of those new options, <a href="https://developers.redhat.com/topics/serverless-architecture">Knative Serving</a>. I provide an overview of OpenShift Serverless and Knative Serving, then show you how to deploy a <a href="https://developers.redhat.com/blog/category/node-js/">Node.js application</a> as a Knative Serving service.
<h2>What is OpenShift Serverless?</h2>
According to the <a href="https://www.openshift.com/blog/openshift-serverless-now-ga">OpenShift Serverless GA release</a>:
<blockquote><i>OpenShift Serverless enables developers to build what they want, when they want, with whatever tools and languages they need. Developers can quickly get their applications up and deployed using serverless compute, and they won't have to build and maintain larger container images to do so.</i></blockquote>
OpenShift Serverless is based on the <a href="https://knative.dev">Knative</a> open source <a href="https://developers.redhat.com/topics/kubernetes">Kubernetes</a> serverless project. While it has a few different parts, we will focus on deploying a serverless Node.js application as a Knative Serving service.

<!--more-->
<h2>Knative Serving</h2>
So, what is Knative Serving? The official OpenShift documentation has a <a href="https://docs.openshift.com/container-platform/4.5/serverless/serverless-getting-started.html">buzzword-filled section about it</a>, but we are most interested in the ability to scale to zero.

Applications running on OpenShift and Kubernetes run inside a <a href="https://developers.redhat.com/topics/containers/">container</a> or <i>pod</i>. An OpenShift pod needs to be <em>up</em> if we want users to be able to access our application. A containerized application deployed as a Knative Serving service can be <em>off</em> until a request comes in—that is what we mean by "scale to zero." When a request comes in, the application starts and begins receiving requests. Knative orchestrates all of this.
<h2>Getting started with Knative Serving</h2>
If you want to follow along with the example, you will need to have OpenShift Serverless installed on your OpenShift cluster. The OpenShift Serverless documentation has instructions for <a href="https://docs.openshift.com/container-platform/4.5/serverless/installing_serverless/installing-openshift-serverless.html">setting up OpenShift Serverless</a>, and for <a href="https://docs.openshift.com/container-platform/4.5/serverless/installing_serverless/installing-knative-serving.html">setting up Knative Serving</a>.

For local development, I use <a href="https://developers.redhat.com/products/codeready-containers/overview">Red Hat CodeReady Containers</a> (CRC) to run OpenShift locally. Note that CRC with OpenShift Serverless installed can be a little memory intensive.
<h3>Deploying the Node.js application</h3>
The example in the <a href="https://docs.openshift.com/container-platform/4.3/applications/application_life_cycle_management/odc-creating-applications-using-developer-perspective.html">OpenShift documentation</a> shows how to use a Git repository, hosted on GitHub, to deploy an application as a Knative Serving service. That's fine, but if I'm in development and coding on my laptop, I don't want to have to push my changes to GitHub just to see my application running.

Another option is to use an already built image to create a Knative Serving service. The YAML for that service might look something like this:
<pre>apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: docker.io/openshift/hello-openshift
          env:
            - name: RESPONSE
              value: "Hello Serverless!"
</pre>
But again, this example shows an image being hosted on Docker Hub, which brings up the same predicament as deploying from GitHub.

For local development, I prefer using the Nodeshift module. I've <a href="https://developers.redhat.com/blog/2019/08/30/easily-deploy-node-js-applications-to-red-hat-openshift-using-nodeshift/">introduced Nodeshift</a> elsewhere, so I won't write much about it here.
<h3>The Node.js example application</h3>
For this example, I'll use an application that I've used before, a basic <a href="https://github.com/nodeshift-starters/nodejs-rest-http">REST application</a> that is built with <a href="https://expressjs.com">Express.js</a>. As a refresher, the Express.js application has an input form that takes a name and sends it to a REST endpoint, which generates a greeting. When you pass in a name, it is appended to the greeting and sent back. To see the application running locally, enter the following command:
<pre>$ npm install &amp;&amp; npm start
</pre>
To deploy the Node.js application as a Knative service, we only have to call Nodeshift with the experimental <code>--knative</code> flag. The command would look something like this:
<pre>$ npx nodeshift --knative
</pre>
This command archives our source code and sends it to OpenShift, where a Source-to-Image (S2I) build results in an <code>ImageStream</code>. This is all standard Nodeshift stuff. Once the build has completed, Nodeshift creates a Knative service, which uses the <code>ImageStream</code> we've just built as its input. This procedure is similar to pulling an image from Docker Hub, but in this case, the image is stored in OpenShift's internal registry.
<h3>Run the application</h3>
We could use <code>oc</code> commands to see that our application is running, but it's easier to understand what is happening with something more visual. Let's use the OpenShift web console's new Topology view, as shown below

<img class="wp-image-737647 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/06/crc-nodejs-serverless-1024x522.png" alt="A screenshot of the serverless Node.js application in the OpenShift dashboard's Topology view." />

The application is deployed as a Knative service. Most likely, the blue circle (which indicates that a pod is running successfully) is not filled. Our app is currently scaled to zero and waiting for a request to come in before it starts up.

Clicking on the link icon in the top-right corner of the application opens it. This is the first time that we are accessing the app, so it takes a few seconds to load. Our application is now starting up. It's a basic Express.js application, so it starts quickly, as you can see below

<img class="wp-image-737657 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/06/nodejs-serverless-applicaiton-1024x515.png" alt="A screenshot of the Express.js application displayed in browser."/>

The application in the Topology view now has that familiar blue circle.

<img class="wp-image-737667 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/06/crc-nodejs-serverless-scaled-1024x526.png" alt="A screenshot of the OpenShift Topology view showing the application now scaled up." />

By default, after 300 seconds (5 minutes), the running pod terminates and scales back to zero. The next time that you access the application, the startup cycle will happen again.
<h2>Conclusion</h2>
In this article, I've shown you a small part of what OpenShift Serverless can do. In future articles, we'll look at more features and how they relate to Node.js. This article focused on deploying a Node.js app as a Knative Serving service, but you might have noticed that Knative and OpenShift Serverless don't care what type of application you use. In a future article, I'll discuss the things that you should consider when creating a Node.js application to be deployed as a serverless application.
