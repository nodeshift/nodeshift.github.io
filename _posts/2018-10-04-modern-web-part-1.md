---
layout: post
title:  "Modern web applications on OpenShift: Part 1 -- Web apps in two commands"
date:   2018-10-04
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/
author: lholmquist
---

In this multi-part series, we will take a look at how to deploy modern web applications, like React and Angular apps, to <a href="http://openshift.com/">Red Hat OpenShift</a> using a new source-to-image (S2I) builder image.

Series overview:
<ul>
  <li>Part 1: How to deploy modern web apps using the fewest steps</li>
  <li>Part 2: <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">How to combine this new S2I image with a current HTTP server image</a>, like NGINX, using an OpenShift chained build for a more production-ready deployment</li>
  <li>Part 3: How to run your app's development server on OpenShift while syncing with your local file system</li>
</ul>
<!--more-->
<h2>Some initial setup</h2>
If you want to follow along, there are some prerequisites. You'll need a running instance of OpenShift. I'll be using minishift which allows you to run OpenShift on your Windows, Mac, or Linux desktop in a VM. To get minishift, download <a href="https://developers.redhat.com/products/cdk/overview/">Red Hat Container Development Kit (CDK)</a>.  Follow<a href="https://developers.redhat.com/products/cdk/hello-world/"> these instructions</a> to install and getting minishift running. For more information see the <a href="https://developers.redhat.com/products/cdk/docs-and-apis/">CDK documentation</a>, and the <a href="https://docs.okd.io/latest/minishift/index.html">documentation on OKD.io</a>.

Once minishift is running, you need to make sure you are logged in and have a project set up, which you can do using code like this:
<pre>$ oc login

$ oc new-project web-apps
</pre>
I also assume you have Node.js 8+ and npm 5.2+ installed.

If all you want to see are the two commands, skip to the "Summary" section.
<h2>What is a modern web application?</h2>
Before we begin, we should probably define what exactly a modern web application is and how it differs from what I like to call a "pure" Node.js application.

To me, a modern web application is something like React, Angular, or Ember, where there is a build step that produces static HTML, JavaScript, and CSS. This build step usually does a few different tasks, like concatenation, transpilation (Babel or Typescript), and minifying of the files. Each of the major frameworks has its own build process and pipeline, but tools like Webpack, Grunt, and Gulp also fall into this category. No matter what tool is used, they all use Node.js to run the build processes.

But the static content that is generated (compiled) doesn't necessarily need a node process to serve it. Yes, you could use something like the <a href="https://www.npmjs.com/package/serve">serve module</a>, which is nice for development since you can see your site quickly, but for production deployments, it is usually recommend to use something like NGINX or Apache HTTP Server.

A "pure" node application, on the other hand, will use a Node.js process to run and can be something like an <a href="http://expressjs.com/">Express.js application</a> (that is, a REST API server), and there isn't usually a build step (I know, I know: Typescript is a thing now). Development dependencies are usually not installed since we only want the dependencies that the app uses to run.

To see an example of deploying a "pure" node app to OpenShift quickly using our Node.js S2I image, check out my post on <a href="https://developers.redhat.com/blog/2018/04/16/zero-express-openshift-3-commands/">deploying an Express.js application to OpenShift</a>.
<h2>Deploying a web app to OpenShift</h2>
Now that we understand the difference between a modern web application and a Node.js application, let's see how we go about getting our web app on OpenShift.

For this post, we will deploy both a React and a modern Angular application. We can create both projects pretty quickly using their respective CLI tools, <code>create-react-app</code> and<code> @angular/cli.</code> This will count as one of the two commands I referred to in the title.
<h3>React App</h3>
Let's start with the React application. If you have <code>create-react-app</code> installed globally, great. But if not, then you can run the command using <code>npx</code> like this:
<pre>$ npx create-react-app react-web-app

</pre>
<i>Note: npx is a tool that comes with npm 5.2+ to run one-off commands. Check out <a href="https://www.npmjs.com/package/npx">more here</a>.</i>

This command will create a new React app, and you should see something like this:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/09/create-react-app-success.png"><img class="aligncenter wp-image-521627 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/09/create-react-app-success-1024x353.png" alt="Screenshot of what you see after successfully creating a React app" width="640" height="221" /></a>

Assuming you are in the newly created project directory, you can now run the second command to deploy the app to OpenShift:
<pre>$ npx nodeshift --strictSSL=false --dockerImage=nodeshift/ubi8-s2i-web-app --imageTag=10.x --build.env YARN_ENABLED=true --expose
</pre>
Your OpenShift web console will look something like this:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/09/react-quick-running.png"><img class="aligncenter wp-image-521727 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/09/react-quick-running-1024x523.png" alt="Screenshot of OpenShift web console after deploying the React app" width="640" height="327" /></a>

And here's what the web console looks like when you run the application:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/09/react-on-openshift.png"><img class="aligncenter wp-image-521737 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/09/react-on-openshift-1024x566.png" alt="Screenshot of what the web console looks like when you run the React app" width="640" height="354" /></a>

