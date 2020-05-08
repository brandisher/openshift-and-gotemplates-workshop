# OpenShift and Go Templates Workshop
A self-led workshop to demonstrate the power of go templates and how much fun you can have while using them with OpenShift!  Each section builds on the previous section but if you have some gotemplate knowledge already then you should be able to jump around without issue.

### Table of Contents
* [Prerequisites](#prerequisites)
* [What is a gotemplate?](#what-is-a-gotemplate)
* [Lesson 1: A simple example](#lesson-1-a-simple-example)
* [Lesson 2: A practical example](#lesson-2-a-practical-example)
* [Lesson 3: Using a gotemplate file](#lesson-3-using-a-gotemplate-file)
* [Lesson 4: Adding more context](#lesson-4-adding-more-context)
* [Lesson 5: Gotemplate nesting and modularity](#lesson-5-gotemplate-nesting-and-modularity)

### Prerequisites
* Access to an OpenShift cluster.  Some lessons may require cluster-admin permissions.
* A project for the workshop.  We'll be using a project named `workshop` with the `httpd-example` template deployed, but you can use whatever project you'd like that has running workloads.
* `oc` in your `$PATH`

### What is a gotemplate?
gotemplate is a format used by the Go programming language.  The documentation for gotemplates can be found [here](https://golang.org/pkg/text/template/).  Keep this handy as a reference as we progress through the workshop.  In OpenShift, gotemplates are very helpful for encapsulating output logic and customizing the view that you get back from the API.

### Lesson 1: A simple example
The simplest example of a gotemplate is essentially a non-operation where we just print out a string that we pass to the template engine.
```
[user@machine]$ oc get pods -o go-template='Hello, World!'
Hello, World![user@machine]$
```
Here's where we can take our first step into leveraging the formatting features of gotemplates.  You'll notice that there's no newline after `Hello, World!` in the previous example; let's add a newline to put our prompt on a fresh line.
```
[user@machine]$ oc get pods -o go-template='Hello, World!{{"\n"}}'
Hello, World!
[user@machine]$
```
Much better!  This is a good opportunity to explain what we did and what's happening under the covers.  First, the double open and close braces `{{ }}` tell the gotemplate engine some processing needs to happen.  The `"\n"` tells the gotemplate engine to print a newline character at the end of `Hello, World!`.  Since everything not inside `{{ }}` is treated as a string literal, if we left off the double quotes from the newline, or simply put `\n` after `Hello, World!`, we'd get an error or output that doesn't achieve our goal.
```
# No braces
[user@machine]$ oc get pods -o go-template='Hello, World!"\n"'
Hello, World!"\n"[user@machine]$

# No double quotes
[user@machine]$ oc get pods -o go-template='Hello, World!{{\n}}'
error: error parsing template Hello, World!{{\n}}, template: output:1: unexpected "\\" in command
```

#### Recap
* We passed a plain string to the gotemplate engine as a command line proof of concept.
* We used `{{"\n"}}` to adjust the output formatting.
* We learned how the gotemplate engine determines what it needs to interpret.

### Lesson 2: A practical example
In order to write a gotemplate, we need to know what our output looks like.  We'll use pods as an example.

```
$ oc get pods
NAME                     READY     STATUS      RESTARTS   AGE
httpd-example-1-build    0/1       Completed   0          12m
httpd-example-1-deploy   0/1       Completed   0          11m
httpd-example-1-rdnbw    1/1       Running     0          11m
```

Unfortunately, that doesn't tell us a whole lot about the structure that the API is giving back to us.  If we add `-o yaml` we'll get a better idea of the structure.

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
Looking at the top level keys, we can see `apiVersion`, `items`, and a number of other keys that I've snipped out for display purposes.  Building on what we learned in the previous example, let's return the apiVersion value for the List object that gets returned.

```
$ oc get pods -o go-template='{{.apiVersion}}{{"\n"}}'
v1
```
That's a simple example of accessing a key in an object to retreive the value.  Since we know that this List object has our pods as well, let's try accessing the `items` key.
```
$ oc get pods -o go-template='{{.items}}'
[text blob]
```
After running that command, you should've received a rather large blob of text which amounts to the go representation of the `items` values.  This of course isn't very helpful, we need to tell the template to read through the list and display something for each item in the list.  That's where `range` comes in; it takes an array, slice, map, or channel and loops through it.  The format is: `{{range pipeline}} T1 {{end}}`.

In the YAML snippet above, we can see that under items > metadata there's a `name` key which contains our pod name.  The structure is traversed using dot notation so to access the name of the item in the `range` structure, we use `.metadata.name`.  Let's use `range` to print out the pod names.
```
$ oc get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw
```
Success!

#### Recap
* We learned how to interrogate the output from the API in order to write our gotemplate.
* We learned how to return an individual value by using a key from the structure that's returned.
* We learned how to use `range` to iterate over a list of objects.
* We learned how to access a nested key's value with `.metadata.name`.

### Lesson 3: Using a gotemplate file
The `oc` utility has an additional option for using gotemplates which is `-o go-template-file=`.  In this section we'll move our gotemplate to a file, and build a more robust set of the details for our pod list.

The first step is to get our gotemplate into a file.  The file name doesn't matter but my recommendation is to be indicative for easy identification.  In this case, we'll use `podlist.gotemplate` as the name and do a test run using the newly created file.
```
$ echo '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' > podlist.gotemplate
$ cat podlist.gotemplate 
{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}
$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw

$
```

We've successfully moved our gotemplate structure into a file and can get data back.  However, we now have a strange blank line at the end of our output.  This happens because gotemplates pass through whitespace without filtering it; its left up to the gotemplate writer to remove that whitespace.  There are two ways we can fix the unnecessary whitespace.

1. Rerun our `echo` command with `-n` to skip the newline character: `echo -n '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' > podlist.gotemplate`
2. Change the gotemplate to remove whitespace.

Option 1 is low friction since you're just adding a flag to the echo command.  Option 2 requires some explanation but will help you in all gotemplates instead of just in this section so let's go with Option 2 by using your favorite text editor to open `podlist.gotemplate`.

Your file should look like this.
```
{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}

```

Let's make this a bit more readable by putting the metadata and newline part of the template on its own line.
```
{{range .items}}
    {{.metadata.name}}{{"\n"}}
{{end}}
```
For the sake of example, if you save the file then rerun the `oc get` command we wind up with _a lot_ of whitespace.
```
$ oc get pods -o go-template-file=podlist.gotemplate

    httpd-example-1-build


    httpd-example-1-deploy


    httpd-example-1-rdnbw

```
To fix this, we'll need to jump back to the gotemplate documentation on [text and spaces](https://golang.org/pkg/text/template/#hdr-Text_and_spaces).  To summarize, adding a hyphen inside `{{ }}` will remove whitespace from the size that the hypen is added to.  So `{{- }}` will remove space to the left, `{{ -}}` will remove space to the right, and `{{- -}}` will remove space from both sides.  Let's put this into practice with out `podlist.gotemplate` file to get the output we're expecting.  In your favorite editor, modify the file to look like this:
```
{{- range .items -}}
    {{.metadata.name}}{{"\n"}}
{{- end -}}
```
Then rerun the command for successful output!
```
$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build
httpd-example-1-deploy
httpd-example-1-rdnbw
```

#### Recap
* We converted our commandline gotemplate structure to a file based gotemplate structure.
* We learned how to influence the whitespace that the gotemplate engine gives back by using hypens in our gotemplate.
* See `./lesson3.gotemplate` for the end result of what your template should look like.

### Lesson 4: Adding more context
In Lesson 3, we laid the foundation for our podlist gotemplate but we didn't accomplish anything substantial in terms of output or giving ourselves more context.  This lesson will focus on expanding the context of our `podlist.gotemplate` to provide a more comprehensive picture of the pods that get returned.

To determine our success criteria, let's determine some questions we want to answer by using the gotemplate.
1. What nodes are the pods running on?
2. What lifecycle phase is the pod in?
3. Are the pods using any volumes?

Number 1 is pretty easy since we're just accessing an individual key.  Let's add `{{.spec.nodeName}}{{"\n"}}` to `podlist.gotemplate` and try it out.
```
$ cat podlist.gotemplate 
{{- range .items -}}
    {{.metadata.name}}{{"\n"}}
    {{.spec.nodeName}}{{"\n"}}
{{- end -}}

$ oc get pods -o go-template-file=podlist.gotemplate
httpd-example-1-build

    worker-0.mycluster.com
httpd-example-1-deploy

    worker-0.mycluster.com
httpd-example-1-rdnbw

    worker-1.mycluster.com
```
This is pretty close but the formatting is off and at a glance, we don't have any labels which describe what we're looking at.  To remove the extra newline, we'll drop the `{{"\n"}}` after the pod name and then we can add some additional text so we know what the output means.
```
$ cat podlist.gotemplate 
{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}{{"\n"}}
{{- end -}}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
```
Much better.  Now we can easily tell what the pod's name is, and what node its running on.  One thing you'll notice about this output is that NODE is indented but if you look at `podlist.gotemplate`, they appear to be on the same level.  This goes back to how the gotemplate engine processes whitespace; the hypen on the right side of our opening `range` block removes the whitspace from the template up until the word `POD`, then it prints out the pod name, followed by 4 spaces, and then `NODE`.  If you want to be more verbose about formatting, you can accomplish the same thing by making your template look like the following:
```
$ cat podlist.gotemplate 
{{- range .items -}}
    POD: {{.metadata.name}}{{"\n" -}}
    {{"    "}}NODE: {{.spec.nodeName}}{{"\n"}}
{{- end -}}

$ oc get pods -o go-template-file=podlist.gotemplate
POD: httpd-example-1-build
    NODE: worker-0.mycluster.com
POD: httpd-example-1-deploy
    NODE: worker-0.mycluster.com
POD: httpd-example-1-rdnbw
    NODE: worker-1.mycluster.com
```
For the rest of this lesson, we'll use the simplified version but when building gotemplates, use your best judgement for where you need to use the formatting tweaks.

Next, we need to get the pod's lifecycle phase which is similar to getting the node that the pod is running on.  All we need to add is a call to `.status.phase` in our template.
```
$ cat podlist.gotemplate 
{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}{{"\n"}}
{{- end -}}

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
The last piece we need to accomplish is getting a list of volumes from the pod.
```
$ cat podlist.gotemplate 
{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{range .spec.volumes -}}
                {{.name}}{{" "}}
             {{- end -}}
             {{"\n"}}
{{- end -}}

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
  1. What nodes are the pods running on?
  2. What lifecycle phase is the pod in?
  3. Are the pods using any volumes?
* See `./lesson4.gotemplate` for the end result of what your template should look like.

### Lesson 5: Gotemplate nesting and modularity
In this lesson, we'll focus on making our template a bit more modular so we can avoid making changes in the main template.  Since `podlist.gotemplate` is rather small, this isn't a big inconvenience but when you get into more complex gotemplates the nesting functionality will save you a lot of time!

If you've been following along, your `podlist.gotemplate` should look like this:
```
{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{range .spec.volumes -}}
                {{.name}}{{" "}}
             {{- end -}}
             {{"\n"}}
{{- end -}}
```
What we're going to do is move the logic for the volumes to it's own template so we can make changes to the volume display easier.  The first thing we need to do is define our template at the top of `podlist.gotemplate` and copy in our `range` block that's next to `VOLUMES`
```
{{- define "volumes" -}}
    {{range .spec.volumes -}}
        {{.name}}{{" "}}
    {{- end -}}
{{- end -}}
```
This template definition allows us to call it directly in our main template; let's make that change next by removing the `range` block next to `VOLUMES` and calling the new template we just defined.  The end result will look like this:
```
{{- define "volumes" -}}
    {{range .spec.volumes -}}
        {{.name}}{{" "}}
    {{- end -}}
{{- end -}}

{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{template "volumes" .}}{{"\n"}}
{{- end -}}
```
You'll notice that all we passed to our newly defined template is "." which amounts to the current object being iterated on.  So `{{template "volumes" .items[0]}}` then `{{template "volumes" .items[1]}}` and so on.  Now let's run it to prove that it works.
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

Now, you might be saying to yourself "Why did we do that work to get the same output we had before?!"  The answer is "modularity".  Taking our build pod as an example, there are a 10+ volumes which quickly becomes unreadable when its all on one line.  If we change the main template there's a risk that we'll impact the formatting of other pieces if we're not careful.  The defined template lets us operate *only* on the volume list without impacting the main template.

Let's give this modularity a shot by adjusting the `volumes` template to display the list vertically instead of horizontally and then run the full template to see the output.
```
$ head -n 5 podlist.gotemplate 
{{- define "volumes" -}}
    {{range .spec.volumes}}
        {{.name -}}
    {{end}}
{{- end -}}

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
That definitely makes the volume list much easier to read!  Now let's use this premise of modularity to add more to our output.  It might be helpful to have the list of labels on the pods also, so let's define a gotemplate for labels.  Since labels are key:value pairs, we'll need to leverage variable assignment to get both the key and the value.  At the top of `podlist.gotemplate` add the following `labels` definition.
```
{{- define "labels" -}}
    {{range $label,$value := .metadata.labels}}
        {{$label}}{{" => "}}{{$value -}}
    {{end}}
{{- end -}}
```
Let's break down what this is doing.
1. Define a template named `labels`.
2. Iterate over the passed in pipeline using `range`.  The label *key* gets assigned to the `$label` variable and the label *value* gets assigned to the `$value` variable.
3. Print out the value of `$label` and `$value` separated by `=>`.  We're using a hyphen on the right side of the `$value` to remove any extra whitespace after the label value gets printed.

Now that we understand what this template is doing, we need to incorporate it into our main template to print out the labels.  Add a `LABELS` section to the main template so it looks like the following and then run the template:
```
$ tail -n 8 podlist.gotemplate 
{{- range .items -}}
    POD: {{.metadata.name}}
    NODE: {{.spec.nodeName}}
    PHASE: {{.status.phase}}
    VOLUMES: {{template "volumes" .}}
    LABELS: {{template "labels" .}}
    {{- "\n"}}
{{- end -}}

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
* We demonstrated how modularity in gotemplates can be used.
* We learned some basic variable assignment within `range` to get key:value pairs.
* See `./lesson5.gotemplate` for the end result of what your template should look like.
