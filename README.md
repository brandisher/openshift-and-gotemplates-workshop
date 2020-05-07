# OpenShift and Go Templates Workshop
A self-led workshop to demonstrate the power of go templates and how much fun you can have while using them with OpenShift!  Each section builds on the previous section but if you have some gotemplate knowledge already then you should be able to jump around without issue.

## Prerequisites
* Access to an OpenShift cluster.  Some steps may require cluster-admin permissions.
* A project for the workshop.  We'll be using a project named `workshop` with the `httpd-example` template deployed, but you can use whatever project you'd like that has running workloads.
* `oc` in your `$PATH`

## Start
### What is a gotemplate?
gotemplate is a format used by the Go programming language.  The documentation for gotemplates can be found [here](https://golang.org/pkg/text/template/).  Keep this handy as a reference as we progress through the workshop.  In OpenShift, gotemplates are very helpful for encapsulating output logic and customizing the view that you get back from the API.

### Step 1: A simple example
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

### Step 2: A practical example
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

To recap, in this section we've covered:
* Retrieving a value by referencing a key in an object.
* Using range to iterate through a list of objects.

### Step 3: Using a gotemplate file
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
* We learned how to influence the whitespace that the gotemplate engine gives back by using hypens in our template.