Before we get into the Angular example, let's see what that last command was doing.

First, we see <code>npx nodeshift</code>. We are using npx to run the nodeshift module. As I've mentioned in previous posts, <a href="https://www.npmjs.com/package/nodeshift">nodeshift</a> is a module for easily deploying node apps to OpenShift.

Next, let's see what options are being passed to nodeshift. The first is <code>--strictSSL=false</code>. Since we are using minishift, which is using a self-signed certificate, we need to tell nodeshift (really, we are telling the request library, which is used under the covers), about this so a security error isn't thrown.

Next is <code>--dockerImage=nodeshift/ubi8-s2i-web-app --imageTag=10.x</code>. This tells nodeshift we want to use the new <a href="https://hub.docker.com/r/nodeshift/ubi8-s2i-web-app/">Web App Builder image</a> and we want to use its 10.x tag.

Next, we want to tell the S2I image that we want to use yarn: <code>--build.env YARN_ENABLED=true</code>. And finally, the <code>--expose</code> flag tells nodeshift to create an OpenShift route for us, so we can get a publicly available link to our application.

Since this is a "get on OpenShift quickly" post, the S2I image uses the <a href="https://www.npmjs.com/package/serve">serve module</a> to serve the generated static files. In a later post, we will see how to use this S2I image with NGINX.
<h3>Angular App</h3>
Now let's create an Angular application. First, we need to create our new application using the Angular CLI. Again, if you don't have it installed globally, you can run it with npx:
<pre>$ npx @angular/cli new angular-web-app
</pre>
This will create a new Angular project, and as with the React example, we can run another command to deploy:
<pre>$ npx nodeshift --strictSSL=false --dockerImage=nodeshift/ubi8-s2i-web-app --imageTag=10.x --build.env OUTPUT_DIR=dist/angular-web-app --expose
</pre>
Again, similar to the React application, your OpenShift web console will look something like this:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/09/angular-quick.png"><img class="aligncenter wp-image-521767 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/09/angular-quick-1024x520.png" alt="Screenshot of the OpenShift web console after deploying an Angular app" width="640" height="325" /></a>

And here's what the web console looks like when you run the application:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/09/angular-on-openshift.png"><img class="aligncenter wp-image-521777 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/09/angular-on-openshift-1024x561.png" alt="Screenshot of what the web console looks like when you run the Angular app" width="640" height="351" /></a>

Let's take a look at that command again. Even though it looks very similar to the command we used for the React app, there is are some very important differences.

The differences are with the <code>build.env</code> flag: <code>--build.env OUTPUT_DIR=dist/angular-web-app</code>. There are two things different here.

First, we removed the <code>YARN_ENABLED</code> variable, since we aren't using yarn for the Angular project.

The second is the addition of the <code>OUTPUT_DIR=dist/angular-web-app</code> variable. So, by default, the S2I image will look for your compiled code in the <code>build</code> directory. React uses <code>build</code> by default; that is why we didn't set it for that example. However, Angular uses something different for its compiled output. It uses <code>dist/&lt;PROJECT_NAME&gt;</code>, which in our case is <code>dist/angular-web-app</code>.
<h2>Summary</h2>
For those who skipped to this section to see the two commands to run, here they are:

<strong>React:</strong>
<pre>$ npx create-react-app react-web-app

$ npx nodeshift --strictSSL=false --dockerImage=nodeshift/ubi8-s2i-web-app --imageTag=10.x --build.env YARN_ENABLED=true --expose
</pre>
<strong>Angular:</strong>
<pre>$ npx @angular/cli new angular-web-app

$ npx nodeshift --strictSSL=false --dockerImage=nodeshift/ubi8-s2i-web-app --imageTag=10.x --build.env OUTPUT_DIR=dist/angular-web-app --expose
</pre>
<h2>Additional resources</h2>
In this article, we saw how quickly and easily we can deploy a modern web app to OpenShift using the new S2I Web App Builder image. The examples use the community version of the image, but very soon there will be an official <a href="https://developers.redhat.com/products/rhoar/overview/">Red Hat Openshift Application Runtime (RHOAR)</a> tech preview release. So watch out for that.

In the coming articles, we will take a deeper look at what the image is actually doing and how we can use more of its advanced features, as well as some advanced features of OpenShift, to deploy a more production-worthy application.

Read <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">part 2 of this series</a> to learn how to combine this new S2I image with a current HTTP server image like NGINX, using an OpenShift chained build for a more production-ready deployment.

Read <a href="https://developers.redhat.com/blog/2019/01/17/modern-web-applications-on-openshift-part-3-openshift-as-a-development-environment/">part 3 of this series</a> to see how you can run your app’s “development workflow” on OpenShift.

For more information, download the free ebook <em><a href="https://developers.redhat.com/books/deploying-openshift/">Deploying to OpenShift</a></em>.
