In this tutorial we will see real world example by Running  On Kubernetes in most easy way. 

Okteto is one of namespace base sandboxing using syncthing  technology . so you don't need to think of managing & configuring cluster . thats lead to use in this use case to our next point 

 # How Okteto Will benfits for Al Base Application 
 1. having easy control using okteto application catlog 
 2. faster & productive release 
 3. okteto pipline is inbuild ci/cd solution 
 4. one click deployement from github 
 5. okteto provide resource quota for developer pro 8GB memory , CPU 4.0 & 20 GB storage that can make more east way to use pre-trained model in machine learning developement containers 
 6. kuberntes itself provide self healing feature by scaling applications
 7. those who coming from containerization world okteto support docker compose native format to run top of kubernetes 
 8. easy to run microserivce base application 
 9. okteto build images more faster which is powered by buildkit 
 10. keep kubernetes namespace sandbox as developer or testing or production envirment

 before jumping into the practical demo lets understand Pre-requisite to build application 

before jumping into the practical demo lets understand Pre-requisite to build application 

# What is Spago ?

A**Machine Learning**library written in pure Go designed to support relevant neural architectures in**Natural Language Processing**.

spaGO is self-contained, in that it uses its own lightweight_computational graph_framework for both training and inference, easy to understand from start to finish.

availabe on github :- https://github.com/nlpodyssey/spago for more infromation 
thanks to our friend @matheogrella for creating spago ML/NLP library 


#  Why spago ?
 
 - Educational point of view highly recommanded those who getting started with GO+ML ?
 - very few dependencies , developed from scratch 
 - Production use - pivot towards a compresive , ready to use cloud native NLP library 
 - e.g stateless service , TLS encryption , gRPC , Docker , Okteto Cloud 
 - Go doesn't have a rich ecosystem & spago fills that niche by providing most important achitectures in the field .
 - ready to use compatible with  pytorch model - flair & tranformers 
 
  
  
# Quick Start Setup 

```
   git clone -b okteto-stack-v0.4.1 https://github.com/nlpodyssey/spago/
   
```

# Lets start with dockerizing spago 

as we know spago is written in golang lets containerize it here is Dokerfile :

```

cat Dockerfile 

```

lets understand construction of multistage dockerfile 


1. at 1st stage its use base image as  `FROM golang:1.15.6-alpine3.12 as Builder '  
2. some of the go packages used by spago required gcc. openssl is used to generate a self-signed cert in order to test the docker images the packages ca-certificates is needed in order to run the servers using TLS .
```
RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
		ca-certificates \
		gcc \
		musl-dev \
		openssl \
        ;
```
3.  The spago user is created so that the servers can be run with limited privileges.

```
RUN adduser -S spago

```
4. Build statically linked Go binaries without CGO.

```
RUN mkdir /build
ADD . /build/
WORKDIR /build
RUN go mod download
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-extldflags=-static" -o docker-entrypoint docker-entrypoint.go

```
5. A self-signed certificate and private key is generated so that the servers can easily support TLS without requiring the user to generate their own certificates.

```
RUN mkdir /etc/ssl/certs/spago \
	&& openssl req \
		-x509 \
		-nodes \
		-newkey rsa:2048 \
		-keyout /etc/ssl/certs/spago/server.key \
		-out /etc/ssl/certs/spago/server.crt \
		-days 3650 \
		-subj "/C=IT/ST=Piedmont/L=Torino/O=NLP Odyssey/OU=spaGo/emailAddress=matteogrella@gmail.com/CN=*" \
	&& chmod +r /etc/ssl/certs/spago/server.key \
	;

```

6. he definition of the runtime container now follows.
```
FROM debian:buster-slim
```

7. Copy the user info from the Builder container.
```
COPY --from=Builder /etc/passwd /etc/passwd
USER spago
```

8. Copy the CA certs and the self-signed cert.
```
COPY --from=Builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=Builder /etc/ssl/certs/spago/server.crt /etc/ssl/certs/spago/server.crt
COPY --from=Builder /etc/ssl/certs/spago/server.key /etc/ssl/certs/spago/server.key
```
9. Copy the docker entrypoint from the Builder container.
```
COPY --from=Builder /build/docker-entrypoint /docker-entrypoint
```

10. Setup the environment and run the script docker-entrypoint.sh so that a help screen is printed to the user when no commands are given.
```
ENV GOOS linux
ENV GOARCH amd64
ENTRYPOINT ["/docker-entrypoint"]
CMD ["help"]
```

# How to build Dockerfiles with Okteto 

```
okteto build .
 i  Running your build in tcp://buildkit.cloud.okteto.net:1234...
