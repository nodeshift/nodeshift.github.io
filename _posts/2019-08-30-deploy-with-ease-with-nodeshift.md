---
layout: post
title:  "Easily deploy Node.js applications to Red Hat OpenShift using Nodeshift"
date:   2019-08-30
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2019/08/30/easily-deploy-node-js-applications-to-red-hat-openshift-using-nodeshift/
author: lholmquist
---

I recently wrote articles on <a href="https://developers.redhat.com/blog/2018/04/16/zero-express-openshift-3-commands/">deploying an Express.js application to OpenShift</a>, <a href="https://developers.redhat.com/blog/2018/05/15/debug-your-node-js-application-on-openshift-with-chrome-devtools/" target="_blank" rel="noopener noreferrer">how to debug your Node.js application on OpenShift with Chrome Dev Tools</a> and a short series on <a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">deploying modern web applications to OpenShift</a>. All of those articles used a node module called <a href="https://www.npmjs.com/package/nodeshift" target="_blank" rel="noopener noreferrer">Nodeshift</a>, but I did a Jedi, hand-wavy thing when talking about it. This next series of articles takes a deeper look at what Nodeshift is and how it is used to ease the deployment of Node.js apps to OpenShift during development.<!--more-->
<h2>Basic app deployment on Red Hat OpenShift</h2>
Although there are different approaches to how one deploys an application to <a href="https://developers.redhat.com/openshift/">Red Hat OpenShift</a>, we will look at the workflow I like to use. This specific workflow uses Source-to-Image (S2I) images and source code that is located on my local machine. Before we take a look at Nodeshift, though, let's first take a quick look at some of the parts that this workflow uses. This flow can logically be broken into two parts: the <strong>Build Phase</strong> and the <strong>Deploy Phase</strong>.
<h3>Part 1: The Build Phase</h3>
The first phase of this workflow is all about building an image to eventually run in the Deploy phase. For our Node.js app, this is the phase where we install our dependencies and run any build scripts. If you are familiar with the phases of S2I, this phase is where the assemble script runs.

Using a BuildConfig, we can specify where our code comes from and what type of strategy to use when building the code. In our case, we use the DockerImage strategy since we are using a Node.js S2I image. The BuildConfig also tells OpenShift where to put our built code when it is done: In our case, an ImageStream.

Initially, we create an empty ImageStream, and then we populate that with the results of a successful build. In fact, if you were to look at OpenShift's internal image registry you would see that image there, similar to how you would see a container image on your local machine when running something like <code>docker images</code>.
<h3>Part 2: The Deploy Phase</h3>
The second phase of this workflow is all about running our application and setting it up to be accessed. For our Node.js app, this is the phase where we might run something like <code>npm run start</code> to launch our application. Again, if you are familiar with the phases of S2I, this phase is where the run script runs. By default, the Node.js S2I image that we use here this same command: <code>npm run start</code>.

Using a DeploymentConfig, we can then trigger the S2I run phase. DeploymentConfigs are also used to describe our application (what ImageStream to use, any environment variables, setting up health checks, and so on). Once a Deployment is successful, a running Pod is created.

Next, we need a Service for the new Pod's internal load balancing, as well as a Route if we want to access our application outside of the OpenShift context.

While this workflow is not too complicated, there are many different pieces that work together. Those pieces are also YAML files, which at times can be difficult to read and interpret.
<h2>Nodeshift basics</h2>
Now that we have a little background on deploying applications to OpenShift, let's talk about Nodeshift and what it is. According to the Nodeshift module readme:
<blockquote>Nodeshift is an opinionated command-line application and programmable API that you can use to deploy Node.js projects to OpenShift.</blockquote>
The opinion that Nodeshift takes is the workflow that I've just described, which allows the user to develop their application and deploy it to OpenShift, without having to think about all those different YAML files.

Nodeshift is also written in Node.js, so it can fit into a Node developer's current workflow or be added to an existing project using <code>npm install</code>. The only real prerequisite is that you are logged into your OpenShift cluster using <code>oc login</code>, but that isn't really a requirement. You can also specify an external config file, which we will see in a later article about more advanced usage.
<h3>Running Nodeshift</h3>
Using Nodeshift on the command line is easy. You can install it globally:
<pre>$ npm install -g nodeshift

