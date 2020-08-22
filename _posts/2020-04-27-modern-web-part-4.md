---
layout: post
title:  "Modern web applications on OpenShift, Part 4: Openshift Pipelines"
date:   2020-04-27
categories: Openshift Nodeshift
cannonical_url: https://developers.redhat.com/blog/2020/04/27/modern-web-applications-on-openshift-part-4-openshift-pipelines/
author: lholmquist
---

When I wrote part 3 of this series, <em><a href="https://developers.redhat.com/blog/2019/01/17/modern-web-applications-on-openshift-part-3-openshift-as-a-development-environment/">Modern web applications on OpenShift: Part 3 — OpenShift as a development environment</a></em>, I said that was the final part. However, there is new tech that fits in very nicely with deploying modern Web Applications to OpenShift, so part 4 is necessary. As a refresher, in the <a href="https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/">first article</a>, we looked at how to deploy a modern web application using the fewest commands. In the <a href="https://developers.redhat.com/blog/2018/10/23/modern-web-applications-on-openshift-part-2-using-chained-builds/">second part</a>, we took a deeper look into how the new source-to-image (S2I) web app builder works and how to use it as part of a chained build. In <a href="https://developers.redhat.com/blog/2019/01/17/modern-web-applications-on-openshift-part-3-openshift-as-a-development-environment/">the third</a>, we took a look at how to run your app's "development workflow" on <a href="https://developers.redhat.com/openshift/">Red Hat OpenShift</a>. This article talks about OpenShift Pipelines and how this tool can be used as an alternative to a chained build.

<!--more-->
<h2>What is OpenShift Pipelines?</h2>
OpenShift Pipelines are a cloud-native, continuous integration and delivery (CI/CD) solution for building pipelines using Tekton. Tekton is a flexible, Kubernetes-native, open source CI/CD framework that enables automating deployments across multiple platforms (Kubernetes, serverless, VMs, etc) by abstracting away the underlying details.

This post assumes some knowledge of Pipelines, so if you are new to this technology, check out this <a href="https://github.com/openshift/pipelines-tutorial">official tutorial first</a>.
<h2>Setting up your environment</h2>
To effectively follow this article, there is some initial setup:
<ol>
  <li>Set up an OpenShift 4 cluster: I've been using CodeReady Containers (CRD) to set up this environment (here are <a href="https://cloud.redhat.com/openshift/install/crc/installer-provisioned">the setup instructions</a>).</li>
  <li>Install the Pipeline Operator once your cluster is up and running, which involves only a couple of clicks (here is how to <a href="https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md" target="_blank" rel="noopener noreferrer">install the Operator</a>).</li>
  <li>Get the <a href="https://github.com/tektoncd/cli#installing-tkn">Tekton CLI (<code>tkn</code>) here</a>.</li>
  <li>Run the <code>create-react-app</code> CLI tool to create the application that we will eventually deploy (this <a href="https://github.com/nodeshift-starters/react-pipeline-example" target="_blank" rel="noopener noreferrer">basic React example</a>).</li>
  <li>(Optional) Clone the repo to run the example locally by running <code>npm install</code> and then <code>npm start</code>.</li>
</ol>
The application repo also has a <code>k8s</code> directory, which contains Kubernetes/OpenShift YAMLs that can be used to deploy the application. You can find the <code>Tasks</code>, <code>ClusterTasks</code>, <code>Resources</code> and <code>Pipelines</code> that we will be creating <a href="https://github.com/nodeshift/webapp-pipeline-tutorial">in this repo</a>.
<h2>Getting started</h2>
We first need to create a new project on our OpenShift cluster for this example. The new project will be called <code>webapp-pipeline</code>. Create this new project by calling:

`$ oc new-project webapp-pipeline`

The naming here is important for this tutorial, so if you decide to change it, pay attention to where I call out that the project name, and update accordingly. From here, this article will work backward. We will create all of the small pieces that make up the pipeline first, and then we will create the pipeline.

So, first up...
<h2>Tasks</h2>
Let's create a couple of <em>tasks</em> that can help us deploy our application later as part of our <em>pipeline</em>. The first task, <code>apply_manifests_task</code>, is responsible for applying the Kubernetes resource (service, deployment, and route) YAMLs that are in the applications k8s directory. The second task, <code>update_deployment_task</code>, is responsible for updating the deployed image with the new image our pipeline creates.

Don't worry too much about these right now. These tasks are more like utilities, and we will see them in a little while. For now, create these tasks with:
<pre>$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/tasks/update_deployment_task.yaml
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/tasks/apply_manifests_task.yaml
</pre>
You can then use the <code>tkn</code> CLI to make sure they were created:
<pre>$ tkn task ls

NAME                AGE
apply-manifests     1 minute ago
update-deployment   1 minute ago
</pre>
<strong>Note:</strong> These tasks are local to your current project.
<h2>Cluster tasks</h2>
A cluster task is pretty much the same thing as a task. It is still a reusable collection of steps that, when combined, perform a specific task, except that cluster task is available to the whole cluster. To see a list of the cluster tasks that came pre-installed when you added the pipeline Operator, use the <code>tkn</code> CLI again:
<pre>$ tkn clustertask ls