[+] Building 109.8s (20/20) FINISHED                                                                                                                                 
 => [internal] load .dockerignore                                                                                                                               1.0s
 => => transferring context: 2B                                                                                                                                 1.0s
 => [internal] load build definition from buildkit-192814155                                                                                                    1.3s
 => => transferring dockerfile: 2.71kB                                                                                                                          1.3s
 => [internal] load metadata for docker.io/library/debian:buster-slim                                                                                           2.8s
 => [internal] load metadata for docker.io/library/golang:1.15.6-alpine3.12                                                                                     2.8s
 => [internal] load build context                                                                                                                              79.0s
 => => transferring context: 21.66MB                                                                                                                           79.0s
 => [builder 1/9] FROM docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                      18.7s
 => => resolve docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                               0.0s
 => => sha256:801bfaa63ef2094d770c809815b9e2b9c1194728e5e754ef7bc764030e140cea 2.80MB / 2.80MB                                                                  1.9s
 => => sha256:1db7f31c0ee6d2a9f0c724c904ce2025164938e289d0250dd31d6bfafc452237 154B / 154B                                                                      1.8s
 => => sha256:ee0a1ba97153114db0134946070dd8e9886006994efe6e8fc8bd700f6970095f 280.81kB / 280.81kB                                                              1.9s
 => => sha256:1463476d8605182a5da61e0831798a541859065e9dd1caeb98fd040b32f0d486 4.61kB / 4.61kB                                                                  0.0s
 => => sha256:ecebeec079cfa4bc37115663487510e4fbcb10bfbf86180cd713a275eddfbbb5 106.71MB / 106.71MB                                                              5.1s
 => => sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2 1.65kB / 1.65kB                                                                  0.0s
 => => sha256:380503d71db14d717c8f4174552ed1996956e8de61f7806e5d9bcaa4c6651101 1.36kB / 1.36kB                                                                  0.0s
 => => sha256:63b48972323ac14c5e2bdf776112bcb9fd364c214f62cc954a0cbac0defb6fc1 125B / 125B                                                                      1.8s
 => => unpacking docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                             9.3s
 => [stage-1 1/6] FROM docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                             8.3s
 => => resolve docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                                     0.0s
 => => sha256:589ac6f94be479ab633e3f57adb8d2e4dcbe9afbdb4b155e3ce74e0aae1e00d7 1.46kB / 1.46kB                                                                  0.0s
 => => sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9 1.85kB / 1.85kB                                                                  0.0s
 => => sha256:b1af07039fe341833982bae85a2724ac8600ec5c74c37277c7a6ef7cddfb2cd0 529B / 529B                                                                      0.0s
 => => sha256:a076a628af6f7dcabc536bee373c0d9b48d9f0516788e64080c4e841746e6ce6 27.11MB / 27.11MB                                                                2.1s
 => => unpacking docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                                   1.9s
 => [builder 2/9] RUN set -eux;  apk add --no-cache --virtual .build-deps   ca-certificates   gcc   musl-dev   openssl         ;                                3.3s
 => [builder 3/9] RUN adduser -S spago                                                                                                                          0.1s
 => [builder 4/9] RUN mkdir /build                                                                                                                              0.1s 
 => [builder 5/9] ADD . /build/                                                                                                                                 0.3s 
 => [builder 6/9] WORKDIR /build                                                                                                                                0.0s
 => [builder 7/9] RUN go mod download                                                                                                                           7.4s
 => [builder 8/9] RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-extldflags=-static" -o docker-entrypoint docker-entrypoint.go                  17.8s
 => [builder 9/9] RUN mkdir /etc/ssl/certs/spago  && openssl req   -x509   -nodes   -newkey rsa:2048   -keyout /etc/ssl/certs/spago/server.key   -out /etc/ssl  0.3s
 => [stage-1 2/6] COPY --from=Builder /etc/passwd /etc/passwd                                                                                                   0.0s
 => [stage-1 3/6] COPY --from=Builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt                                                     0.0s
 => [stage-1 4/6] COPY --from=Builder /etc/ssl/certs/spago/server.crt /etc/ssl/certs/spago/server.crt                                                           0.0s
 => [stage-1 5/6] COPY --from=Builder /etc/ssl/certs/spago/server.key /etc/ssl/certs/spago/server.key                                                           0.0s
 => [stage-1 6/6] COPY --from=Builder /build/docker-entrypoint /docker-entrypoint                                                                               0.1s
 ✓  Build succeeded
 i  Your image won't be pushed. To push your image specify the flag '-t'.
