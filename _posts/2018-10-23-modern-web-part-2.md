---
layout: post
title:  "Modern web applications on OpenShift: Part 2 -- Using chained builds"
date:   2018-10-23
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/
author: lholmquist
---

<a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">In the previous article</a>, we took a quick look at a new source-to-image (S2I) builder image designed for building and deploying modern web applications on <a href="http://openshift.com/">OpenShift</a>. While the last article was focused on getting your app deployed quickly, this article will look at how to use the S2I image as a "pure" builder image and combine it with an OpenShift c<em>hained build</em>.

<!--more-->
<h2>Pure builder image</h2>
As mentioned in the previous post, most modern web apps now have a build step. Common workflows done in the build step are things like transpiling your code, concatenating multiple files, and minifying. Once these workflows are done, the resulting files, which are static HTML, JavaScript, and CSS, are put into an output folder. The location of the folder usually depends on the build tools you are using, but for something like React, the location is <code>./build</code> (more on this location in a minute).
<h3>Source-to-Image (S2I)</h3>
This post isn't going to go into the "what and how" of S2I; you can <a href="https://docs.okd.io/latest/architecture/core_concepts/builds_and_image_streams.html#source-build">read more here</a>, but we should understand two of the phases that happen in order to better understand what the Web App Builder image is doing.
<h4>Assemble phase</h4>
The assemble phase is very similar to what happens when running <code>docker build</code>. The result of this phase will be a new Docker image. This phase also happens when a build is run on OpenShift.

For the Web App Builder image, the <a href="https://github.com/bucharest-gold/centos7-s2i-web-app/blob/master/s2i/assemble#L47">assemble script</a> is responsible for installing your app's dependencies and running your build. By default, the builder image will use <code>npm run build</code>, but that can be overridden by providing an <code>NPM_BUILD</code> environment variable.

As I said before, the location of your "built" app depends on the build tools you are using. For example, React uses <code>./build</code>, but an Angular app uses <code>project_name/dist</code>. And, as you saw in the previous post, this output directory, which defaults to <code>build</code>, can be overridden using the <code>OUTPUT_DIR</code> environment variable. Since there are differences in output locations between frameworks, you copy the generated output into a common directory inside the image, <code>/opt/apt-root/output</code>. This will be important further down this post, but first let's take a quick look at the next phase, the run phase.
<h4>Run phase</h4>
This phase is run when <code>docker run</code> is called on the newly created image from the assemble phase. This is also what is run during an OpenShift deployment. By default, the <a href="https://github.com/bucharest-gold/centos7-s2i-web-app/blob/master/s2i/run">run script</a> will use the <a href="https://www.npmjs.com/package/serve">serve module</a> to serve the static content located in the common output directory mentioned above.

While this works for getting your app deployed quickly, it is not really the recommended way of serving static content. Since we are really serving only static content, we don't really even need Node.js installed in our image. We just need a web server.

This situation—where our building needs are different from our runtime needs—is where chained builds can help.
<h2>Chained builds</h2>
To quote the official OpenShift documentation on <a href="https://docs.okd.io/latest/dev_guide/builds/advanced_build_operations.html#dev-guide-chaining-builds">chained builds</a>:
<blockquote>"Two builds can be chained together: one that produces the compiled artifact, and a second build that places that artifact in a separate image that runs the artifact."</blockquote>
What this means is that we can use the Web App Builder image to run our build, and then we can use a web server image, like NGINX, to serve our content.

This allows us to use the Web App Builder image as a "pure" builder and also keep our runtime image small.

Let's take a look at an example to see how this all comes together.

This <a href="https://github.com/lholmquist/react-web-app">example app</a>, is a basic React application created using the <code>create-react-app</code> CLI tool.

I've added an <a href="https://github.com/lholmquist/react-web-app/blob/master/.openshiftio/application.yaml">OpenShift template file</a> to piece everything together.

Let's take a look at some of the more important parts of this file.
<pre>parameters:
  - name: SOURCE_REPOSITORY_URL
    description: The source URL for the application
    displayName: Source URL
    required: true
  - name: SOURCE_REPOSITORY_REF
    description: The branch name for the application
    displayName: Source Branch
    value: master
    required: true
  - name: SOURCE_REPOSITORY_DIR
    description: The location within the source repo of the application
    displayName: Source Directory
    value: .
    required: true
  - name: OUTPUT_DIR
    description: The location of the compiled static files from your web apps builder
    displayName: Output Directory
    value: build
    required: false
</pre>
The parameter section should be pretty self-explanatory, but I want to call out the <code>OUTPUT_DIR</code> parameter. For our React example, we don't need to worry about it, since the default value is what React uses, but if you are using Angular or something else, you could change it.

