---
layout: post
title:  "Zero to Express on OpenShift in Three Commands"
date:   2018-04-16
categories: Openshift Nodeshift
author: lholmquist
---

<strong>(Edit: November 22, 2019) The Node images used in this post, both community <code>centos7</code> and <code>product</code>, are no longer being updated and maintained. For community images, please use the Universal Base Image (UBI)-based node images located here: <a href="http://registry.access.redhat.com/ubi8/nodejs-10">registry.access.redhat.com/ubi8/nodejs-10</a></strong>

<strong>For a fully supported Product version of Node.js, please check out the Red Hat Software Collections Node.js image, <a href="https://access.redhat.com/containers/#/registry.access.redhat.com/rhscl/nodejs-10-rhel7">RH SCL Node.js</a>.</strong>

With the recent <a href="https://developers.redhat.com/blog/2018/03/12/rhoar-nodejs-annoucement/" target="_blank" rel="noopener noreferrer">announcement that Node.js is generally available as part of Red Hat OpenShift Application Runtimes,</a> I wanted to see how easy it was to deploy an <a href="https://expressjs.com/">Express.js</a> app on OpenShift.
<h3>Getting Started</h3>
Before we start, there are some required prerequisites. You need to have <a href="https://nodejs.org" target="_blank" rel="noopener noreferrer">Node 8.x</a> and <a href="https://www.npmjs.com/" target="_blank" rel="noopener noreferrer">npm 5.2 </a> or greater installed. npm comes with the official node distribution, so if you install Node from <a href="https://nodejs.org" target="_blank" rel="noopener noreferrer">Nodejs.org</a>, you should be good.

You'll also need access to an OpenShift environment or the Red Hat Container Development Kit (CDK) minishift environment. For this example, I'll be using minishift. You can find instructions on getting minishift up and running <a href="https://developers.redhat.com/products/cdk/hello-world/" target="_blank" rel="noopener noreferrer">here</a>. For my local minishift, I start it with this command:
<pre>$ minishift start --memory=6144 --vm-driver virtualbox</pre>
You also need to be logged in to whatever OpenShift cluster you are using (OpenShift or minishift) using <code>oc login</code>.
<h3>Spoiler Alert</h3>
For those who don't want to read the whole post and don't want to scroll to the end, here are the three commands that need to be run:
<pre>$ npx express-generator .</pre>
<pre>$ npx json -I -f package.json -e 'this.scripts.start="PORT=8080 node ./bin/www"'</pre>
<pre>$ npx nodeshift --strictSSL=false --expose</pre>
<h3>Generate an Express App</h3>
What is Express, you say? Well, according to the <a href="https://expressjs.com/" target="_blank" rel="noopener noreferrer">Express website</a>, Express is a "Fast, unopinionated, minimalist web framework for Node.js."

One pretty cool thing about Express is the <i>Express application generator tool</i>: <code>express-generator</code>. This is a command-line tool that <a href="https://expressjs.com/en/starter/generator.html">"quickly creates an application skeleton"</a>. But wait: didn't I just say that Express was unopinionated? It is, but this is the opinionated skeleton creator. ¯_(ツ)_/¯

The Express website recommends installing the <code>express-generator</code> module globally, like this:
<pre>npm install -g express-generator</pre>
But we aren't going to do that. Instead, we are going to use a fairly new feature from npm, called <code>npx</code>.

<code>npx</code> gives us the ability to run one-off commands with out having to install things globally. There is more to <code>npx</code> that just that feature, so if you are interested in all the cool things <code>npx</code> can do, check it out <a href="https://medium.com/@maybekatz/introducing-npx-an-npm-package-runner-55f7d4bd282b">here</a>.

With this new-found knowledge, we can now generate our Express app like this:
<pre>$ npx express-generator .</pre>
Let's take a quick look at what is actually happening with this command. First, <code>npx</code> sees that we want to run the <code>express-generator</code> command, so <code>npx</code> does some magic to see if we have it installed locally (in our current directory), and then it checks our global modules. Because it is not there, it downloads it for this one-time use.

<code>express-generator</code> is run in our current directory, which is denoted by that <strong>.</strong> at the end of the command.

