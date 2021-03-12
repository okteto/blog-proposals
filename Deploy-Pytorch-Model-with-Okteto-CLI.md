Imagine you are done building a model that generates unique love letters with no plagiarism and you want everyone to use the model and send unique, and personalized love letters to their partner or crush. This is where deployment comes in.

> The purpose of deploying your model is so that you can make the predictions from a trained ML model available to others.


![okteto](https://cdn-images-1.medium.com/max/800/1*nhMzSMXovJepnvn2DRknjQ.jpeg)

In this article, we will learn how to deploy a PyTorch GRU text generation model using Okteto.
> Effective deployment of machine learning models is more of an art than science
The PyTorch model I'm using in this article is a love letter generation model that generates unique love letters without any trace of plagiarism. This is very close to the real model that we currently use at Rosalove AI
## Table of content
- Text generation
- What's Okteto
- Okteto cloud
- Initial setup
- Deploy PyTorch model on Okteto cloud
- Resources
- Next: One-click Pytorch model deployment with Okteto cloud

## Text generation model with Pytorch
Text generation is a Natural Language Generation application that generates text. This application (also called a model) is trained with text and uses text to generate semantic and coherent text. It is also trained to learn the likelihood of occurrence of a word based on the previous sequence of words used in the text.
So we are going to clone a text generation model built with PyTorch and deploy it on okteto.
firstly, let's open your terminal and clone the repo



```python
$ git clone https://github.com/Emekaborisama/Text-Generation-with-pytorch.git
```

![okteto](https://cdn-images-1.medium.com/max/800/1*7aoy2ISJUpPCitN3-1lDFA.png)

We have trained the model and deploy it using flask API, the training script can be found on the "train" folder and the inference and API script on the "app" folder
When you clone the repo you'd notice the folder arrangement which I recommend for anyone deploying a machine learning model with flask and okteto.
After cloning the repo navigate to the text-generation folder and install the required packages
After installing the required packages, we have to test the API locally to see if everything is working fine.


```python
$ cd Text-Generation-with-pytorch
$ pip install -r requirements.txt
$ python 3 main.py
```

Basically, our app run successfully without an error and its time to explore the goodness of okteto

![okteto](https://cdn-images-1.medium.com/max/800/1*Zg4IoifUwMV0wr1fXbuwJg.png)

## What's okteto
Okteto is a developer platform used to accelerate the development workflow of cloud-native applications. We'll use Okteto to develop our PyTorch model and to deploy it as a flask API
## Okteto cloud
Okteto Cloud gives instant access to secure Kubernetes namespaces to enable developers to code, build, and run Kubernetes applications entirely in the cloud. With Okteto cloud, you can deploy your Machine Learning model with just a click of a button
## Okteto CLI
Okteto CLI tool that lets you develop your Cloud Native applications directly in Kubernetes. With the Okteto CLI, your code locally and Okteto will automatically sync your changes to your cluster.
To install Okteto CLI use the command below
```python
$ brew install okteto
```
Deployment with Okteto
Before using Okteto, we need to log into the service. You only need to do this once. Log in by using the command below.
```python 
$ okteto login
```
A browser tab will open automatically, confirming that you are now logged in. If this the first time that you use Okteto, it will ask you to log in with your Github identity

![okteto](https://cdn-images-1.medium.com/max/800/1*u_-5FJFRO_kG8dK1z8PeEg.png)

Now, let's connect with our namespace.
Run the ```"okteto namespace"``` command to activate your namespace and download the credentials to it. You can learn more about this command here.
```python 
$ okteto namespace [namespace name] e.g Okteto namespace emekaborisama
```

![okteto](https://cdn-images-1.medium.com/max/800/1*_EM0I4HBvQ4iLqnHXRNocw.png)


After activating our namespace, we will create a manifest file "okteto-stack.yaml". This file defines our application using a docker-compose like format.

```python
$ touch okteto.stack.yaml
```

Stacks are made for developers who don't want to deal with the complexities of Kubernetes manifests. Read more here
Paste the script inside the okteto-stack.yaml file.

```python
name: textgen
services:
 textgeneration:
 public: true
 image: okteto.dev/text_generation_with_pytorch
 build: .
 replicas: 1
 ports:
 - 8080
 resources:
 cpu: 2000m
 memory: 4280Mi
 ```
It's time to press the RED button to push "project deploy" to production.
Deploy your application using the command below

```python
$ okteto stack deploy - build
```

This command containerize your application, builds the instructions on the okteto-stack.yaml file and then deploys the application

![Okteto](https://cdn-images-1.medium.com/max/1000/1*A96mHOPSFs-bMMXxEkNbyA.png)

Now we have deployed our PyTorch generation model, the deployment command starts an application on your dashboard.
Hurray!!!!!!!!!!!!!!!

![okteto](https://cdn-images-1.medium.com/max/800/1*zGRPbBwBoSkIU4bzLfnafw.png)

Now, let's test our API with Postman. Copy the URL and use the "POST" method with /lovetextgen endpoint on Postman

![okteto](https://cdn-images-1.medium.com/max/800/1*PPQSEDmt-LSAnGz-vIkVcQ.png)

## Conclusion
So far, we have been able to build a text generation PyTorch model and deploy it with Flask and Okteto CLI and I love how Okteto makes it super easier to deploy my Machine Learning API on Kubernetes.
In our next article, we will learn how to deploy the model using Okteto cloud and Github.