Now let's take a look at the image streams.
<pre>- apiVersion: v1
  kind: ImageStream
  metadata:
    name: react-web-app-builder  // 1
  spec: {}
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: react-web-app-runtime  // 2
  spec: {}
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: web-app-builder-runtime // 3
  spec:
    tags:
    - name: latest
      from:
        kind: DockerImage
        name: nodeshift/ubi8-s2i-web-app:10.x
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: nginx-image-runtime // 4
  spec:
    tags:
    - name: latest
      from:
        kind: DockerImage
        name: 'centos/nginx-112-centos7:latest'
</pre>
First, let's take a look at the third and fourth images. We can see that both are defined as Docker images, and we can see where they come from.

The third is the <code>web-app-builder</code> image, <code>nodeshift/ubi8-s2i-web-app</code>, which is using the 10.x tag from the <a href="https://hub.docker.com/r/nodeshift/ubi8-s2i-web-app/">Docker hub</a>.

The fourth is an NGINX image (version 1.12) using the latest tag from the <a href="https://hub.docker.com/r/centos/nginx-112-centos7/">Docker hub</a>.

Now let's take a look at those first two images. Both images are empty to start. These images will be created during the build phase, but for completeness, let me explain what will go into each one.

The first image, <code>react-web-app-builder</code>, will be the result of the "assemble" phase of the <code>web-app-builder-runtime</code> image once it is combined with our source code. That is why I've named it "<code>-builder</code>."

The second image, <code>react-web-app-runtime</code>, will be the result of combining the <code>nginx-image-runtime</code> with the some of the files from the <code>react-web-app-builder</code> image. This image will also be the image that is "deployed" and will contain only the web server and the static HTML, JavaScript, and CSS for the application.

This might sound a little confusing now, but once we look at the build configurations, things should be a little more clear.

In this template, there are two build configurations. Let's take a look at them one at a time.
<pre>  apiVersion: v1
  kind: BuildConfig
  metadata:
    name: react-web-app-builder
  spec:
    output:
      to:
        kind: ImageStreamTag
        name: react-web-app-builder:latest // 1
    source:   // 2
      git:
        uri: ${SOURCE_REPOSITORY_URL}
        ref: ${SOURCE_REPOSITORY_REF}
      contextDir: ${SOURCE_REPOSITORY_DIR}
      type: Git
    strategy:
      sourceStrategy:
        env:
          - name: OUTPUT_DIR // 3
            value: ${OUTPUT_DIR}
        from:
          kind: ImageStreamTag
          name: web-app-builder-runtime:latest // 4
        incremental: true // 5
      type: Source
    triggers: // 6
    - github:
        secret: ${GITHUB_WEBHOOK_SECRET}
      type: GitHub
    - type: ConfigChange
    - imageChange: {}
      type: ImageChange
</pre>
The first one, <code>react-web-app-builder</code> above, is pretty standard. We see that line 1 tells us the result of this build will be put into the <code>react-web-app-builder</code> image, which we saw when we took a look at the image stream list above.

Next, line 2 is just telling us where the code is coming from. In this case, it is a git repository, and the location, <code>ref</code>, and context directory are defined by the parameters we saw earlier.

Again, line 3, we saw in the <code>parameters</code> section. This will add the <code>OUTPUT_DIR</code> environment variable, which in our example will be <code>build</code>.

Line 4 is just telling us to use the <code>web-app-builder-runtime</code> image that we saw in the <code>ImageStream</code> section.

Line 5 is saying we want to use an incremental build if the S2I image supports it. The Web App Builder image does support it. On the first run, once the assemble phase is complete, the image will save the <code>node_modules</code> folder into an archive file. Then on subsequent runs, the image will un-archive that <code>node_modules</code> folder, which will speed up build times.

The last thing to call out, line 6, is just a few triggers that are set up, so when something changes, this build can be kicked off without manual interaction.

As I said before, this is a pretty standard build configuration. Now let's take a look at the second build configuration. Most of it is very similar to the first, but there is one important difference:
<pre>apiVersion: v1
  kind: BuildConfig
  metadata:
    name: react-web-app-runtime
  spec:
    output:
      to:
        kind: ImageStreamTag
        name: react-web-app-runtime:latest // 1
    source: // 2
      type: Image
      images:
        - from:
            kind: ImageStreamTag
            name: react-web-app-builder:latest // 3
          paths:
            - sourcePath: /opt/app-root/output/.  // 4
              destinationDir: .  // 5

    strategy: // 6
      sourceStrategy:
        from:
          kind: ImageStreamTag
          name: nginx-image-runtime:latest
        incremental: true
      type: Source
    triggers:
    - github:
        secret: ${GITHUB_WEBHOOK_SECRET}
      type: GitHub
    - type: ConfigChange
    - type: ImageChange
      imageChange: {}
    - type: ImageChange
      imageChange:
        from:
          kind: ImageStreamTag
          name: react-web-app-builder:latest // 7
