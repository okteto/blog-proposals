As a developer, there's going to be a time where you need to work with different Kubernetes clusters. At a minimum, you need to work with two clusters. One is your local clusters, and a second one could be a remote cluster for testing purposes in a cloud provider or elsewhere. Kubernetes has a set of configurations you can use to interact with different clusters from your workstation using kubectl. At the end of the day, remember that you're interacting with an API to persist the desired state for your workloads.

Even though kubectl offers a set of commands you could use to work with different clusters, you might need another set of tools to make your life easier. And if you're using Okteto, you don't need to use another tool and almost forget about kubectl to work with different clusters. We'll see in this post how's that possible.

But first, let's briefly talk about contexts in Kubernetes, which is the configuration you use to work with multiple clusters.

## What's a Kubernetes Context?

A context in Kubernetes a set of parameters you have to communicate with a cluster from your local workstation using kubectl.

Each context contains a cluster, namespace, and user information that kubectl uses to make the cluster's API calls. You can find all these configurations in the `~/.kube/config` file. However, configuring this file could be overwhelming if you're starting with Kubernetes, especially if you want to start coding your applications. Most of the time, you don't have to configure the kubeconfig file as most of the Kubernetes installation providers already offer a command to generate that file.

If you're already using Kubernetes, you might give it a look in your `~/.kube/config` file, and you might find out that you already have the configuration necessary to connect to different clusters. In case you want to understand more about Kubernetes contexts, you can [visit the official Kubernetes documentation site](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) and learn how to [configure access to multiple clusters using contexts](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/). In this post, I'll focus on the different ways you have to use contexts to increase your development productivity when working with more than one cluster.

## Using Kubernetes Contexts

Once you have your kubeconfig file configured with the contexts you'll need, it's time to use them.

The idea behind configuring a Kubernetes context is that once you have configured your access to the different clusters, you can run this command:

```
$ kubectl config use-context docker-desktop
Switched to context "docker-desktop."
$ kubectl config use-context remote-cluster
Switched to context "remote-cluster".
```

However, the above commands are still a little bit longer. Although you could create aliases in your workstation, I'd highly recommend you to use [kubectx](https://github.com/ahmetb/kubectx), a tool for switching and managing contexts. Besides changing contexts, you can easily go back to the previous contexts, show the current context, rename or delete a context, and more. So the above commands now will be:

```
$ kubectx docker-desktop
Switched to context "docker-desktop".
$ kubectx remote-cluster
Switched to context "remote-cluster".
$ kubectx -
Switched to context "docker-desktop".
```

Every subsequence command you run will be making calls to the corresponding Kubernetes cluster.

## Working with Multiple Namespaces

At this point, you're able to connect and work with multiple Kubernetes clusters from your workstation. But remember that a context is also referring to a namespace in Kubernetes. That namespace will be the default one when you don't specify a namespace in a kubectl command. For instance, your `docker-desktop` context has the `default` namespace configured. Then, every time you run `kubectl get pods`, it will return the list of pods from the `default` namespace.

However, there are going to be times where you also need to work with different namespaces. Perhaps you're in charge of two applications, and each application uses a namespace to deploy the Kubernetes objects it needs. Every time you run a kubectl command, you'd have to specify the name of the namespace you want to use, like this:

```
$ kubectl get pods -n service-a
$ kubectl get svc -n service-b
```