NAME                       AGE
buildah                    1 day ago
buildah-v0-10-0            1 day ago
jib-maven                  1 day ago
kn                         1 day ago
maven                      1 day ago
openshift-client           1 day ago
openshift-client-v0-10-0   1 day ago
s2i                        1 day ago
s2i-go                     1 day ago
s2i-go-v0-10-0             1 day ago
s2i-java-11                1 day ago
s2i-java-11-v0-10-0        1 day ago
s2i-java-8                 1 day ago
s2i-java-8-v0-10-0         1 day ago
s2i-nodejs                 1 day ago
s2i-nodejs-v0-10-0         1 day ago
s2i-perl                   1 day ago
s2i-perl-v0-10-0           1 day ago
s2i-php                    1 day ago
s2i-php-v0-10-0            1 day ago
s2i-python-3               1 day ago
s2i-python-3-v0-10-0       1 day ago
s2i-ruby                   1 day ago
s2i-ruby-v0-10-0           1 day ago
s2i-v0-10-0                1 day ago
</pre>
We will now create two cluster tasks. The first creates an S2I image and pushes it into the internal OpenShift registry, and the second builds our NGINX-based image using the contents of our built application
<h3>Creating and pushing the image</h3>
For the first cluster task, we follow the part of the same process that we used in the previous article about chain builds. There, we used an S2I image (<code>ubi8-s2i-web-app</code>) to "build" our web application. This action resulted in an image that was stored in the internal OpenShift registry. We will use that web-app S2I image to create a <code>DockerFile</code> for our application and then use Buildah to actually build and push that image into the internal OpenShift registry: This is actually what OpenShift does if you deploy your applications with NodeShift.

If you're wondering how I knew that I needed all of those steps, the answer is that I didn't. I copied <a href="https://github.com/openshift/pipelines-catalog/blob/master/s2i-nodejs/s2i-nodejs-task.yaml">the official Node.js version</a> and updated it for my needs.

Now, create the <code>s2i-web-app</code> cluster task:

`$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/clustertasks/s2i-web-app-task.yaml`

I won't go into each of the entries in that file but I do want to point out a particular parameter: <code>OUTPUT_DIR</code>:
<pre>params:
      - name: OUTPUT_DIR
        description: The location of the build output directory
        default: build
</pre>
This parameter defaults to <code>build</code>, which is where React puts its built content. Different frameworks could have different locations. For example, Ember uses <code>dist</code>. The output of this first cluster task will be an image that contains our built HTML, JavaScript, and CSS.
<h3>Build the NGINX-based image</h3>
For the second cluster task, we need to create the task that builds our NGINX-based image using the contents of our built application. This is basically the chained build part of the aforementioned article.

To do this, create the <code>webapp-build-runtime</code> cluster task the same as with the other one:

`$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/clustertasks/webapp-build-runtime-task.yaml`

If you looked at the code for these cluster tasks, you would notice that we are not specifying what Git repo we are working with, or what image names we are creating. We only specify that we are passing in a Git repo or an image, for example, and that we are outputting an image. This process allows us to reuse these cluster tasks with different applications.

This leads us nicely into...
<h2>Resources</h2>
Since we just learned that our cluster tasks are meant to be generic as possible, we need to create resources to use as inputs (the Git repo) and outputs (the resulting images). The first resource we need is the Git repo where our application is. This resource can look something like this:
<pre># This resource is the location of the git repo with the web application source
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: web-application-repo
spec:
  type: git
  params:
    - name: url
      value: https://github.com/nodeshift-starters/react-pipeline-example
    - name: revision
      value: master