</pre>
This second build configuration, <code>react-web-app-runtime</code>, starts off in a fairly standard way.

Line 1 isn't anything new. It is telling us that the result of this build will be put into the <code>react-web-app-runtime</code> image.

As with the first build configuration, we have a source section, line 2, but this time we say our source is coming from an image. The image that it is coming from is the one we just created, <code>react-web-app-builder</code> (specified in line 3). The files we want to use are located inside the image and that location is specified in line 4: <code>/opt/app-root/output/</code>. If you remember, this is where our generated files from our app's build step ended up.

The destination directory, specified in line 5, is just the current directory (this is all happening inside some magic OpenShift thing, not on your local computer).

The strategy section, line 6, is also similar to the first build configuration. This time, we are going to use the <code>nginx-image-runtime</code> that we looked at in the <code>ImageStream</code> section.

The final thing to point out is the trigger section, line 7, which will trigger this build anytime the <code>react-web-app-builder</code> image changes.

The rest of the template is fairly standard deployment configuration, service, and route stuff, which we don't need to go into. Note that the image that will be deployed will be the <code>react-web-app-runtime</code> image.
<h2>Deploying the application</h2>
Now that we've taken a look at the template, let's see how we can easily deploy this application.

We can use the OpenShift Client tool <code>oc</code> to deploy our template:
<pre>$ find . | grep openshiftio | grep application | xargs -n 1 oc apply -f

$ oc new-app --template react-web-app -p SOURCE_REPOSITORY_URL=https://github.com/lholmquist/react-web-app
</pre>
The first command above is just an overly engineered way of finding the <code>./openshiftio/application.yaml</code> template.

The second creates a new application based on that template.

Once those commands are run, we can see that there are two builds:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-build-2.png"><img class="aligncenter wp-image-526067 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-build-2-1024x521.png" alt="Screen showing the two builds" width="640" height="326" /></a>

Back on the Overview screen, we should see the running pod:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-overview.png"><img class="aligncenter wp-image-526077 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-overview-1024x513.png" alt="Screen showing the running pod" width="640" height="321" /></a>

Clicking the link should navigate to our application, which is the default React App page:

<a href="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-web.png"><img class="aligncenter wp-image-526087 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2018/10/react-web-app-web-1024x487.png" alt="Screen that is displayed after navigating to the app" width="640" height="304" /></a>
<h2>Extra Credit</h2>
For those who are into using Angular, here is an <a href="https://github.com/lholmquist/angular-web-app">example of that</a>.
The template is mostly the same, except for that <code>OUTPUT_DIR</code> variable.
<h2>Extra Extra Credit</h2>
This post showed how to use the NGINX image as our web server, but it's fairly easy to swap that out if you wanted to use an Apache server. It can actually be done in one or maybe two (for completeness) steps.

All you need to do is in the template file, swap out the <a href="https://github.com/lholmquist/react-web-app/blob/master/.openshiftio/application.yaml#L66">NGINX image</a> for the <a href="https://hub.docker.com/r/centos/httpd-24-centos7/">Apache image</a>.
<h2>Summary</h2>
While the first post in this series showed how to quickly get a modern web application on OpenShift, this post went deeper into what the Web App Builder image is doing and how to combine it, using a chained build, with a pure web server such as NGINX for a more production-ready build.

In the next and final (probably) post, we will take a look at how to run our web application's development server on OpenShift, while keeping our local and remote files in sync.
<h2>Series overview</h2>
<ul>
  <li>Part 1: <a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">How to deploy modern web apps using the fewest steps</a></li>
  <li>Part 2: <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">How to combine this new S2I image with a current HTTP server image</a>, like NGINX, using an OpenShift chained build for a more production-ready deployment</li>
  <li>Part 3: <a href="https://developers.redhat.com/blog/2019/01/17/modern-web-applications-on-openshift-part-3-openshift-as-a-development-environment/">How to run your app's development server on OpenShift while syncing with your local file system</a></li>
</ul>
<h2>Additional resources</h2>
<ul>
  <li><a href="https://developers.redhat.com/books/deploying-openshift/">Deploying to OpenShift: a guide for impatient developers</a>: free ebook</li>
  <li>Information on <a href="https://developers.redhat.com/topics/kubernetes/">OpenShift and Kubernetes</a></li>
</ul>
