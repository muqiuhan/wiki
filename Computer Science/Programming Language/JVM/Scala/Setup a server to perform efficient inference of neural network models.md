#scala #fp #machine-learning #algorithm

# Overview

This is the first in series of posts which will cover how one might setup a server to perform efficient inference of neural network models on both CPUs and GPUs using the [Scala](https://www.scala-lang.org/) programming language. This entry will introduce key concepts at a high level as well as introduce the baseline neural network model we will use throughout the series.

Note that throughout this series certain concept or technologies will be mentioned but not explained in detail. This is because most of these technologies are deep and complex in their own right and there is simply not enough time to discuss them here. Instead we will provide a brief description, a link to find more information and a justification as to why that technology is important.

One such topic is machine learning and neural networks as a whole. The motivation of this series is not to become an expert on machine learning or neural networks. In fact the training of neural networks will not be covered in any capacity. We are simply interested with the integration of a pre-trained network into a live application. We model a given network as a [pure function](https://en.wikipedia.org/wiki/Pure_function) which accepts an input tensor A as well as weights W, and returns an output B.

# Intended Audience

These posts will be geared towards those with some experience with Scala, specifically with experience with the [Typelevel](https://typelevel.org/) ecosystem. The idea is that this series will help provide a happy path to getting started with high performance machine learning inference, for developers using this stack. However if you do not have a ton of experience with Scala or Typelevel, you should still be able to follow along as we will link to relevant documentation, and tools like Triton, gRPC and ONNX are language agnostic, so the knowledge you gain on these topics will be transferable to other languages.

# Motivation

The motivation for these posts is two-fold. First is that neural networks are increasingly common part of solutions in just about every technical domain, and thus it is important to leverage them in a efficient manner. Solutions such as [AWS Sagemaker](https://aws.amazon.com/pm/sagemaker/), [Digital Ocean Paperspace](https://www.paperspace.com/) and [Azure Machine Learning](https://azure.microsoft.com/en-us/products/machine-learning/) exist to fill this gap, but there are reasons you would not want to use those services, whether it be organization rules on [data governance](https://en.wikipedia.org/wiki/Data_governance), the avoidance of vendor lock or simply that those services aren’t an appropriate solution to your specific problem. The second motivation is simply to learn. Taking a problem, approaching it from multiple angles and then continuously refining and analyzing our solutions is a great way to gain a deep understanding of a certain domain.

# Baseline Model

We will use the well studied problem of image classification as our baseline problem. Image classification has uses in everywhere from self-driving vehicles to [medical image analysis](https://arxiv.org/abs/2202.08546). [YOLO](https://arxiv.org/abs/1506.02640) (You Only Look Once) is a family of vision models can be used to address tasks such as [classification, detection and segmentation](https://docs.ultralytics.com/tasks/). At the time of writing the most recent version of YOLO is YOLOv8 and that is the model we will use throughout this series. [YOLOv8](https://arxiv.org/abs/2305.09972) was trained on the [ImageNet](https://www.image-net.org/), which is a dataset of hierarchically organized concepts into nodes in a tree. Each node has around 1000 images that relate to it. Our systems will take an image as input and return the top K predictions for the label that best describes the image, along with the probability assigned to that label. The diagram below shows an example of the inference process at a high level. A user sends a picture of a golden retriever as an HTTP request via curl, the service processes the request and returns a map from the image name to an ordered list of tuples where the first element is the assigned probability that the image belongs to the label, which is the second element.

_Simplified Inference Process. A request is made to our live service and a mapping of the image to a sorted list of pairs is returned._

# Concepts

### Throughput[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#throughput)

The total amount of requests that a system can process over a period of time. Systems are often measured using throughput as high throughput systems scale better in general. Consider a case in which you expect to be be receiving ~1000 requests/second for a sustained period of time and you want each request to take no more than one second. If your system has a measured throughput of 100 requests/second you now know that you will need to run and load balance along at least 10 instances of your hypothetical service. Increasing the throughput of your system will reduce the number of instances you need to run.

### Latency[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#latency)

The total time in-between when a network request is sent to a system, and when a response is received. As programmers we generally want to develop low latency systems. What is determined as low enough depends on a use case. For a search engine like google or an internet database like [IMDB](https://www.imdb.com/), a few fractions of a second of latency is often low enough, as humans tend to perceive that as instantaneous. However for something like a self driving car or an application that deals with high frequency financial data, a few milliseconds of latency may be the target. We care about latency as that will be one of the metrics we use to benchmark our systems.

### CPU[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#cpu)

Central Processing Unit. The main processor on a given machine. Unless explicitly specified otherwise the instructions generated by a programming language will run on this processor.

### GPU[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#gpu)

Graphical Processing Unit. A multi-core processor capable of efficiently performing [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel) tasks.  We care about them as tensor operations commonly found in neural networks are often embarrassingly parallel and thus the training and evaluation of neural networks is greatly accelerated by a GPU.

### Protocol Buffers[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#protocol-buffers)

[Protocol Buffers are language-neutral, platform-neutral extensible mechanisms for serializing structured data](https://protobuf.dev/). We care about them as they are a building block for both gRPC and the ONNX file format, and they allow for type safe network RPC calls.

### gRPC[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#grpc)

[Googles Remote Procedure Call framework](https://grpc.io/) is a generalized method for different systems to communicate with each other. Data is serialized using Protocol Buffers and is then sent across network boundaries in an extremely efficient way. Aside from being very performant gRPC is also driven by a specification language designed to have code generation tools built around it. This means that you can expose a gRPC server written in Scala and then generate a client (stub) in python, Ruby, Go or any other language to interact with the Scala service. It is important to note that gRPC is not supported by browsers and is generally used by back-end services to communicate with each other.

### ONNX[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#onnx)

[Open Neural Network Exchange](https://onnx.ai/) is an open format used to describe a neural network. This format is de-coupled from the actual runtime that executes the network. The ability to swap out back-ends is very powerful as different problems will have different hardware constraints. If you do not have GPUs available you can use the default CPU runtime. If you do have access to GPUs then you can use TensorRT or a [CUDA](https://developer.nvidia.com/cuda-toolkit) runtime. As advances in the machine learning field advance more performant runtimes may be developed and if they choose to support ONNX then you will be able to use them with little to no refactoring.  Most frameworks for developing and training neural networks such as PyTorch, TensorFlow or MXNet support exporting to ONNX format. This means you can have multiple different projects using different frameworks to develop your deliverables and by exporting them to ONNX you wont have to change your deployment strategy.

### [TensorRT](https://github.com/NVIDIA/TensorRT)[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#tensorrt)

An SDK developed by Nvidia to optimize and accelerate inference on GPUs. Built on top of CUDA libraries. We want to use it as it can provide state of the art latency and throughput.

### In-flight Batching[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#in-flight-batching)

When performing model inference a neural network can either process one tensor at a time, or it can process a batch of tensors. Due to the parallel nature of neural computation processing a batch of tensors is often much more efficient. When deploying a live service that can take requests from multiple sources, we may not have enough data to “fill” a batch with any given request. In-flight batching is the process of taking requests that arrive at roughly the same time and dynamically batch them together.

### [Triton Inference Server](https://github.com/triton-inference-server/server)[](https://mattlangsenkamp.github.io/posts/scala-machine-learning-deployment-entry-0/#triton-inference-server)

A model deployment server developed by Nvidia. It takes a model and creates either HTTP or gRPC endpoints to serve requests. It can target different back-ends such as ONNX or TensorRT. It also has the ability to perform inflight batching.