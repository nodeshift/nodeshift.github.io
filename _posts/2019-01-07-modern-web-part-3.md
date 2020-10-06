---
layout: post
title:  "Modern web applications on OpenShift: Part 3 -- Openshift as a development environment"
date:   2019-01-07
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2019/01/17/modern-web-applications-on-openshift-part-3-openshift-as-a-development-environment/
author: lholmquist
---

Welcome back to the final part of this multipart series about deploying modern web applications on Red <a href="http://openshift.com/">Hat OpenShift</a>. In the <a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">first post</a>, we took a look at how to deploy a modern web application using the fewest commands.

In the <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">second part</a>, we took a deeper look into how the new source-to-image (S2I) web app builder works and how to use it as part of a chained build.

This third and final part will take a look at how you can run your app's "development workflow" on OpenShift.<!--more-->
<h2>Development workflow</h2>
As mentioned in the <a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">first post</a>, a common development workflow for modern web applications is to run a "development server" that watches your local files for changes. When a change occurs, the application's build is run and the browser is refreshed with your updated app.

Most of the modern frameworks have this "development server" built into their respective CLI tools.
<h3>A local example</h3>
Let's first start with running our application locally, so we can see how this workflow is supposed to work. We are going to continue with the <a href="https://github.com/lholmquist/react-web-app">React example</a> that we saw in the previous articles. Even though we are using React as an example here, the workflow concepts are very similar for all the other modern frameworks.

For this React example, to start the "development server" we run the following:
<pre>$ npm run start
</pre>
We should see something like this in our terminal:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-dev-server-local-1.png"><img class="aligncenter wp-image-528457 size-full" src="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-dev-server-local-1.png" alt="Starting the development server" width="572" height="253" /></a>

And our application should open in our default browser:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-localhost.png">
<img class="aligncenter wp-image-528467 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-localhost-1024x620.png" alt="Application running in a browser" width="640" height="388" /></a>

Now, if we make a change to a file, we should see the application running in the browser refresh with the latest changes.

As I said before, this is a common workflow for local development, but how can we get this workflow onto OpenShift?
<h3>Development server on OpenShift</h3>
In the <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">previous article</a>, we took a look at the run phase of the S2I image. We saw that the default way of serving our web app is with the <code>serve</code> module.

However, if we <a href="https://github.com/nodeshift/ubi8-s2i-web-app/blob/master/s2i/run#L10">look closely at that run script</a>, we can see that we can specify an environment variable, <code>$NPM_RUN</code>, which gives us the ability to execute a custom command.

For example, using the <code>nodeshift</code> module, the command to deploy our application might look something like this:
<pre>$ npx nodeshift --deploy.env NPM_RUN="yarn start" --dockerImage=nodeshift/ubi8-s2i-web-app
</pre>
<em>Note: The above example has been shortened to show an idea.</em>

Here we are adding the <code>NPM_RUN</code> environment variable to our deployment. This will tell our run phase to run <code>yarn start</code>, which starts the React development server inside our OpenShift pod.

If you took a look at the log of the running pod, you might see something like this running:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2019/01/react-pod-dev-server.png"><img class="aligncenter wp-image-553467 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2019/01/react-pod-dev-server-1024x550.png" alt="Log of the running pod" width="640" height="344" /></a>

Of course, this doesn't really matter unless we can sync our local code with the code that is being watched on our remote cluster.
<h3>Remote and local sync</h3>
Luckily, we can use <code>nodeshift</code> again to help us out. We can use the <code>watch</code> command.

After we run the command to deploy our application's development server, we can then run this command:
<pre>$ npx nodeshift watch
</pre>
This will connect to the running pod we just created and sync our local files with our remote cluster, while also watching our local system for changes.

So if you were to update the <code>src/App.js</code> file, that change will be detected and copied to the remote cluster, and the running development server will then refresh the browser.

For completeness, here are the full commands:
<pre>$ npx nodeshift --strictSSL=false --dockerImage=nodeshift/ubi8-s2i-web-app --build.env YARN_ENABLED=true --expose --deploy.env NPM_RUN="yarn start" --deploy.port 3000

$ npx nodeshift watch --strictSSL=false
</pre>
The <code>watch</code> command is an abstraction on top of the <code>oc rsync</code> command. To learn more about how that works, <a href="https://docs.okd.io/latest/dev_guide/copy_files_to_container.html" target="_blank" rel="noopener noreferrer">check it out here</a>.

Even though the example we saw was using React, this technique also works with other frameworks. You just need to change the <code>NPM_RUN</code> environment variable.
<h2>Conclusion</h2>
In this 3 part series, we saw how to deploy modern web applications to OpenShift in a few ways.

<a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/" target="_blank" rel="noopener noreferrer">In part one,</a> we saw how to get started quickly with the new Web App S2I Image.

<a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/" target="_blank" rel="noopener noreferrer">Part 2 dove a little deeper</a> into how the S2I image worked and how to use chained builds.

This last part was a brief overview of how you can run a development server on OpenShift, and the next talks about <a href="https://developers.redhat.com/blog/2020/04/27/modern-web-applications-on-openshift-part-4-openshift-pipelines/">OpenShift Pipelines and how this tool can be used as an alternative to a chained build</a>.
<h2>Additional resources</h2>
<ul>
  <li><a href="https://developers.redhat.com/books/deploying-openshift/">Deploying to OpenShift: a guide for impatient developers</a> (free ebook)</li>
  <li><a href="https://developers.redhat.com/blog/2018/06/11/container-native-nodejs-istio-rhoar/" rel="bookmark">Building Container-Native Node.js Applications with Red Hat OpenShift Application Runtimes and Istio</a></li>
  <li><a href="https://developers.redhat.com/blog/2018/05/15/debug-your-node-js-application-on-openshift-with-chrome-devtools/" rel="bookmark">How to Debug Your Node.js Application on OpenShift with Chrome DevTools</a></li>
  <li><a href="https://developers.redhat.com/blog/2018/04/16/zero-express-openshift-3-commands/" rel="bookmark">Zero to Express on OpenShift in Three Commands</a></li>
  <li><a href="https://developers.redhat.com/blog/2018/03/12/rhoar-nodejs-annoucement/" rel="bookmark">Announcing: Node.js General Availability in Red Hat OpenShift Application Runtimes</a></li>
  <li><a href="https://developers.redhat.com/blog/2018/12/21/monitoring-node-js-applications-on-openshift-with-prometheus/" rel="bookmark">Monitoring Node.js Applications on OpenShift with Prometheus</a></li>
  <li>Other articles on <a href="https://developers.redhat.com/topics/kubernetes/">OpenShift and Kubernetes</a></li>
</ul>
