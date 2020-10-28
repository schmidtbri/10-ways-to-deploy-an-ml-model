Title: 10 Ways to Deploy a Machine Learning Model
Date: 2020-10-28 08:00
Category: Blog
Slug: 10-ways-to-deploy-an-ml-model
Authors: Brian Schmidt
Summary: In previous blog posts we've seen how it is possible to deploy the same model in ten different ways. The model itself was developed one time and released as a package, which was then used in each deployment. These blog posts started as an exercise in finding new and interesting ways to deploy an ML model, so we decided to write this blog post about some of the things that we've learned along the way. In order to be able to deploy the same model in 10 different ways, we needed to build the model so that it was not incompatible with all the different ways we wanted to deploy it. We also needed to make it easy to install and to make sure that the model published metadata about itself. All of these features of the model became very important once we needed to deploy it into a real software system.


This blog post builds on the ideas started in
[three]({filename}/articles/a-simple-ml-model-base-class/post.md)
[previous]({filename}/articles/improving-the-mlmodel-base-class/post.md)
[blog
posts]({filename}/articles/using-ml-model-abc/post.md).

In this blog post I'll show how to deploy the same ML model that we
deployed as a batch job in this [blog
post]({filename}/articles/etl-job-ml-model-deployment/post.md),
as a task queue in this [blog
post]({filename}/articles/task-queue-ml-model-deployment/post.md),
inside an AWS Lambda in this [blog
post]({filename}/articles/lambda-ml-model-deployment/post.md),
as a Kafka streaming application in this [blog
post]({filename}/articles/streaming-ml-model-deployment/post.md),
a gRPC service in this [blog
post]({filename}/articles/grpc-ml-model-deployment/post.md),
as a MapReduce job in this [blog
post]({filename}/articles/map-reduce-ml-model-deployment/post.md),
as a Websocket service in this [blog
post]({filename}/articles/websocket-ml-model-deployment/post.md),
as a ZeroRPC service in this [blog
post]({filename}/articles/zerorpc-ml-model-deployment/post.md),
and as an Apache Beam job in this [blog
post]({filename}/articles/apache-beam-ml-model-deployment/post.md).

Introduction
============

In previous blog posts we've seen how it is possible to deploy the same
model in ten different ways. The model itself was developed one time and
released as a package, which was then used in each deployment. These
blog posts started as an exercise in finding new and interesting ways to
deploy an ML model, so we decided to write this blog post about some of
the things that we've learned along the way.

In order to be able to deploy the same model in 10 different ways, we
needed to build the model so that it was not incompatible with all the
different ways we wanted to deploy it. We also needed to make it easy to
install and to make sure that the model published metadata about itself.
All of these features of the model became very important once we needed
to deploy it into a real software system.

In a [previous blog
post]({filename}/articles/improving-the-mlmodel-base-class/post.md),
we developed a model that we called the "iris\_model". This model was
designed for the purposes of the blog posts that we planned to write
later on, so it followed several best practices that we will be
describing in this blog post. To make sure that the model was compatible
with every deployment option we wanted to pursue, we needed to build it
to work as a software component, as a software library, and as a
software package. In this blog post we'll describe how and why these
approaches make it easier to deploy the model.

To be able to abstract away the details of an ML model from the code
that is using it, we developed the MLModel base class in
[these]({filename}/articles/a-simple-ml-model-base-class/post.md)
[blog
posts]({filename}/articles/improving-the-mlmodel-base-class/post.md).
The base class is used to create a standard interface for the prediction
code of an ML model, which makes it easier to deploy the model. This
approach made it possible to write the model deployment code in such a
way that it can support any model that implements the MLModel interface.
This approach can be thought of as applying the strategy design pattern
to machine learning models. In this blog post we'll describe how the
strategy pattern is useful in ML model deployments.