You might don't see a problem with the above commands. However, when you're coding, you might need to change namespaces a little bit quicker and avoid any confusion if you forgot to include the namespace's proper name. If you don't want to specify a namespace in every kubectl command you run, you'd have to modify the `~/.kube/config` file. However, there's a project called [kubens](https://github.com/ahmetb/kubectx) (from the same project creator as kubectx) that you could use to switch between different namespaces quickly.  kubens will update the Kubernetes context for you automatically. The above commands now can be like this:

```
$ kubens
default
service-a
service-b
$ kuebns service-a
Context "docker-desktop" modified.
Active namespace is "service-a".
$ kubectl get pods
$ kubens service-b
Context "docker-desktop" modified.
Active namespace is "service-b".
$ kubectl get svc
```

As you've noticed, kubectx and kubens help you work with different clusters and namespaces, which is a little less problematic.

But what if there's even a better way for developers? Let's see.

## Developing with Contexts in Okteto

Okteto has support for configuring the context and namespace your applications might need. Hence, you have consistency at coding time because all developers are going to use the same configurations. Also, you'll have fewer commands to run. Even if the existing commands from kubectcl or the kubectx utility, you still need to switch manually. Conversely, with Otketo, you can include this configuration in the YAML manifest.

For instance, following [our sample from Rust](https://okteto.com/blog/getting-started-with-okteto-and-rust/), you could modify the `okteto.yaml` to this:

```
name: hello-world
image: okteto/rust:1
command: ["bash"]
workdir: /okteto
context: docker-desktop
securityContext:
  capabilities:
    add:
    - SYS_PTRACE
forward:
  - 3000:3000
  - 2345:2345
persistentVolume:
  enabled: false
```

Notice how we're specifying which context to use with the line `context: docker-desktop`. And even if you're currently pointing to the remote cluster context, Okteto will use the Kubernetes cluster from the Docker Desktop app. Additionally, if you don't want to have hardcoded values, you could use an environment variable, like `context: $APP_CONTEXT`. Developers can have different names for a local context, and they only need to set up `$APP_CONTEXT` environment variable so that Okteto points to the proper cluster.

Let's give it a try, run the following command to force the usage of the remote cluster:

```
$ kubectx remote-cluster
Switched to context "remote-cluster".
```

Now that the `okteto.yml` has the context configuration, run the `okteto up` command and compile the application, like this:

```
$ okteto up
 ✓  Development container activated
 ✓  Connected to your development container
 ✓  Files synchronized
    Context:   docker-desktop
    Namespace: default
    Name:      hello-world
    Forward:   2345 -> 2345
               3000 -> 3000

Welcome to your development container. Happy coding!
default:hello-world okteto> cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.78s
     Running `target/debug/hello-world`
Listening on http://0.0.0.0:3000
```

Notice how the Okteto output tells you which context is using.

You should now be able to see the application running in the local cluster, like this:

```
$ curl localhost:3000
Hello Kubernetes!
```

When you close the Okteto terminal, notice that you're still using the remote cluster context. Locally, nothing changed because Okteto allowed knows which context to use to deploy your application where it should be running. At least, you're going to make sure you're developing in a local cluster and not in a production cluster.

## Configuring Namespaces in Okteto

Additionally, Okteto allows you to change the namespace you'd like to use and follow the same principle: giving fewer things to worry about to a developer. Like the way you configure the context property, you also have the `namespace` property. So, your `okteto.yml` manifest could look like this:

```
name: hello-world
image: okteto/rust:1
command: ["bash"]
workdir: /okteto
context: docker-desktop
namespace: hello-rust
securityContext:
  capabilities:
    add:
    - SYS_PTRACE
forward:
  - 3000:3000
  - 2345:2345
persistentVolume:
  enabled: false
```

To give it a try to this configuration, let's go back to using the local context and make sure to set up the `default` namespace, like this:

```
$ kubectx docker-desktop
Switched to context "docker-desktop".
$ kubens default
Context "docker-desktop" modified.
Active namespace is "default".
```

Also, create a new namespace called `hello-rust` where you'll deploy your application:

```
$ kubectl create namespace hello-rust
$ kubectl apply -f k8s.yaml -n hello-rust
```

When you run `okteto up`, and then `cargo run`, your application will be running in the `hello-rust` namespace, even if your workstation uses the `default` namespace. You could confirm that nothing is running the `default` namespace by running the `kubectl get pods` outside of the Okteto terminal.

```
$ okteto up
 ✓  Development container activated
 ✓  Connected to your development container
 ✓  Files synchronized
    Context:   docker-desktop
    Namespace: hello-rust
    Name:      hello-world
    Forward:   2345 -> 2345
               3000 -> 3000

Welcome to your development container. Happy coding!
hello-rust:hello-world okteto>
```

Notice from the output that Okteto tells you in which context and namespace it's running.

## Conclusion

Kubernetes already has the tooling you need to work with multiple clusters and namespaces. However, this could affect a developer's productivity and is still error-prone. You might be pointing to the wrong cluster or namespace, and you didn't know. When you use Okteto for developing your applications, you can make sure the applications will use the proper context and namespace even if your kubectl has different settings.