$ nodeshift --help
</pre>
or by using <code><a href="https://www.npmjs.com/package/npx">npx</a></code>, which is the preferred way:
<pre>$ npx nodeshift --help
</pre>
As is the case with every other command-line tool, running Nodeshift with that <code>--help</code> flag shows us the commands and flags that are available to use:
<pre>Commands:
  nodeshift deploy                default command - deploy             [default]
  nodeshift build                 build command
  nodeshift resource              resource command
  nodeshift apply-resource        apply resource command
  nodeshift undeploy [removeAll]  undeploy resources

Options:
  --help                   Show help                                   [boolean]
  --version                Show version number                         [boolean]
  --projectLocation        change the default location of the project   [string]
  --configLocation         change the default location of the config    [string]
  --dockerImage            the s2i image to use, defaults to
                           nodeshift/centos7-s2i-nodejs                 [string]
  --imageTag               The tag of the docker image to use for the deployed
                           application.             [string] [default: "latest"]
  --outputImageStream      The name of the ImageStream to output to.  Defaults
                           to project name from package.json            [string]
  --outputImageStreamTag   The tag of the ImageStream to output to.     [string]
  --quiet                  supress INFO and TRACE lines from output logs
                                                                       [boolean]
  --expose                 flag to create a default Route and expose the default
                           service
                               [boolean] [choices: true, false] [default: false]
  --namespace.displayName  flag to specify the project namespace display name to
                           build/deploy into.  Overwrites any namespace settings
                           in your OpenShift or Kubernetes configuration files
                                                                        [string]
  --namespace.create       flag to create the namespace if it does not exist.
                           Only applicable for the build and deploy command.
                           Must be used with namespace.name            [boolean]
  --namespace.remove       flag to remove the user created namespace.  Only
                           applicable for the undeploy command.  Must be used
                           with namespace.name                         [boolean]
  --namespace.name         flag to specify the project namespace name to
                           build/deploy into.  Overwrites any namespace settings
                           in your OpenShift or Kubernetes configuration files
                                                                        [string]
  --deploy.port            flag to update the default ports on the resource
                           files. Defaults to 8080               [default: 8080]
  --build.recreate         flag to recreate a buildConfig or Imagestream
           [choices: "buildConfig", "imageStream", false, true] [default: false]
  --build.forcePull        flag to make your BuildConfig always pull a new image
                           from dockerhub or not
                               [boolean] [choices: true, false] [default: false]
  --build.incremental      flag to perform incremental builds, which means it
                           reuses artifacts from previously-built images
                               [boolean] [choices: true, false] [default: false]
  --metadata.out           determines what should be done with the response
                           metadata from OpenShift
        [string] [choices: "stdout", "ignore", ""] [default: "ignore"]
  --cmd                                                      [default: "deploy"]
</pre>
Let's take a look at the most common usage.
<h3>Deploying Nodeshift</h3>
Let's say we have a simple express.js application that we have been working on locally, which we've bound to port 8080, and we want to deploy this application to OpenShift. We just run:
<pre>  $ npx nodeshift
</pre>
Once that command runs, Nodeshift goes to work. Here are the steps that the command goes through using the default deploy command:
<ol>
  <li>Nodeshift packages your source code into a <code>tar</code> file to upload to the OpenShift cluster.</li>
  <li>Nodeshift looks at the <code>files</code> property of your application's <code>package.json</code> (by default, it ignores any <code>node_modules</code>, <code>tmp</code>, or <code>.git</code> folders):
<ul>
  <li>If a <code>files</code> property exists, Nodeshift uses <code>tar</code> to archive those files.</li>
  <li>If there is no <code>files</code> property, Nodeshift archives the current directory.</li>
</ul>
</li>
  <li>Once the archive is created, a new BuildConfig and ImageStream are created on the remote cluster.</li>
  <li>The archive is uploaded.</li>
  <li>An OpenShift Build starts running on OpenShift.</li>
  <li>Nodeshift watches that build process and outputs the remote log to the console.</li>
  <li>Once the build is completed, Nodeshift then creates a DeploymentConfig, which triggers an actual deployment, and also a Kubernetes Service. (A Route is not created by default, but if one is desired, you can use the <code>--expose</code> flag.)</li>
</ol>
If you make code changes and run the <code>nodeshift</code> command again, the process happens again, but this time it uses the existing config files that were created on the first run.
<h2>Until next time</h2>
In this article, we looked at the anatomy of a Red Hat OpenShift deployment and how Nodeshift can help abstract the complexity with a simple example. Stay tuned for future articles, in which we will look at other commands that Nodeshift provides. In those articles, we will explore several commonly used options and show how to use Nodeshift in our code instead of just using it at the command line.
