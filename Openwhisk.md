# Deploying-Openwhisk-on-Kubernetes

## Apache Openwhisk Introduction
Apache OpenWhisk is an open-source, distributed Serverless platform that executes functions (fx) in response to events at any scale. OpenWhisk manages the infrastructure, servers, and scaling using Docker containers so you can focus on building amazing and efficient applications.

### OpenWhisk Programming Model
![image](https://user-images.githubusercontent.com/37688219/160309490-47484078-253b-4b01-90a2-c0839f6f6220.png)


#### **Event** 
* Events drive the Serverless execution of functional code called Actions. 
* Events can come from any Event Source or Feed service including, datastores, Message Queues, Mobile and Web Applications, Sensors, Chatbots, Scheduled tasks (via Alarms), etc.

#### **Actions** 
* Actions are stateless functions (code snippets) that run on the OpenWhisk platform. 
* Actions encapsulate application logic to be executed in response to events. 

#### **Trigger** 
* Triggers are named channels for classes or kinds of events sent from Event Sources.

#### **Rule** 
* Rules are used to associate one trigger with one action. After this kind of association is created, each time a trigger event is fired, the action is invoked.


### Deploy
* Apache OpenWhisk builds its components using containers it easily supports many deployment options both locally and within Cloud infrastructures. 
* Options include many of today's popular Container frameworks such as Kubernetes and OpenShift, and Compose. In general, the community endorses deployment on Kubernetes using Helm charts since it provides many easy and convenient implementations for both Developers and Operators alike.

### OpenWhisk Architecture
[Openwhisk Working](https://github.com/apache/openwhisk/blob/master/docs/about.md#how-openWhisk-works)

![image](https://user-images.githubusercontent.com/37688219/160309511-f66d9f79-e071-43a4-8429-c3ef878f0690.png)


#### **Entering the system: Nginx**
* An HTTP and reverse proxy server.  It is mainly used for SSL termination and forwarding appropriate HTTP calls to the next component.
* The command sent via the wsk CLI is essentially an HTTP request against the OpenWhisk system. 

#### **Entering the system: Controller**
* Nginx forwards the HTTP request to the Controller.
* Controller is a Scala-based implementation of the actual REST API and thus serves as the interface for everything that you want to do.
* When the HTTP request is forwarded, the controller first determines what action that you are trying to take, based on the HTTP method that you used in your HTTP request.
* Given the central role of the Controller, the following steps will all involve it to a certain extent.

#### **Authentication and Authorization: CouchDB**
* The Controller verifies who you are and if you have the privilege to do what you want to do with that entity (Authorization).
* The credentials included in the request are verified against the so-called subjects database in a CouchDB instance. 
* It is checked that the user exists in OpenWhisk’s database and that it has the privilege to invoke the action. The latter effectively gives the user the privilege to invoke the action

#### **Getting the action: CouchDB… again**
* As the controller is now sure the user is allowed in and has the privileges to invoke his action, it actually loads this action from the whisks database in CouchDB.
* The record of the action contains mainly the code to execute and default parameters that you want to pass to your action, merged with the parameters you included in the actual invoke request. It also contains the resource restrictions imposed on it in execution, such as the memory it is allowed to consume.

#### **Who’s there to invoke the action: Load Balancer**
* The Load Balancer, which is part of the Controller, has a global view of the executors available in the system by checking their health status continuously. Those executors are called Invokers. 
* The Load Balancer, knowing which Invokers are available, chooses one of them to invoke the action requested

#### **Please form a line: Kafka**
* Controller and Invoker solely communicate through messages buffered and persisted by Kafka. That lifts the burden of buffering in memory, risking an OutOfMemoryException, off of both the Controller and the Invoker while also making sure that messages are not lost in case the system crashes.
* To get the action invoked then, the Controller publishes a message to Kafka, which contains the action to invoke and the parameters to pass to that action (in this case none). This message is addressed to the Invoker which the Controller chose above from the list of available invokers.
* Once Kafka has confirmed that it got the message, the HTTP request to the user is responded to with an ActivationId. The user will use that later on, to get access to the results of this specific invocation


#### **Actually invoking the code already: Invoker**
* The Invoker’s duty is to invoke an action. To execute actions in an isolated and safe way it uses Docker.

#### **Storing the results: CouchDB again**
* As the result is obtained by the Invoker, it is stored into the activations database as an activation under the ActivationId mentioned further above. The activations database lives in CouchDB.
Now you can use the REST API again (start from step 1 again) to obtain your activation and thus the result of your action