When we started implementing all of the different deployments for the
model, we started seeing patterns around the way that the model is
accessible to its clients. These patterns coalesced into a few different
classes of model deployments which help to talk about the strengths and
weaknesses of each approach to deploying the model. In this blog post,
we'll describe an ontology that can help developers to talk about and
choose the best approach to deploying an ML model.

ML Models as Software Components
================================

To create an ML model that is easy to deploy, we need to build it as a
software component. A software component is simply a small part of a
bigger software system that can be easily isolated from the rest of the
system. That is to say, the component is not deeply tied to the rest of
the system and it exposes an interface so that the rest of the system
can access it. A software component is designed to fulfill a small part
of the requirements of a larger software system, and to be easy to
integrate with other software components in the system. Good software
components are designed to be reused in many contexts and must follow
good design patterns to achieve this goal.

One of the most important parts of a software component is the public
API of the component. The API of the IrisModel class has proven to be
very simple and adaptable to a wide variety of technologies. For
example, when we deployed the IrisModel as a [Websocket
service]({filename}/articles/websocket-ml-model-deployment/post.md),
we didn't need to rewrite any of the model code to adapt it to the model
component's API. The reason for this is that the [IrisModel
class](https://github.com/schmidtbri/ml-model-abc-improvements/blob/master/iris_model/iris_predict.py#L10-L67)
inherits from the [MLModel
interface](https://github.com/schmidtbri/ml-model-abc-improvements/blob/master/iris_model/iris_predict.py#L10-L67).
This interface has a few requirements: your model must instantiate
itself, it must receive prediction requests, and it must publish certain
metadata about itself. By creating a standard interface around these
requirements, the MLModel interface makes it possible to deploy a wide
range of machine learning models in the same way.

When we designed the MLModel interface we made sure that it would not
enforce any specific technology on the user. For example, there is no
requirement that says that the models that implement the MLModel
interface must use a specific serialization and deserialization
standard. In all of the blog posts where we deployed the iris\_model
package we used JSON for serialization and deserialization, but this was
an implementation detail that can easily be changed since the model code
itself does not do any serialization or deserialization. Another
important aspect of the design is the fact that the MLModel interface
does not enforce any particular integration pattern on the code. For
example, we were able to create a [RESTful
service]({filename}/articles/using-ml-model-abc/post.md)
and [a batch
job]({filename}/articles/etl-job-ml-model-deployment/post.md)
with the same model. In fact, the choice of deployment technology had no
effect on the model codebase. This makes it possible to reuse the same
model in many different contexts.

Certain technologies required advanced knowledge of the schema of the
data that the model component would receive and send back. For example,
the [gRPC
service]({filename}/articles/grpc-ml-model-deployment/post.md)
required that we compile a protocol buffer from the input and output
schemas of the model. In this case we were able to isolate the
requirements of the deployment from the model itself by leveraging the
schema metadata provided by the model. In other cases, the schema
metadata was only useful for documentation purposes, since a user of the
model would need to know about the model's input and about schemas to be
able to use it. Because we return schema information from the API of the
ML model software component, we were able to handle this situation
smoothly.

ML Models as Libraries
======================

To create an ML model that is easy to deploy, we must build it so that
it works as a software library. A software library is a collection of
reusable software components that can be used in many different
contexts. A library is designed and built so that it is reusable.

By treating a machine learning model as a library we gain many different
benefits, for example, models can easily be reused in many different
services and applications without having to copy and paste the model
code and parameters. There is no need to embed an ML model inside of a
codebase in such a way that it cannot be reused somewhere else because
the library can be installed into a project. When we used the
iris\_model library in our deployments, all we had to do was execute
"from iris\_model.iris\_predict import IrisModel" and the model would be
available to be used.

Another benefit that we gain when we treat ML models as libraries is
that it is easy to version them. Since libraries are built and released
many times, everyone understands how to version them and release them
for use by other developers. The semantic versioning standard has been
used widely in the software world and we used it to version the
iris\_model package. One of the main benefits of a strong versioning
standard for ML models is that everyone understands that the ML model
will be evolving in the future, and that they can access newer versions
of the model by installing a newer version of the library.

By thinking about ML models as libraries we break the pattern of making
custom models for very specific use cases. If we are going to spend the
time and effort to build a complex ML model, why not make it easy to
reuse in different contexts? This requires a bit of realignment in most
cases, but it is certainly possible.

ML Models as Packages
=====================

To create an ML model that is easy to deploy, we must build it so that
is a software package. A software package is a distributable file that
contains the necessary files to install a software component or library
in the programming environment. Software packages are usually managed
using package managers. Software libraries are usually released as
packages as well, to make them easy to install.

One of the most important factors that allowed us to deploy the
IrisModel model in 10 different ways is the fact that the model code is
isolated inside of a Python package. The first two blog posts were
concerned with creating a model codebase that could be installed into
any python environment. Once we could install the model as a python
package with the pip install command, it was easy to reuse the same
model in many different contexts.

An important part of this approach is the fact that we can install all
of the dependencies of the model package automatically when the model
package is installed. Often, a model that runs in one person's computer
won't run in another person's computer because dependency management is
not taken care of. In order to create a python package, the dependencies
of the package must be listed in the setup.py file of the Python
project, because of this the ML model is a lot easier to work with and
can be easily installed by anybody. For example, the iris\_model package
lists the exact version of scikit-learn that it needs, which takes the
guesswork out of installing and using it.

Lastly, by distributing the ML model as a package, we're able to
download and install the model parameters along with the model code.
Oftentimes, an ML model is just a file that contains serialized model
parameters (often a pickle file). However, distributing a model this way
ignores the fact that we might need to install some custom prediction
code along with the model parameters. By using a package manager, we are
able to ensure that the model parameters and the prediction codebase are
installed correctly into the programming environment. In the case of the
IrisModel package, the model parameters were installed by including the
file in the package's manifest which ensures that the parameters are
copied into the distributable file.

ML Models and the Strategy Pattern
==================================

The strategy pattern is a design pattern used in object oriented design.
It is a behavioral design pattern that allows a software component to
select an appropriate algorithm at runtime to execute a task. The
strategy pattern is applied by defining an interface that every
implementation of the strategy must inherit and implement. The MLModel
class that the IrisModel class inherits from fulfills this purpose. The
benefit that we gain from using the strategy pattern is that we can
write code that doesn't care about the details of a machine learning
model's prediction algorithm, because it can use any algorithm that
meets the requirements of the interface.

In practice, this means that we were able to deploy an ML model simply
by installing the package and writing a reference to the class that
implements the MLModel interface into the configuration. The deployment
code reads the configuration at runtime, loads the right model, and
makes it available to the client. Some model deployments that we built
were even able to handle multiple models. For example, the ZeroRPC
service that we created in [this blog
post]({filename}/articles/zerorpc-ml-model-deployment/post.md)
is able to dynamically create an endpoint for every model that is listed
in the configuration.

By creating models as components and making them available as packages,
we're able to make models reusable in many different situations. When we
use the strategy pattern, we get a similar benefit, because the pattern
makes it possible to reuse the model deployment code to deploy any model
in the future. As long as the model we want to deploy implements the
MLModel interface, we are able to reuse the deployment codebase to
deploy it. In the future, it would be easy to build reusable codebases
that can deploy models, the code would be configured with the model that
needs to be deployed and there would be no need to create a custom
service for each model that wanted to deploy.

An Ontology of ML Model Deployments
===================================

Now that we have deployed the same model in ten different ways, we can
compare and contrast the ways the model was deployed. This section tries
to build a complete picture of the effect that a deployment option can
have on the way we can use the model.

Interactive and Non-Interactive Model Deployments
-------------------------------------------------

ML models can be deployed in an interactive manner and a non-interactive
manner. A model is deployed "interactively" when a client of the model
is able to request predictions from the model and get a prediction
directly back without waiting an indeterminate amount of time to get the
prediction. Interactive model deployments make the model directly
available to the client through an API and make it possible for the
client to send in any data allowed by the model's input schema to make a
prediction. In "non-interactive" model deployments, the client is not
able to send data to the model directly, which usually means that the
client has to access predictions that were previously stored in a data
store. The distinction between interactive and non-interactive model
deployments can have a large impact on the design of the client systems
that make use of the ML model. If a model is deployed non-interactively,
the clients of the system don't have direct access to the model and they
can't send any data they want to the model, the only predictions that
are available from the model are the ones previously made and stored.

An example of an interactive deployment is the REST service that we
built in [this blog
post]({filename}/articles/using-ml-model-abc/post.md).
The service is designed to run continuously, which means that a client
can contact the service anytime, request a prediction, and get a
prediction back directly from the model. An example of a non-interactive
deployment is the batch job that we built in this [blog
post]({filename}/articles/etl-job-ml-model-deployment/post.md),
since a user of the model can only access the predictions that are saved
by the batch job. At first sight, it would seem that the task queue
deployment that we built in [this blog
post]({filename}/articles/task-queue-ml-model-deployment/post.md)
is non-interactive because the user has to wait to get a prediction.
However, the task queue is actually interactive because the predictions
are always made from the input provided by the client and the
predictions become available to the client after the asynchronous task
completes.

Single-Record and Batch Model Deployments
-----------------------------------------

Single-record model deployments are designed to receive inputs from
clients, make a single prediction, and return the results to the client.
Batch model deployments are designed to receive many inputs from the
client system, make predictions and return the results to the client as
a batch of records. Batch systems often make better use of resources
because they are able to
[vectorize](https://en.wikipedia.org/wiki/Array_programming)
their operations, this makes their operation more efficient.
Single-record systems are usually more responsive to clients because
they are able to quickly return a result.

System performance can be measured in two ways: throughput and latency.
Throughput is defined as the number of records that can be processed by
the system in a given period of time. Latency is the amount of time it
takes the system to process a single request. A single-record model
deployment is often optimizing for the total latency of a single
request, and a batch model deployment is often optimizing for the total
throughput of the system.

An example of a single-record model deployment is the gRPC service that
we built in this [blog
post]({filename}/articles/grpc-ml-model-deployment/post.md).
The gRPC only allows one prediction to be made for each RPC call to the
model, this is enforced in the protocol buffer interface definition of
the service which does not allow arrays of prediction inputs to be
received by the service. An example of a batch model deployment is the
MapReduce job we built in this [blog
post]({filename}/articles/map-reduce-ml-model-deployment/post.md).
The MapReduce system is specifically designed to allow massive parallel
batch jobs that run across multiple computers in a cluster. The system
is most efficient when processing large datasets because of the amount
of time it takes to start a processing run. The distinction between
single-record and batch deployments can sometimes be hard to draw
because we can support multiple predictions in the gRPC service API, as
long as the client is willing to wait for all of the predictions to
complete. As always, there are many tradeoffs that we can make between
the two extremes.

Synchronous and Asynchronous Model Deployments
----------------------------------------------

Synchronous ML model deployments are characterized by the client being
blocked while the model is making a prediction. An asynchronous model
deployment allows the client system to request a prediction from the
model and not wait for the prediction to complete to continue
processing. Typically, an asynchronous deployment allows the client to
retrieve the model's prediction after it completes, but this is not
required for the system to be considered asynchronous. The predictions
made by a synchronous model deployment are returned to the client as
soon as they are completed.

An example of a synchronous model deployment is the AWS Lambda
deployment we built in this [blog
post]({filename}/articles/lambda-ml-model-deployment/post.md).
The Lambda receives prediction requests through an AWS API Gateway,
makes a prediction and returns it while the client system waits for it.
An example of an asynchronous model deployment is the task queue we
built for this [blog
post]({filename}/articles/task-queue-ml-model-deployment/post.md).
The task queue is specifically designed to receive predictions requests
from clients and fulfill them while the client system works on other
things. The task queue makes the prediction available to the client in a
"result backend" which can be accessed by the client once the prediction
is completed. Another asynchronous deployment is the Kafka stream
processor we built in this [blog
post]({filename}/articles/streaming-ml-model-deployment/post.md),
although it is not designed to return the prediction results directly to
the client like the task queue deployment.

Real-time and Non-real-time Model Deployments
---------------------------------------------

Another area of optimization for ML model deployments is the ability to
return a prediction very quickly. A real-time system needs to be
optimized to have very low and very predictable latency so that we can
ensure that interactions with the model can always happen quickly and
end within a defined period of time.

An example of a real time model deployment is the Websocket service that
we created in [this blog
post.]({filename}/articles/websocket-ml-model-deployment/post.md)
The Websocket service is particularly useful for this type of deployment
because websocket connections are designed to transfer data with very
low overhead. Some examples of a non-real-time service is the Apache
Beam ETL job we built in [this blog
post]({filename}/articles/apache-beam-ml-model-deployment/post.md)
and the Hadoop MapReduce job we built in [this blog
post]({filename}/articles/map-reduce-ml-model-deployment/post.md).
These deployments are designed to make millions of predictions and are
optimized for that purpose, which means that they are not useful in
situations in which we need real-time predictions.

In the blog posts that we wrote, we didn't try to deploy a model on a
consumer device like a phone or tablet. All of the approaches we took
were designed to execute the model on a server and return the prediction
to the client through the network. For a real-time system, being able to
execute directly on the client device would be more efficient and faster
since no network hop is required.

Deterministic and Non-deterministic Models
------------------------------------------

The last distinction we will make is between deterministic and
nondeterministic model prediction code. Deterministic models will always
return the same result when given the same input, non-deterministic
models can return different results when given the same input. This
distinction can have a large impact on the deployment of the model. If
we don't distinguish between models that are deterministic and
non-deterministic, doing things like storing predictions for later use
and prediction caching can become much more complicated. Any model that
is being deployed that is non-deterministic should publish that fact to
its users so that they can be ready to deal with the side effects of
non-determinism.

Conclusion
==========

At the beginning of this series of blog posts we challenged ourselves to
come up with a simple base class that would enable us to abstract out
the details of a machine learning model. We started by creating a base
class that could hide the details of the ML model behind an abstraction,
then added features that we thought would be useful. From the beginning,
the base class was designed to make it easy to deploy machine learning
models. The base class was not designed for the training parts of a
model codebase.

To be able to introspect details about the model, we also added the
ability for the model to provide metadata about itself. The metadata
aspect of the model was not really required for most model deployments,
but it did become important for certain deployments. Model metadata like
the version and the input and output schemas of the model becomes more
important when we have to manage dozens or hundreds of deployed models.

To enable us to easily deploy any ML model, we also needed to make the
model codebase easy to install, which we accomplished by making the ML
model into a Python package that could be installed with the pip package
manager. By making the model codebase easy to install we enabled anybody
to reuse the model in whichever context they needed it without having to
understand the code or manually install the dependencies of the model.
Having the model inside of a package also allowed us to install the very
same model in 10 different applications with no changes to the model
code.

Overall, this series of blog posts is much less concerned with the
details of training a machine learning model. It is mainly concerned
with integrating the trained ML model with other software systems. To
this end, we sought to use a wide variety of integration technologies to
make sure that our approach worked in every situation. In every case,
the model codebase remained the same and we did not have to adapt it to
any of the integrations. This speaks to the flexibility of the approach,
which allowed us to isolate the details of the ML model from the
deployment and integration problems. Furthermore, we can reuse any of
the deployment codebases to deploy any ML model code that implements the
MLModel base class, which makes the deployment codebases reusable as
well.

To sum up, the best strategy for building an ML model that can be used
in many different contexts is to: code the model prediction code behind
an interface, build and release the model as a package, and then to
install it into the environment where it will be used. All deployment
details should be kept out of the model package so that we are able
choose the right approach to model deployment later on.