```

We all know now how to build images . fun part you can pust to docker hub also its very similar to the docker build 

# why container for machine learning developement? 

## Bare necessities
-  Compute: High-performance CPUs and GPUs to train models.
-  Storage: For large training datasets and metadata you generated during training.
-  Frameworks and libraries: To provide APIs and execution environment for training.
-  Source control: For collaboration, backup, and automation.

## to fullfil our necessities we need portable envirnment & container's became defacto standard .

At some point in your machine learning development process, you’ll hit one of these two walls:

1. You’re experimenting and you have too many variations of your training scripts to run, and you’re bottlenecked by your single machine.
2. You’re running training on a large model with a large dataset, and it’s not feasible to run on your s
ingle machine and get results in a reasonable amount of time.

# Where okteto fits into this use case? 

- Okteto syncth the binaries over the okteto cloud 
- Developer need to build applications frequently
- rapid developement & delievery 
- okteto support technology that powered by kubernetes , helm 
- okteto dasbords to monitor the real time logs 
- free inbuild application 
- okteto supports CI/CD pipline thats okteto pipleline 
- okteto give specific amount of the resource quota include cpu , memory , storage that can boost to deploy pre-train model  
- okteto button on click deployement from github 
- deploy any github repo with specific branch that contains okteto manifiest 


#

understanding `okteto-stack.yaml ` & how spago works ?

# lets see the how spago works ? 

basically if you see spago library contains few inbuild CLI 

if you take look closer we are using entery point as `/docker-entrypoint` we can overide those binary & run the cli to run pre-train models 
spaGO supports two main use cases, which are explained more in detail in the following.

# CLI mode

Several programs can be leveraged to tour the current NLP capabilities in spaGO. A list of the demos now follows.

- Named Entities Recognition
- Hugging Face Importer
- Question Answering
- Masked Language Model


# Lets see application which use named entitie recognition 


```
# name of the service 
name: spago-okteto
services:
  spago:
    #specify its public or private 
    public: true
   #use image that already available at container registery   
    image: okteto.dev/spago
   #use build context to build images from the corrent direct if its located at another path specify path   
    build: .
   #okteto support command that you want overide entery point of the applicatiom    
    command:
    - sh
    - -c
    # use the command the talk with ner-server and run pre-train model by specifing --model flag & spago use grpc & protobuffers & --repo flag for specifying volumes and you can enable tls or disable as per use case 
    - /docker-entrypoint ner-server server --address=0.0.0.0:3000 --grpc-address=0.0.0.0:3001 --model=goflair-en-ner-fast-conll03-v0.4 --repo=/tmp/spago --tls-disable
    #specify port 
    ports:
      - 3000
    # define resource as per the okteto quota [note: as per the size of model & how much cpu need is really matter if your using pre-train models ]   
    resources:
      storage: 5Gi
      memory: 1Gi
      cpu: 500m
    # define the volumes   
    volumes:
      - /tmp/spago

```

why not add one more example using multi-container feature ! lets do it 

# easy way to deploy applications using docker compose over kubernetes 

okteto stack is something similar to docker compose easy to deploy application 


```
cat okteto-stack.yaml 
name: spago-okteto
services:
  spago:
    public: true
    image: okteto.dev/spago
    build: .
    command:
    - sh
    - -c
    - /docker-entrypoint ner-server server --address=0.0.0.0:3000 --grpc-address=0.0.0.0:3001 --model=goflair-en-ner-fast-conll03-v0.4 --repo=/tmp/spago --tls-disable
    ports:
      - 3000
    resources:
      storage: 5Gi
      memory: 1Gi
      cpu: 500m
    volumes:
      - /tmp/spago
  spago-2:
    public: true
    image: okteto.dev/spago
    build: .
    command:
    - sh
    - -c
    - /docker-entrypoint bert-server  server --address=0.0.0.0:4000 --grpc-address=0.0.0.0:4001 --model=deepset/bert-base-cased-squad2 --repo=/tmp/spago1 --tls-disable
    ports:
      - 4000
    resources:
      storage: 8Gi
      memory: 7Gi
      cpu: 800m
    volumes:
      - /tmp/spago1

