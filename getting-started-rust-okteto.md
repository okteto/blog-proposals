# Getting Started with Okteto and Rust Development in Kubernetes

Without any doubt, Kubernetes has become the default platform to run modern applications. However, for a developer, working with Kubernetes brings a new set of challenges, and a learning curvet that  might be intimidating. Additionally, and more importantly, the development inner loop now requires a few more steps before you can test your app.

To put this in perspective, the inner loop for Rustaceans looks something like this:

* Write some code
* Run the app with "cargo run" (which includes compiling)

And when you add Kubernetes to the formula, the inner loop looks like this:

* Write some code
* Build a container image (which includes compiling)
* Push the container image to the registry
* Deploy the app to Kubernetes

Those extra steps from above easily translates into a minimum of two minutes. If you want to fix something quickly, those extra steps become an eternity. There has to be a better way, right? Yes, there is, and the answer is Okteto.

## Okteto: A Rustacean's Friend

[Okteto](https://github.com/okteto/okteto) is a tool that helps developers to develop applications for Kubernetes without any extra steps in the inner loop. Rustaceans immediately forget  that they're not running applications locally but in a Kubernetes cluster.

Long are the days where you defer for later the stage of testing your applications in Kubernetes. Okteto helps you continue developing with your existing inner loop and testing the app in Kubernetes without any extra steps. Once you have Okteto installed locally, when you run `cargo run`, Okteto takes care of the rest.

But, I know you came here for the details, so let's see how you can get started with Okteto and Rust.

## Step 1: Deploy a Simple Rust Application to Kubernetes

You can use any existing Rust application, but if you don't have one, clone our sample app repository also to get the manifest to deploy the application in Kubernetes:

```
$ git clone https://github.com/okteto/rust-getting-started
$ cd rust-getting-started
```

Before you start running your application locally, let's confirm that you can deploy it to Kubernetes. The repository comes with a manifest you can use to deploy the app to Kubernetes. Any Kubernetes cluster works; it doesn't matter if you're using a local cluster or a remote one running a cloud provider. However, if you don't have a Kubernetes cluster, you can give [Okteto Cloud](https://okteto.com/blog/how-to-develop-go-apps-in-kubernetes/www.okteto.com) a try

Deploy your application to Kubernetes running the following command

```
$ kubectl apply -f k8s.yml
```

You can confirm that the application is running by getting the pod's status, which for now is enough.

## Step 2: Install the Okteto CLI Locally

To get started, you need to install the Okteto CLI in your local workstation, where you type your Rust code.

The Okteto CLI is an open-source project that helps you to continue using your local tooling. Once you finish writing code, you can run "cargo run" with Okteto, and the app container is updated in Kubernetes. Okteto is the tool that helps you to improve your developer productivity when developing apps that run in Kubernetes. So, it would help if you have the Okteto CLI locally.

If you're using Mac or Linux, run the following command to install the Okteto CLI:

```
$ curl https://get.okteto.com -sSfL | sh
```

If you're using Windows, you need to download the executable and[ include its location in your $PATH variable](https://www.architectryan.com/2018/03/17/add-to-the-path-on-windows-10/). Or, you can also[ use the Linux subsystem](https://docs.microsoft.com/en-us/windows/wsl/install-win10) and run the command from above.

To confirm that the Okteto CLI works on your machine, run the following command:

```
$ okteto version
```

Great, you're getting closer to experience getting back your traditional inner development cycle.


## Step 3: Create an Okteto Manifest

Before you can start using Okteto, you need to create a[ YAML manifest](https://okteto.com/docs/reference/manifest/index.html) required to activate your development environment.

You don't have to learn how to create an Okteto manifest yet. Instead, use the Okteto CLI, which will ask you to pick an existing deployment in your Kubernetes cluster (this is why you started by deploying the application). Based on the code you have, the CLI generates a manifest. So, run the following command:

```
$ okteto init
```

You should see an output like this one:

```
This command walks you through creating an okteto manifest.
It only covers the most common items, and tries to guess sensible defaults.
See https://okteto.com/docs/reference/manifest for the official documentation about the okteto manifest.
Use the arrow keys to navigate: ↓ ↑ → ←
Select the deployment you want to develop:
  ▸ hello-world
    Use default values
```

The `hello-world` deployment is the one you deployed previously for the sample Rust app.

And now you should have an okteto.yml file with the following content:

```
name: hello-world
image: okteto/rust:1
command: ["bash"]
workdir: /okteto
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

Without explaining every line of the above manifest, let me highlight and explain a few of them:

* `name`: the name of the Kubernetes deployment you want to put on "development mode."
* `image`: the[ image](https://okteto.com/docs/reference/development-environment) used by the development container
* `command`: the start command of the development container
* `forward`: a list of ports to forward from your development container.

You can get more details about the Okteto manifest at our[ official documentation site](https://okteto.com/docs/reference/manifest/index.html).

## Step 4: Activate Your Development Container

Now you have everything you need to spin up your development environment using Okteto.

Run the following command to enter in a development mode:

```
$ okteto up
```

You should see something like this:

```
✓  Development container activated
 ✓  Files synchronized
    Namespace: default
    Name:      hello-world
    Forward:   8080 -> 8080
               2345 -> 2345
Welcome to your development container. Happy coding!
default:hello-world app>
```

This becomes your new terminal to run your Rust application every time you change its code. All the commands you run in this terminal are running in Kubernetes. Okteto opens a terminal in a container that has all the tooling you need for developing in Rust. To give it a try, run the following command:

```
default:hello-world okteto> cargo run
```

You should see an output like the following where you can see how the app is compiling and launching a proxy:

```
  Compiling hello-world v0.1.0 (/okteto)
    Finished dev [unoptimized + debuginfo] target(s) in 4.72s
     Running `target/debug/hello-world`
Listening on http://0.0.0.0:3000
```

To confirm that the app works, launch a new terminal, and run the following command:

```
$ curl localhost:3000
Hello Kubernetes!
```

You might be thinking that you've just complicated things a little bit, but hold on a second and keep reading.

## Step 5: See Application Changes in Kubernetes

Don't close the Okteto terminal, and let's change something in the Rust sample app.

For instance, head over to the[ src/main.rs file, and in line #9](https://github.com/okteto/rust-getting-started/blob/master/src/main.rs#L9) change response text, like this:

```
async fn hello(_: Request) -> Result&lt;Response, Infallible> {
    Ok(Response::new(Body::from("Hello World from a Kubernetes Cluster!")))
}
```

Go back to the Okteto terminal, cancel the command (Ctrl + C) and rerun the cargo command (as you would typically do if you were testing your application locally):

```
default:hello-world okteto> cargo run
```

Go back to the other terminal where you're testing your application, and rerun the curl command:

```
$ curl localhost:3000
Hello World from a Kubernetes Cluster!
```

See? You didn't need any additional steps to see the changes reflected in Kubernetes.

## Conclusion
And that's it. This is how you can recover your developer productivity when you need to test your apps in Kubernetes. You don't need to extend your inner development loop because Okteto helps you to test remotely in the cluster.

Visit our [website](https://okteto.com/) to learn more about how to improve your team developer productivity with Okteto. Follow us[ on Twitter](https://twitter.com/oktetohq) and join our [#okteto](https://kubernetes.slack.com/messages/CM1QMQGS0/) channel in the [Kubernetes community Slack](http://slack.k8s.io/) to share your feedback with our community.

Happy koding!
