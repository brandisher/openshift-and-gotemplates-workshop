# Customizing OC Output With Go Templates
This is a self-led workshop to demonstrate the power of Go templates and how much fun you can have while using them with OpenShift! Each section builds upon the previous section, but if you have some Go template knowledge already, then you should be able to jump around without issue.

### Table of Contents
* [Prerequisites](#prerequisites)
* [What is a Go template?](#what-is-a-go-template)
* [Lesson 1: A simple example](#lesson-1-a-simple-example)
* [Lesson 2: A practical example](#lesson-2-a-practical-example)
* [Lesson 3: Using a Go Template file](#lesson-3-using-a-go-template-file)
* [Lesson 4: Adding more context](#lesson-4-adding-more-context)
* [Lesson 5: Go Template nesting and modularity](#lesson-5-go-template-nesting-and-modularity)
* [Lesson 6: Conditionals](#lesson-6-conditionals)
* [Lesson 7: Functions](#lesson-7-functions)
* [Beyond the Workshop](#beyond-the-workshop)
* [Use Case: Identifying Tainted Nodes](#use-case-identifying-tainted-nodes)
* [Use Case: Error and Warning Event Dashboard](#use-case-error-and-warning-event-dashboard)

### Prerequisites
* Access to an OpenShift cluster. Some lessons may require cluster-admin permissions.
* A project for the workshop. We'll be using a project named `workshop` with the `httpd-example` template deployed, but you can use whatever project you would like that has running workloads.
* `oc` in your `$PATH`

### What Is a Go Template?
Go template is a format used by the Go programming language. The documentation for Go templates can be found [here](https://golang.org/pkg/text/template/). Keep this handy as a reference as we progress through the workshop. In OpenShift, Go templates are very helpful for encapsulating output logic and customizing the view that you get back from the API.

### Lesson 1: A Simple Example
The simplest example of a Go template is essentially a non-operation where we just print out a string that we pass to the template engine:
```
[user@machine]$ oc get pods -o go-template='Hello, World!'
Hello, World![user@machine]$
```
Here is where we can take our first step into leveraging the formatting features of Go templates. You will notice that there is no newline after `Hello, World!` in the previous example; let's add a newline to put our prompt on a fresh line:
```
[user@machine]$ oc get pods -o go-template='Hello, World!{% raw %}{{"\n"}}{% endraw %}'
Hello, World!
[user@machine]$
```
Much better! This is a good opportunity to explain what we did and what is happening under the covers. First, the double open and close braces `{% raw %}{{ }}{% endraw %}` tell the Go template engine some processing needs to happen. The `"\n"` tells the Go template engine to print a newline character at the end of `Hello, World!`. Since everything not inside `{% raw %}{{ }}{% endraw %}` is treated as a string literal, if we left off the double quotes from the newline, or simply put `\n` after `Hello, World!`, we would get an error or output that does not achieve our goal:
```
# No braces
[user@machine]$ oc get pods -o go-template='Hello, World!"\n"'
Hello, World!"\n"[user@machine]$

# No double quotes
[user@machine]$ oc get pods -o go-template='Hello, World!{% raw %}{{\n}}{% endraw %}'
error: error parsing template Hello, World!{% raw %}{{\n}}{% endraw %}, template: output:1: unexpected "\\" in command
```
#### Recap
* We passed a plain string to the Go template engine as a command line proof of concept.
* We used `{% raw %}{{"\n"}}{% endraw %}` to adjust the output formatting.
* We learned how the Go Template engine determines what it needs to interpret.

### Lesson 2: A Practical Example
In order to write a Go template, we need to know what our output looks like. We will use pods as an example:
```
$ oc get pods
NAME                     READY     STATUS      RESTARTS   AGE
httpd-example-1-build    0/1       Completed   0          12m
httpd-example-1-deploy   0/1       Completed   0          11m
httpd-example-1-rdnbw    1/1       Running     0          11m
```
Unfortunately, that does not tell us a whole lot about the structure that the API is giving back to us. If we add `-o yaml` we will get a better idea of the structure:
```
$ oc get pods -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      k8s.v1.cni.cncf.io/networks-status: ""
      openshift.io/build.name: httpd-example-1
      openshift.io/scc: privileged
    creationTimestamp: 2020-05-05T16:30:40Z
    labels:
      openshift.io/build.name: httpd-example-1
    name: httpd-example-1-build
    ...
```
Looking at the top level keys, we can see `apiVersion`, `items`, and a number of other keys that I have snipped out for display purposes. Building on what we learned in the previous example, let's return the apiVersion value for the List object that gets returned:
```
$ oc get pods -o go-template='{% raw %}{{.apiVersion}}{{"\n"}}{% endraw %}'
v1
```
That is a simple example of accessing a key in an object to retrieve the value. Since we know that this List object has our pods as well, let's try accessing the `items` key:
```
$ oc get pods -o go-template='{% raw %}{{.items}}{% endraw %}'
[text blob]
```
After running that command, you should have received a rather large blob of text which amounts to the Go representation of the items values. This, of course, is not very helpful. We need to tell the template to read through the list and display something for each item in the list. That is where `range` comes in; it takes an array, slice, map, or channel and loops through it. The format is: `{% raw %}{{range pipeline}} T1 {{end}}{% endraw %}`.

In the YAML snippet above, we can see that under items > metadata there is a `name` key which contains our pod name. The structure is traversed using dot notation, so to access the name of the item in the `range` structure, we use `.metadata.name`. Let's use `range` to print out the pod names:
```
$ oc get pods -o go-template='{% raw %}{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}{% endraw %}'
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw
```
Success!

#### Recap
* We learned how to interrogate the output from the API in order to write our Go template.
* We learned how to return an individual value by using a key from the structure that is returned.
* We learned how to use `range` to iterate over a list of objects.
* We learned how to access a nested key's value with `.metadata.name`.

### Lesson 3: Using a Go Template File
The `oc` utility has an additional option for using Go templates, which is `-o go-template-file=`. In this section we will move our Go template to a file, and build a more robust set of details for our pod list.

The first step is to get our Go template into a file. The filename does not matter, but my recommendation is to be indicative for easy identification. In this case, we will use `podlist.gotemplate` as the name and do a test run using the newly created file:
```
$ echo '{% raw %}{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}{% endraw %}' > podlist.gotemplate
$ cat podlist.gotemplate 
{% raw %}{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}{% endraw %}
$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw

$
```
We have successfully moved our Go template structure into a file and can get data back. However, we now have a strange blank line at the end of our output. This happens because Go templates pass through whitespace without filtering it; it is left up to the Go template writer to remove that whitespace. There are two ways we can fix the unnecessary whitespace:

1. Rerun our `echo` command with `-n` to skip the newline character: `echo -n '{% raw %}{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}{% endraw %}' > podlist.gotemplate`
2. Change the Go template to remove whitespace.

Option 1 is low friction, since you are just adding a flag to the echo command. Option 2 requires some explanation but will help you in all Go templates instead of just in this section, so let's go with Option 2 by using your favorite text editor to open `podlist.gotemplate`.

Your file should look like this:
```
{% raw %}{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}{% endraw %}
```

Let's make this a bit more readable by putting the metadata and newline part of the template on its own line:
```
{% raw %}{{range .items}}
    {{.metadata.name}}{{"\n"}}
{{end}}{% endraw %}
```
For the sake of example, if you save the file then rerun the `oc get` command, we wind up with _a lot_ of whitespace:
```
$ oc get pods -o go-template-file=podlist.gotemplate

    httpd-example-1-build


    httpd-example-1-deploy


    httpd-example-1-rdnbw

```
To fix this, we will need to jump back to the Go template documentation on [text and spaces](https://golang.org/pkg/text/template/#hdr-Text_and_spaces). To summarize, adding a hyphen inside `{% raw %}{{ }}{% endraw %}` will remove whitespace from the side that the hyphen is added to. So `{% raw %}{{- }}{% endraw %}` will remove space to the left, and `{% raw %}{{ -}}{% endraw %}` will remove space to the right, and `{% raw %}{{- -}}{% endraw %}` will remove space from both sides. Let's put this into practice with our `podlist.gotemplate` file to get the output we are expecting. In your favorite editor, modify the file to look like this:
```
{% raw %}{{- range .items -}}
    {{.metadata.name}}{{"\n"}}
{{- end -}}{% endraw %}
```
Then rerun the command for successful output:
```
$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw
```
#### Recap
* We converted our command line Go template structure to a file-based Go template structure.
* We learned how to influence the whitespace that the Go template engine gives back by using hyphens in our Go template.
* See `./lesson3.gotemplate` for the end result of what your template should look like.

### Lesson 4: Adding More Context
In Lesson 3, we laid the foundation for our pod list Go template, but we did not accomplish anything substantial in terms of output or giving ourselves more context. This lesson will focus on expanding the context of our `podlist.gotemplate` to provide a more comprehensive picture of the pods that get returned.

To determine our success criteria, let's determine some questions we want to answer by using the Go template.
1. What nodes are the pods running on?
2. What life cycle phase is the pod in?
3. Are the pods using any volumes?

Number 1 is pretty easy since we are just accessing an individual key. Let's add `{% raw %}{{.spec.nodeName}}{{"\n"}}{% endraw %}` to `podlist.gotemplate` and try it out:
```
$ cat podlist.gotemplate 
{% raw %}{{- range .items -}}
    {{.metadata.name}}{{"\n"}}
    {{.spec.nodeName}}{{"\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build

    worker-0.mycluster.com
httpd-example-1-deploy

    worker-0.mycluster.com
httpd-example-1-rdnbw

    worker-1.mycluster.com

```
This is pretty close, but the formatting is off, and at a glance, we do not have any labels that describe what we are looking at. To remove the extra newline, we will drop the `{% raw %}{{"\n"}}{% endraw %}` after the pod name, and then we can add some additional text so we know what the output means:
```
$ cat podlist.gotemplate 
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}{{"\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
```
Much better. Now we can easily tell what the pod's name is, and what node it is running on. One thing you will notice about this output is that NODE is indented, but if you look at `podlist.gotemplate`, they appear to be on the same level. This goes back to how the Go template engine processes whitespace; the hyphen on the right side of our opening range block removes the whitespace from the template up until the word POD, then it prints out the pod name, followed by four spaces, and then NODE. If you want to be more verbose about formatting, you can accomplish the same thing by making your template look like the following:
```
$ cat podlist.gotemplate 
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}{{"\n" -}}
    {{"    "}}NODE: {{.spec.nodeName}}{{"\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
```
For the rest of this lesson, we will use the simplified version, but when building Go templates, use your best judgement for where you need to use the formatting tweaks.

Next, we need to get the pod's life cycle phase, which is similar to getting the node that the pod is running on. All we need to add is a call to `.status.phase` in our template:
```
$ cat podlist.gotemplate 
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}{{"\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
```
The last piece we need to accomplish is getting a list of volumes from the pod:
```
$ cat podlist.gotemplate 
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{range .spec.volumes -}}
                {{.name}}{{" "}}
             {{- end -}}
             {{"\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: buildcachedir buildworkdir [...snipped for brevity]
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: deployer-token-7jw9f 
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
    VOLUMES: default-token-ths25
```
#### Recap
* We expanded `podlist.gotemplate` to provide more context and to answer these questions:
  * What nodes are the pods running on?
  * What life cycle phase is the pod in?
  * Are the pods using any volumes?
* See `./lesson4.gotemplate` for the end result of what your template should look like.

### Lesson 5: Go Template Nesting and Modularity
In this lesson, we will focus on making our template a bit more modular so we can avoid making changes in the main template. Since `podlist.gotemplate` is rather small, this is not a big inconvenience, but when you get into more complex Go templates, the nesting functionality will save you a lot of time!

If you have been following along, your `podlist.gotemplate` should look like this:
```
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{range .spec.volumes -}}
                {{.name}}{{" "}}
             {{- end -}}
             {{"\n"}}
{{- end -}}{% endraw %}
```
What we are going to do is move the logic for the volumes to its own template so we can make changes to the volume display easier. The first thing we need to do is define our template at the top of `podlist.gotemplate` and copy in our `range` block that is next to `VOLUMES`:
```
{% raw %}{{- define "volumes" -}}
    {{range .spec.volumes -}}
        {{.name}}{{" "}}
    {{- end -}}
{{- end -}}{% endraw %}
```
This template definition allows us to call it directly in our main template; let's make that change next by removing the `range` block next to `VOLUMES` and calling the new template we just defined. The end result will look like this:
```
{% raw %}{{- define "volumes" -}}
    {{range .spec.volumes -}}
        {{.name}}{{" "}}
    {{- end -}}
{{- end -}}

{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{template "volumes" .}}{{"\n"}}
{{- end -}}{% endraw %}
```
You will notice that all we passed to our newly defined template is "." which amounts to the current object being iterated on. So `{% raw %}{{template "volumes" .items[0]}}{% endraw %}` then `{% raw %}{{template "volumes" .items[1]}}{% endraw %}` and so on. Now let's run it to prove that it works:
```
$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: buildcachedir buildworkdir [...snipped]
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: deployer-token-7jw9f 
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
    VOLUMES: default-token-ths25
```
That looks good!

Now, you might be saying to yourself, "Why did we do that work to get the same output we had before?!" The answer is "modularity." Taking our build pod as an example, there are 10-plus volumes that quickly become unreadable when it is all on one line. If we change the main template, there is a risk that we will impact the formatting of other pieces if we are not careful. The defined template lets us operate only on the volume list without impacting the main template.

Let's give this modularity a shot by adjusting the `volumes` template to display the list vertically instead of horizontally and then run the full template to see the output:
```
$ head -n 5 podlist.gotemplate 
{% raw %}{{- define "volumes" -}}
    {{range .spec.volumes}}
        {{.name -}}
    {{end}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        buildcachedir
        buildworkdir
        builder-dockercfg-dpmt2-push
        builder-dockercfg-dpmt2-pull
        build-system-configs
        build-ca-bundles
        build-proxy-ca-bundles
        container-storage-root
        build-blob-cache
        builder-token-46g6q
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        deployer-token-7jw9f
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
    VOLUMES: 
        default-token-ths25
```
That definitely makes the volume list much easier to read! Now let's use this premise of modularity to add more to our output. It might be helpful to have the list of labels on the pods also, so let's define a Go template for labels. Since labels are key:value pairs, we will need to leverage variable assignment to get both the key and the value. At the top of `podlist.gotemplate` add the following `labels` definition:
```
{% raw %}{{- define "labels" -}}
    {{range $label,$value := .metadata.labels}}
        {{$label}}{{" => "}}{{$value -}}
    {{end}}
{{- end -}}{% endraw %}
```
Let's break down what this is doing:
1. Define a template named `labels`.
2. Iterate over the passed in pipeline using `range`. The label *key* gets assigned to the `$label` variable and the label *value* gets assigned to the `$value` variable.
3. Print out the value of `$label` and `$value` separated by `=>`. We are using a hyphen on the right side of the `$value` to remove any extra whitespace after the label value gets printed.

Now that we understand what this template is doing, we need to incorporate it into our main template to print out the labels. Add a `LABELS` section to the main template so it looks like the following and then run the template:
```
$ tail -n 8 podlist.gotemplate 
{% raw %}{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{template "volumes" .}}
    LABELS: {{template "labels" .}}
    {{- "\n"}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        buildcachedir
        buildworkdir
        builder-dockercfg-dpmt2-push
        builder-dockercfg-dpmt2-pull
        build-system-configs
        build-ca-bundles
        build-proxy-ca-bundles
        container-storage-root
        build-blob-cache
        builder-token-46g6q
    LABELS: 
        openshift.io/build.name => httpd-example-1
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        deployer-token-7jw9f
    LABELS: 
        openshift.io/deployer-pod-for.name => httpd-example-1
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
    VOLUMES: 
        default-token-ths25
    LABELS: 
        deployment => httpd-example-1
        deploymentconfig => httpd-example
        name => httpd-example
```
#### Recap
* We learned how to define a template and nest it in another template.
* We demonstrated how modularity in Go templates can be used.
* We learned some basic variable assignments within `range` to get key:value pairs.
* See `./lesson5.gotemplate` for the end result of what your template should look like.

### Lesson 6: Conditionals
In this lesson we will learn about conditionals and how to use them to influence our output.

Through the lessons so far, we have been using the same `oc get pods` command as the vehicle to test out `podlist.template`. But what happens if we accidentally do `oc get services` with `podlist.gotemplate`? Will the output be misleading or incorrect? Let's test:
```
$ oc get services -o go-template-file=podlist.gotemplate
POD: httpd-example
    NODE: <no value>
    PHASE: <no value>
    VOLUMES: 
    LABELS: 
        app => httpd-example
        template => httpd-example
```
We can quickly see that this is partially correct and partially misleading. `httpd-example` may look similar to a pod name, but it is actually the name of the service. However, the labels are correct; modularity for the win! Now we need to make sure that our template only runs for pods. What we will add is an `if` statement and check to ensure that the value of `Kind` is equal to the string `Pod`. If that evaluates to `true` then the template will run for that particular object. Adjust your main template to look like the following:
```
$ tail -n 10 podlist.gotemplate 
{% raw %}{{- range .items -}}
    {{- if eq .kind "Pod" -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{template "volumes" .}}
    LABELS: {{template "labels" .}}
    {{- "\n"}}
    {{- end -}}
{{- end -}}{% endraw %}
```
If you check in the [Go Template Actions](https://golang.org/pkg/text/template/#hdr-Actions) documentation, you will see there is an option to do `{% raw %}{{if pipeline}}{% endraw %}` where pipeline can be an actual pipeline (like our list of objects) or a sequence of commands/functions. In this case, we are using the `eq` (equals) function to check if the value of `.kind` matches the string `Pod`. If this returns true, it will print out all of the data we expect to see for a pod like in the previous lessons. More on `eq` and other comparison functions can be found in the [Go Template Functions](https://golang.org/pkg/text/template/#hdr-Functions) documentation.

To test out our template, let's do `oc get all` so we are checking against more than just the `Service` use case. We will also run a sanity check using `oc get services` to ensure nothing gets returned:
```
$ oc get all -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        buildcachedir
        buildworkdir
        builder-dockercfg-dpmt2-push
        builder-dockercfg-dpmt2-pull
        build-system-configs
        build-ca-bundles
        build-proxy-ca-bundles
        container-storage-root
        build-blob-cache
        builder-token-46g6q
    LABELS: 
        openshift.io/build.name => httpd-example-1
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        deployer-token-7jw9f
    LABELS: 
        openshift.io/deployer-pod-for.name => httpd-example-1
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
    PHASE: Running
    VOLUMES: 
        default-token-ths25
    LABELS: 
        deployment => httpd-example-1
        deploymentconfig => httpd-example
        name => httpd-example

$ oc get services -o go-template-file=podlist.gotemplate
$
```
Fantastic! We can see only our pods get returned and when we run `podlist.gotemplate` against a non-pod resource, we do not get any output. There is one more area that we can use conditionals on in `podlist.gotemplate`, specifically the `labels` template. In general, a pod will always have a name, phase, node, and volume(s), but it may not have labels. If I remove the `openshift.io/build.name` from `pod/httpd-example-1-build`, the output is blank, which does not tell us if there is an issue processing the labels or if there are actually no labels:
```
$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        buildcachedir
        buildworkdir
        builder-dockercfg-dpmt2-push
        builder-dockercfg-dpmt2-pull
        build-system-configs
        build-ca-bundles
        build-proxy-ca-bundles
        container-storage-root
        build-blob-cache
        builder-token-46g6q
    LABELS:
```
We can use conditionals to make this more obvious when there are no labels. Let's adjust our `labels` template to include a conditional and rerun our test:
```
$ head -n 9 podlist.gotemplate 
{% raw %}{{- define "labels" -}}
    {{if .metadata.labels}}
    {{- range $label,$value := .metadata.labels}}
        {{$label}}{{" => "}}{{$value -}}
    {{end}}
    {{else}}
        No labels
    {{end}}
{{- end -}}{% endraw %}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
    PHASE: Succeeded
    VOLUMES: 
        buildcachedir
        buildworkdir
        builder-dockercfg-dpmt2-push
        builder-dockercfg-dpmt2-pull
        build-system-configs
        build-ca-bundles
        build-proxy-ca-bundles
        container-storage-root
        build-blob-cache
        builder-token-46g6q
    LABELS: 
        No labels
```
Now we can see definitively when there are no labels instead of assuming that blank space means labels!

#### Recap
* We learned about conditionals and how important they are for Go template accuracy.
* We learned how to filter out objects from our output using conditionals.
* We used conditionals to make our output clearer when there are no labels on a pod.
* See `./lesson6.gotemplate` for the end result of what your template should look like.

### Lesson 7: Functions
Go templates have a set of [predefined functions](https://golang.org/pkg/text/template/#hdr-Functions) that can be used to improve your Go template design. In this lesson, we will put some of those functions to use in our `podlist.gotemplate` file. If you have been following along, then you should be caught up, but you can also make a copy of `lesson6.gotemplate` to get back up to speed: `cp lesson6.gotemplate podlist.gotemplate`.

The first function we are going to use is the `len` function, which returns the count of whatever argument is passed to it. In this case, it might be helpful for us to know how many containers a pod is running, so we will use `len` to put this information into our template. Since this is a simple function call, we do not need to define another template; simply add `CONTAINER COUNT: `{% raw %}{{len .spec.containers}}{% endraw %}` into your main Go template block just under `POD: {% raw %}{{.metadata.name}}{% endraw %}` so your main block should look like this:
```
POD: {% raw %}{{.metadata.name}}{% endraw %}
CONTAINER COUNT: {% raw %}{{len .spec.containers}}{% endraw %}
NODE: {% raw %}{{.spec.nodeName}}{% endraw %}
PHASE: {% raw %}{{.status.phase}}{% endraw %}
VOLUMES: {% raw %}{{template "volumes" .}}{% endraw %}
LABELS: {% raw %}{{template "labels" .}}{% endraw %}
```
Run the Go template to see your new container count metric:
```
$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-7xz5x
    CONTAINER COUNT: 1
    NODE: worker-0.mycluster.com
    PHASE: Running
    VOLUMES: 
        default-token-fdrhj
    LABELS: 
        deployment => httpd-example-1
        deploymentconfig => httpd-example
        name => httpd-example
```
We can also use functions to simplify our template design. Let's take this line from the `labels` template as an example:
```
{% raw %}{{$label}}{{" => "}}{{$value -}}{% endraw %}
```
There is a lot of Go template  specific notation in there that makes it harder to tell what is going on. We can use the `print` function to make this easier to read by changing the line like so:
```
{% raw %}{{print $label " => " $value -}}{% endraw %}
```
Now it is much more clear that we are going to print `$label` followed by `=>` and then `$value`.

#### Recap
* We learned about the set of predefined functions available in Go templates.
* We added a container count metric using the `len` function.
* We simplified our `labels` template by using the `print` function.
* See `./lesson7.gotemplate` for the end result of what your template should look like.
---
### Beyond the Workshop
At this point, the above workshop has covered most of what you would need to know in order to be proficient in using Go templates. The next part of this workshop will be covering specific use cases where Go templates can be used to facilitate with cluster operation and interrogation. The use cases will cover more advanced Go template writing like chaining functions, deeper logic, and operationalizing Go templates. The use case sections are not designed in any particular order and can be used atomically.

### Use Case: Identifying Tainted Nodes
Use this Go template to identify which nodes have taints configured:

**Go Template**: [tainted-nodes.gotemplate](tainted-nodes.gotemplate)
```
$ oc get nodes -o go-template-file=tainted-nodes.gotemplate 
NODE: worker-1.mycluster.com
TAINTS:
  Key: app
  Value: myapp
  Effect: NoSchedule
```
**Tips**
* The template is List- and single-object aware, so it will work with `oc get nodes` and `oc get nodes [node]`

**Test Conditions**
* Tested against list of nodes
* Tested against a single node
* Tainted node `worker-1` using `oc adm taint nodes worker-1.mycluster.com app=myapp:NoSchedule`

### Use Case: Error and Warning Event Dashboard
Use this Go template to get a shorthand list of warning/error events, their counts, and the namespace in which they occurred:

**Go Template**: [error-warning-event-dashboard.gotemplate](error-warning-event-dashboard.gotemplate)
```
$ oc get events -o go-template-file=error-warning-event-dashboard.gotemplate
COUNT    NAMESPACE       MESSAGE
13       openshift-sdn   Liveness probe failed: ovs-ofctl: /var/run/openvswitch/br0.mgmt: failed to open socket (Broken pipe)
```
**Tips**
* Add `--sort-by={% raw %}{.count}{% endraw %}` to apply event sorting by count.

**Test Conditions**

Project with no events:
```
$ oc get events -o go-template-file=error-warning-event-dashboard.gotemplate
There are no events.
```

Project with no Warning/Error events:
```
$ oc get events -o go-template-file=error-warning-event-dashboard.gotemplate
COUNT    NAMESPACE       MESSAGE

$ oc get events
LAST SEEN   TYPE     REASON             OBJECT                                  MESSAGE
12m         Normal   Killing            pod/httpd-example-1-7xz5x               Stopping container httpd-example
11m         Normal   Pulled             pod/httpd-example-1-hhzcj               Container image "image-registry.openshift-image-registry.svc:5000/workshop/httpd-example@sha256:afe489302ba6d0f29bd1da7c612be8c8d3c00ed29fb29f114a17a9c81a53d6e0" already present on machine
11m         Normal   Created            pod/httpd-example-1-hhzcj               Created container httpd-example
11m         Normal   Started            pod/httpd-example-1-hhzcj               Started container httpd-example
<unknown>   Normal   Scheduled          pod/httpd-example-1-hhzcj               Successfully assigned workshop/httpd-example-1-hhzcj to worker-0.mycluster.com
<unknown>   Normal   Scheduled          pod/httpd-example-1-nrr69               Successfully assigned workshop/httpd-example-1-nrr69 to worker-2.mycluster.com
...[snipped]
```