```

Now we have two application one use ner-server & another bert server both build pre-train mode & ready to use with real time example .


lets deploy this real time application 

if you have not installed okteto cli follow refer :- [https://okteto.com/docs/getting-started/installation]
great now we have okteto cli installed lets verify cli working or not ?


```
okteto
Manage development containers

Usage:
  okteto [command]

Available Commands:
  analytics   Enable / Disable analytics
  build       Build (and optionally push) a Docker image
  create      Creates resources
  delete      Deletes resources
  doctor      Generates a zip file with the okteto logs
  down        Deactivates your development container
  exec        Execute a command in your development container
  help        Help about any command
  init        Automatically generates your okteto manifest file
  login       Log into Okteto
  namespace   Downloads k8s credentials for a namespace
  pipeline    Pipeline management commands
  push        Builds, pushes and redeploys source code to the target deployment
  restart     Restarts the deployments listed in the services field of the okteto manifest
  stack       Stack management commands
  status      Status of the synchronization process
  up          Activates your development container
  version     View the version of the okteto binary

Flags:
  -h, --help              help for okteto
  -l, --loglevel string   amount of information outputted (debug, info, warn, error) (default "warn")


```

# Now lets build okteto stack 


```
okteto stack deploy --build --wait 
 i  Running your build in tcp://buildkit.cloud.okteto.net:1234...
 i  Building image for service 'spago'...
[+] Building 146.9s (22/22) FINISHED                                                                                                                                 
 => [internal] load .dockerignore                                                                                                                               0.8s
 => => transferring context: 2B                                                                                                                                 0.7s
 => [internal] load build definition from buildkit-149758304                                                                                                    1.4s
 => => transferring dockerfile: 2.71kB                                                                                                                          1.4s
 => [internal] load metadata for docker.io/library/debian:buster-slim                                                                                           1.2s
 => [internal] load metadata for docker.io/library/golang:1.15.6-alpine3.12                                                                                     1.1s
 => [builder 1/9] FROM docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                       0.0s
 => => resolve docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                               0.0s
 => [stage-1 1/6] FROM docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                             0.0s
 => => resolve docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                                     0.0s
 => [internal] load build context                                                                                                                             113.7s
 => => transferring context: 21.66MB                                                                                                                          113.6s
 => CACHED [builder 2/9] RUN set -eux;  apk add --no-cache --virtual .build-deps   ca-certificates   gcc   musl-dev   openssl         ;                         0.0s
 => CACHED [builder 3/9] RUN adduser -S spago                                                                                                                   0.0s
 => CACHED [builder 4/9] RUN mkdir /build                                                                                                                       0.0s
 => [builder 5/9] ADD . /build/                                                                                                                                 0.3s
 => [builder 6/9] WORKDIR /build                                                                                                                                0.1s
 => [builder 7/9] RUN go mod download                                                                                                                           7.1s
 => [builder 8/9] RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-extldflags=-static" -o docker-entrypoint docker-entrypoint.go                  17.3s
 => [builder 9/9] RUN mkdir /etc/ssl/certs/spago  && openssl req   -x509   -nodes   -newkey rsa:2048   -keyout /etc/ssl/certs/spago/server.key   -out /etc/ssl  0.3s
 => CACHED [stage-1 2/6] COPY --from=Builder /etc/passwd /etc/passwd                                                                                            0.0s
 => CACHED [stage-1 3/6] COPY --from=Builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt                                              0.0s
 => [stage-1 4/6] COPY --from=Builder /etc/ssl/certs/spago/server.crt /etc/ssl/certs/spago/server.crt                                                           0.0s
 => [stage-1 5/6] COPY --from=Builder /etc/ssl/certs/spago/server.key /etc/ssl/certs/spago/server.key                                                           0.0s
 => [stage-1 6/6] COPY --from=Builder /build/docker-entrypoint /docker-entrypoint                                                                               0.1s
 => exporting to image                                                                                                                                          2.2s
 => => exporting layers                                                                                                                                         0.0s
 => => exporting manifest sha256:74432d6f5feff4f660857a43635e43180ba92bc6999d9ef5f078ed01f58b6d76                                                               0.0s
 => => exporting config sha256:a686bbcdad665ef54528f21f6f44e662e16dded3ebca320bd77337ace99408ba                                                                 0.0s
 => => pushing layers                                                                                                                                           1.4s
 => => pushing manifest for registry.cloud.okteto.net/sangam14/spago:latest                                                                                     0.7s
 => exporting cache                                                                                                                                             0.3s
 => => preparing build cache for export                                                                                                                         0.3s
 ✓  Image for service 'spago' successfully pushed
 i  Building image for service 'spago-2'...