</pre>
This <code>PipelineResource</code> is of the <code>git</code> type. We can see in the params section that the <code>url</code> targets a specific repo and that we also specify the master branch (this is optional, but I'm including it for completeness).

The next resource we need is an image where we will store the result of the <code>s2i-web-app</code> task. This process might look something like:
<pre># This resource is the result of running "npm run build",  the resulting built files will be located in /opt/app-root/output
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: built-web-application-image
spec:
  type: image
  params:
    - name: url
      value: image-registry.openshift-image-registry.svc:5000/webapp-pipeline/built-web-application:latest
</pre>
This <code>PipelineResource</code> is of the <code>image</code> type and the value of <code>url</code> points to the internal OpenShift Image Registry—specifically the one in the <code>webapp-pipeline</code> namespace. If you are using a different namespace, then change that value accordingly.

The last resource will also be an <code>image</code> type, and this will be the resulting NGINX image that we will eventually deploy:
<pre># This resource is the image that will be just the static html, css, js files being run with nginx
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: runtime-web-application-image
spec:
  type: image
  params:
    - name: url
      value: image-registry.openshift-image-registry.svc:5000/webapp-pipeline/runtime-web-application:latest
</pre>
Again, this resource will store the image inside the internal OpenShift registry in the <code>webapp-pipeline</code> namespace.

To create all of these resources at once, run this <code>create</code> command:
<pre>$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/resources/resource.yaml
</pre>
You can then see the created resources using:
<pre>$ tkn resource ls
</pre>
<h2>The pipeline</h2>
Now that we have all of the pieces, let's put them together in our pipeline. You can create the pipeline by running the following command:
<pre>$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/pipelines/build-and-deploy-react.yaml
</pre>
Before we run this command, let's take a look at the pieces. First, the name:
<pre>apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: build-and-deploy-react
</pre>
Then, in the <code>spec</code> section, we see specify the resources that we created earlier:
<pre>spec:
  resources:
    - name: web-application-repo
      type: git
    - name: built-web-application-image
      type: image
    - name: runtime-web-application-image
      type: image
</pre>
We then create tasks for our pipeline to run. The first task we want to run is the <code>s2i-web-app</code> cluster task we created earlier:
<pre>tasks:
    - name: build-web-application
      taskRef:
        name: s2i-web-app
        kind: ClusterTask
</pre>
This task takes an input (which is the <code>git</code> resource) and an output (which is the <code>built-web-application-image</code> resource). We also pass a parameter to tell our cluster task that we don't need to verify TLS because we are using self-signed certificates:
<pre>resources:
        inputs:
          - name: source
            resource: web-application-repo
        outputs:
          - name: image
            resource: built-web-application-image
      params:
        - name: TLSVERIFY
          value: "false"
</pre>
The next task has a similar setup, but this time calls the <code>webapp-build-runtime</code> cluster task we created earlier:
<pre>name: build-runtime-image
    taskRef:
      name: webapp-build-runtime
      kind: ClusterTask
</pre>
Similar to our previous task, we are passing a resource, but this time it is the <code>built-web-application-image</code> (this was the output of our previous task). Again, we specify an image as the output. This task should run after the previous task, so we add the <code>runAfter</code> field:
<pre>resources:
        inputs:
          - name: image
            resource: built-web-application-image
        outputs:
          - name: image
            resource: runtime-web-application-image
        params:
        - name: TLSVERIFY
          value: "false"
      runAfter:
        - build-web-application
</pre>
The next two tasks are responsible for applying the service, route, and deployment YAML files that live in the web application's <code>k8s</code> directory, and then updating the deployment with the newly created image. These are the two cluster tasks we defined in the beginning.
<h3>Run the pipeline</h3>
Now that all of the pieces are created, we can finally run our new pipeline with the following command:

`$ tkn pipeline start build-and-deploy-react`

At this point, the CLI will become interactive, and you will need to choose the appropriate resources at each prompt. For the <code>git</code> resource, choose <code>web-application-repo</code>. Then, choose <code>built-web-application-image</code> for the first image resource and <code>runtime-web-application-image</code> for the second image resource:

<pre>? Choose the git resource to use for web-application-repo: web-application-repo (https://github.com/nodeshift-starters/react-pipeline-example)
? Choose the image resource to use for built-web-application-image: built-web-application-image (image-registry.openshift-image-registry.svc:5000/webapp-pipeline/built-web-
application:latest)
? Choose the image resource to use for runtime-web-application-image: runtime-web-application-image (image-registry.openshift-image-registry.svc:5000/webapp-pipeline/runtim
e-web-application:latest)
Pipelinerun started: build-and-deploy-react-run-4xwsr
</pre>

Check the status of the pipeline by running:

`$ tkn pipeline logs -f`

After the pipeline has finished and the application is deployed, get the exposed route with this little command:
{% raw %}
`$ oc get route react-pipeline-example --template='http://{{.spec.host}}'`
{% endraw %}

For a more visual approach, we can look at our pipeline in the web console's <strong>Developer</strong> view and click the <b>Pipelines</b> section, as shown in Figure 1.

<img class="wp-image-693187 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/03/web-app-pipeline-web-console1-1024x621.png" alt="Pipeline overview" />

Click the running pipeline to see more detail, as shown in Figure 2.

<img class="wp-image-693177 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/03/web-app-pipeline-web-console-2-1024x610.png" alt="Pipeline detail"/>

Once you have the details, you can see your running application in the <strong>Topology</strong> view, as shown in Figure 3.

<img class="size-large wp-image-693207" src="https://developers.redhat.com/blog/wp-content/uploads/2020/03/web-app-pipeline-web-console-3-1024x612.png" alt="Web App Topology View"/>

Clicking the icon on the circle's top right opens the application. Figure 4 shows what this will look like.

<img class="wp-image-693217 size-large" src="https://developers.redhat.com/blog/wp-content/uploads/2020/03/web-app-pipeline-web-console-4-1024x617.png" alt="Running React Application" />

<h3>Wrapping up</h3>
In this article, we saw how to mimic the chained-build template approach with the use of OpenShift Pipelines. You can find all of the things we created <a href="https://github.com/nodeshift/webapp-pipeline-tutorial">in this repo</a>. There are also some GitHub issues there for adding an example with another framework besides React, so feel free to contribute!
