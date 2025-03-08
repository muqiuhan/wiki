#scala #fp #machine-learning

# Overview

This is the second post in a series about inference of machine learning models using scala. The first post can be found [here](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/). This post will detail how to use a functional streaming library ([fs2](https://fs2.io/#/)) to perform machine learning model inference using the [Triton Inference Server](https://github.com/triton-inference-server/server) and [gRPC](https://grpc.io/). The post will be broken up into a few different parts. First we will set up our scala, python and docker dependencies. Then we will get Triton up and running using Docker. Finally we will set up fs2 to read from a text file containing image paths. We will use [opencv](https://opencv.org/) to format our images into the representation Triton expects. Finally we will load images and send them to Triton in batches, displaying the result to the console.

The github repo for these tutorials can be found [here](https://github.com/MattLangsenkamp/scala-machine-learning-deployment)

## Setup[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-1/#setup)

It is expected that you have the following tools installed:

- scala build tool [sbt](https://www.scala-sbt.org/)
- python build tool [poetry](https://python-poetry.org/)
- Cuda toolkit and Nvidia Docker. More detailed installation tips can be found in the [github readme](https://github.com/MattLangsenkamp/scala-machine-learning-deployment)

### Scala and File Directory[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-1/#scala-and-file-directory)

First create a new project using the scala 3 giter template/sbt and move to the newly created directory.

```Scala
sbt new scala/scala3.g8 
#   name [Scala 3 Project Template]: scalamachinelearningdeployment 
#   Template applied in ./scalamachinelearningdeployment cd scalamachinelearningdeployment`
```
Next we will add the fs2 gRPC plugin. Add the following to `project/plugins.sbt`. This is what will turn our `.proto` files into code we can use to talk with Triton. We will talk more about `.proto` files and gRPC later.
```scala
addSbtPlugin("org.typelevel" % "sbt-fs2-grpc" % "2.7.4")`
```
We then need to create a module to store our `.proto` files in, and to run code generation from.
```scala
mkdir -p protobuf/src/main/protobuf/`
```
Create a file called `downloadprotos.sh` and add the following content. These are the proto files provided by the Triton Inference Server. They allow for us to communicate with Triton in any language that can generate code from `.proto` files.
```shell
for PROTO in 'grpc_service' 'health' 'model_config' 
do     
  wget -O ./protobuf/src/main/protobuf/$PROTO.proto https://raw.githubusercontent.com/triton-inference-server/common/main/protobuf/$PROTO.proto 
done
```
Then run the script to download the files.
```shell
chmod +x downloadprotos.sh
./downloadprotos.sh`
```
Finally we need to configure our `build.sbt`. There are a couple key steps to make note of:

1. Create variables to manage our dependencies
2. Create a module for the protobuf subdirectory, explicitly stating we depend on the gRPC plugin
3. Add our dependencies to our root module and make the root module depend to the protobuf module

```scala

```