[+] Building 7.5s (22/22) FINISHED                                                                                                                                   
 => [internal] load .dockerignore                                                                                                                               0.6s
 => => transferring context: 2B                                                                                                                                 0.6s
 => [internal] load build definition from buildkit-222011199                                                                                                    0.8s
 => => transferring dockerfile: 2.71kB                                                                                                                          0.8s
 => [internal] load metadata for docker.io/library/debian:buster-slim                                                                                           1.2s
 => [internal] load metadata for docker.io/library/golang:1.15.6-alpine3.12                                                                                     1.2s
 => [stage-1 1/6] FROM docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                             0.0s
 => => resolve docker.io/library/debian:buster-slim@sha256:59678da095929b237694b8cbdbe4818bb89a2918204da7fa0145dc4ba5ef22f9                                     0.0s
 => [internal] load build context                                                                                                                               2.7s
 => => transferring context: 49.45kB                                                                                                                            2.7s
 => [builder 1/9] FROM docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                       0.0s
 => => resolve docker.io/library/golang:1.15.6-alpine3.12@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                               0.0s
 => CACHED [builder 2/9] RUN set -eux;  apk add --no-cache --virtual .build-deps   ca-certificates   gcc   musl-dev   openssl         ;                         0.0s
 => CACHED [builder 3/9] RUN adduser -S spago                                                                                                                   0.0s
 => CACHED [builder 4/9] RUN mkdir /build                                                                                                                       0.0s
 => CACHED [builder 5/9] ADD . /build/                                                                                                                          0.0s
 => CACHED [builder 6/9] WORKDIR /build                                                                                                                         0.0s
 => CACHED [builder 7/9] RUN go mod download                                                                                                                    0.0s
 => CACHED [builder 8/9] RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-extldflags=-static" -o docker-entrypoint docker-entrypoint.go            0.0s
 => CACHED [builder 9/9] RUN mkdir /etc/ssl/certs/spago  && openssl req   -x509   -nodes   -newkey rsa:2048   -keyout /etc/ssl/certs/spago/server.key   -out /  0.0s
 => CACHED [stage-1 2/6] COPY --from=Builder /etc/passwd /etc/passwd                                                                                            0.0s
 => CACHED [stage-1 3/6] COPY --from=Builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt                                              0.0s
 => CACHED [stage-1 4/6] COPY --from=Builder /etc/ssl/certs/spago/server.crt /etc/ssl/certs/spago/server.crt                                                    0.0s
 => CACHED [stage-1 5/6] COPY --from=Builder /etc/ssl/certs/spago/server.key /etc/ssl/certs/spago/server.key                                                    0.0s
 => CACHED [stage-1 6/6] COPY --from=Builder /build/docker-entrypoint /docker-entrypoint                                                                        0.0s
 => exporting to image                                                                                                                                          1.6s
 => => exporting layers                                                                                                                                         0.0s
 => => exporting manifest sha256:27aac3abc82baf879d2cfd222d5a262c0fd335ea22bb14ecf6091a01b9ee5bf1                                                               0.0s
 => => exporting config sha256:d9dc98cc4e032714b219261220eb5ed7e95f40cc998799ba6f6d2d45a6190d81                                                                 0.0s
 => => pushing layers                                                                                                                                           1.1s
 => => pushing manifest for registry.cloud.okteto.net/sangam14/spago:latest                                                                                     0.5s
 => exporting cache                                                                                                                                             0.4s
 => => preparing build cache for export                                                                                                                         0.4s
 ✓  Image for service 'spago-2' successfully pushed
 ✓  Successfully deployed stack 'spago-okteto'
```

wait for few min to download pre-train model .... & application is live 

example 1 :- https://spago-sangam14.cloud.okteto.net/ner-ui


example2 :- https://spago-2-sangam14.cloud.okteto.net/bert-qa-ui


both application deployed successfully 


# fun demos ! time with power of okteto cloud 
if you like this tag @okteto & @BiradarSangam and share your fun demo with as with example 2 

 