The result should look something like this:

<img class="alignnone size-large wp-image-475687" src="https://developers.redhat.com/blog/wp-content/uploads/2018/03/express-quick-example-screenshot-1-1024x524.png" alt="" />

<code>express-generator</code> also gives us some instructions on how to install the dependencies and then how to run the application. You can skip that for now.
<h3>Update the package.json File</h3>
Now that we created our basic Express application using one command, we need to add one thing to <code>package.json</code> before we deploy our app.

We need to pass a <code>PORT</code> environment variable to our start script.

One way to do this is to open a text editor and do it that way, but that would add a few more steps. To do this in one command, we can use the <a href="https://www.npmjs.com/package/json" target="_blank" rel="noopener noreferrer">json module</a>.
<pre>$ npx json -I -f package.json -e 'this.scripts.start="PORT=8080 node ./bin/www"'</pre>
As before, we are using the <code>npx</code> command to allow us to not have to install the <code>json</code> module globally.

Let's see what is going on with the options passed to the <code>json</code> module.

<code>-I -f package.json</code> means that we want to edit in place the file <code>package.json</code>. The <code>-e</code> option will execute some JavaScript code, which in this case is setting the <code>scripts.start</code> property from <code>package.json</code> with the string <code>"PORT=8080 node ./bin/www"</code>.

For more information on the <code>json</code> module, check out the <a href="http://trentm.com/json/">documentation</a>.
<h3>Deploy the Application to OpenShift</h3>
And now, the final step is to run this command:
<pre>$ npx nodeshift --strictSSL=false --expose</pre>
Here, we are using the <a href="https://www.npmjs.com/package/nodeshift" target="_blank" rel="noopener noreferrer">nodeshift module</a> to deploy our application. <code>nodeshift</code> is a CLI or programmable API that helps with deploying Node apps to OpenShift.

<code>npx</code> is doing the same thing as in the previous examples.

<code>nodeshift</code> is using two flags. The first, <code>strictSSL=false</code>, is needed when deploying to minishift or someplace that is using a self-signed certificate. If we were deploying to a real OpenShift cluster, we could leave that out.

The second flag, <code>expose</code>, tells <code>nodeshift</code> that it should create a <a href="https://docs.openshift.com/online/architecture/networking/routes.html"><em>route</em></a> for us, which allows our application to be seen by the outside world. (If you are running minishift locally, only you can see the application.)

The output of this command will look something like this:

<img class="alignnone size-large wp-image-475707" src="https://developers.redhat.com/blog/wp-content/uploads/2018/03/express-quick-example-nodeshift-output-1024x633.png" alt="" />

If we head over to the web UI of our running minishift, we can see that the created pod is now running successfully.

<img class="alignnone size-large wp-image-475727" src="https://developers.redhat.com/blog/wp-content/uploads/2018/03/express-quick-example-ui-1024x518.png" alt="" />

Then, if we click the link, we can see our example app running:

<img class="alignnone size-full wp-image-475737" src="https://developers.redhat.com/blog/wp-content/uploads/2018/03/express-quick-example-express.png" alt="" />

<strong>Note:</strong> The example above will use the latest <a href="https://hub.docker.com/r/bucharestgold/centos7-s2i-nodejs/">community s2i images</a> (9.x at the time of this writing). To use a fully supported version of Node.js on OpenShift all you need is to add the "--dockerImage" flag.

This will integrate the Red Hat OpenShift Application Runtime version Node.js (8.x) which you can get full production and developer support as part of our product subscription.

This might look something like this:
<pre>$ npx nodeshift --strictSSL=false --expose --dockerImage=registry.access.redhat.com/rhoar-nodejs/nodejs-8</pre>
<h3>Recap</h3>
In this post, the commands were a little spread out, so let's see them all together again:
<pre>$ npx express-generator .</pre>
<pre>$ npx json -I -f package.json -e 'this.scripts.start="PORT=8080 node ./bin/www"'</pre>
<pre>$ npx nodeshift --strictSSL=false --expose</pre>
The example app we created was very simple, but it shows how quickly you can get started using Node.js on OpenShift.
