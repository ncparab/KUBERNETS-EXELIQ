#######################
KOPS Installation
#######################

Cloud Agnostic Kubernetes Cluster setup:
----------------------------------------

KOPS installation
==================

- Install Kops:

.. code-block:: bash

   $ curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
   $ chmod +x kops-linux-amd64
   $ sudo mv kops-linux-amd64 /usr/local/bin/kops

.. image:: kubeadm/kops1.PNG
   :width: 800px
   :height: 150px
   :alt: alternate text

Create an AWS S3 bucket for kops to persist the cluster state:

.. image:: kubeadm/kops2.PNG
   :width: 800px
   :height: 200px
   :alt: alternate text
   
.. image:: kubeadm/kops3.PNG
   :width: 800px
   :height: 200px
   :alt: alternate text

Enable versioning for the above S3 bucket:
 
.. image:: kubeadm/kops4.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

Provide a name for the Kubernetes cluster and set the S3 bucket URL in the following environment variables:

.. code-block:: bash

   $ export KOPS_CLUSTER_NAME=exeliq.k8s.local
   $ export KOPS_STATE_STORE=s3:// exeliq-kops-cluster

Create a Kubernetes cluster definition using kops by providing the required node count, node size, and AWS zones. The node size or rather the EC2 instance type would need to be decided according to the workload that you are planning to run on the Kubernetes cluster:

.. code-block:: bash

   $ sudo kops create cluster --name=exeliq.k8s.local --state s3://exeliq-kops-cluster --zones=us-east-1a --yes

.. image:: kubeadm/kops5.PNG
   :width: 800px
   :height: 500px
   :alt: alternate text

Meanwhile, AWS Ec2 instances will be launched in the specified Regoin – master of type C4.large by default.

.. image:: kubeadm/kops6.PNG
   :width: 800px
   :height: 200px
   :alt: alternate text

Once the instances are registered with the master, Validate the cluster

.. code-block:: bash

   $ kops validate cluster --name= exeliq.k8s.local

.. image:: kubeadm/kops7.PNG
   :width: 800px
   :height: 200px
   :alt: alternate text
   
Ensure All of the Kubernetes daemons are up and running.

.. image:: kubeadm/kops8.PNG
   :width: 800px
   :height: 200px
   :alt: alternate text

Create deployment/run the Application specifying the image

.. code-block:: bash

   $ kubectl run hello-world --replicas=5 --labels="run=load-balancer-example" --image=gcr.io/google-samples/node-hello:1.0  --port=8080

.. image:: kubeadm/kops9.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text
   
.. image:: kubeadm/kops10.PNG
   :width: 800px
   :height: 500px
   :alt: alternate text

Expose the deployment:

.. image:: kubeadm/kops11.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

.. code-block:: bash

   $ kubectl expose deployment hello-world --type=LoadBalancer --name=my-service

.. image:: kubeadm/kops12.PNG
   :width: 800px
   :height: 500px
   :alt: alternate text

.. image:: kubeadm/kops13.PNG
   :width: 800px
   :height: 100px
   :alt: alternate text
   
.. image:: kubeadm/kops15.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text
   
Access your service that is deployed in kubernetes with the external IP along with port specified within the service in a browser outside of the cluster.

.. image:: kubeadm/kops14.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text
   
Access your service that is deployed in kubernetes with the external IP along with port specified within the service in a browser outside of the cluster.


Deploying our Python Flask application:
========================================

Since we showcased the usage of local registry using minikube we’ll demonstrate this section using docker hub for the same python flask app. 

Sequence of steps to deploy flask application:

1)Build the Docker image the same way as shown in minikube section.

2)Tag the docker image with the username: 

Ex: $docker tag de52b31bd609 exeliq/flaskapp:latest

3)Once the image is tagged with your username, push the image to the DockerHub registry(it can be public/private), To push the image to docker hub, “create secret”.

.. code-block:: bash

   $ kubectl create secret docker-registry dockcred --docker-server=docker.io --docker-username=${username} --docker-              password=${password} -—docker-email=${email}
   
4)You can run the docker image in two ways. The first is to run the image with Kubectl as shown below. But this way, is constrained to allow only few parameters to be specified to the kubectl command. 

.. image:: kubeadm/kops16.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

5)CREATING A SERVICE THROUGH A YAML DESCRIPTOR

While the declarative approach of creating YAML script for the deployment would allow to declare more specifications such as replicasets, port etc., . Now create a yaml file for the kubernetes deployment to be created. The deployment yaml file consists of specifications in a declarative way that pulls the image from DockerHub, creates deployment and deploy the application. The Yaml looks something like this:

.. code-block:: bash

   #Flaskapp.yaml
   apiVersion: apps/v1beta1 
   kind: Deployment
   metadata:
     name: flask
   spec:
     replicas: 1
     template:
       metadata:
          labels:
            app: flask
     spec:
       containers:
        - name: flask
             image: exeliq/flask_py:latest
          ports:
        - containerPort: 5000

6)Now verify that the deployment is successful by viewing the deployment status of the application yaml  created as deployment by kubernetes. 
Initially, 

.. image:: kubeadm/kops17.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

After a while, Make sure the deployment is “Available”,

.. image:: kubeadm/kops18.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text
   
7)View the running Pods – the pods should be in a “running” state.

.. image:: kubeadm/kops19.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

You can also see the node where the pod has been scheduled  by using the $kubectl describe ${pods_name} command, which shows many other details of the pod

8)Edit Deployment(Ex:ReplicaSets):
Replicasets are used to define the number of pods that an application serves and is defined as per the load requirements in the application deployment YAML file. The requests are sent(loadbalanced) to the pods in a round robin fashion by kubernetes engine. In the above application YAML, we’ve declared replica to be 1, hence the deployment is created with 1 desired pod and the $kubectl get pods lists only one pod. But if you specify a different desired number of pods in the pod replica of  the deployment YAML , it would generate that number of pods. 

In order to EDIT and redeploy the existing YAML, we would specify with kubectl command as shown:

.. code-block:: bash

   $kubectl edit  ${deployment_name} 
              
.. image:: kubeadm/kops20.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text
   
The application deployment YAML is configured to set replicas to be 3. 
      
.. image:: kubeadm/kops21.PNG
   :width: 800px
   :height: 500px
   :alt: alternate text

Now $kubectl get pods will list all the 3 pods destined to serve the same application.

.. code-block:: bash

   $ kubectl get	pods		
   
.. image:: kubeadm/kops22.PNG
   :width: 800px
   :height: 100px
   :alt: alternate text

The scaling of pods can also be done on the ReplicationController like this:

.. code-block:: bash

   $ kubectl scale rc kubia --replicas=3

9)Now inorder to apply this configuration:

.. code-block:: bash

   $ kubectl rollout status ${deployment_name}

.. image:: kubeadm/kops23.PNG
   :width: 800px
   :height: 50px
   :alt: alternate text

10)Exposing Deployments:

The first method of expose an appliacation  externally is by creating a service and setting its type to NodePort. By creating a NodePort service, you make Kubernetes reserve a port on all its nodes (the same port number is used across all of them) and forward incoming connections to the pods that are part of the service.This is similar to a regular service (their actual type is ClusterIP), but a NodePort service can be accessed not only through the service’s internal cluster IP, but also through any node’s IP and the reserved node port.

11)Now,  Inorder to make the application available out side of the Cluster, we would expose the deployment with :

.. code-block:: bash

    $ kubectl expose deployment flask –type=LoadBalancer –port=8080
    
In here, Note that the deployment is exposed as a service of type “LoadBalancer”, this deployment when exposed as a service of type creates a LoadBlancer that serves the incoming requests and the IP of the loadbalancer is shown as ExternalIP of the “flask” service as shown below:

.. image:: kubeadm/kops24.PNG
   :width: 800px
   :height: 100px
   :alt: alternate text

12)Kubernetes clusters running on cloud providers usually support the automatic provision of a load balancer from the cloud infrastructure.You can see the LoadBalancer created in the Underlying Cloud Provider premise(AWS) as soon as the deployment is exposed as a service of type LoadBlancer.

.. image:: kubeadm/kops25.PNG
   :width: 800px
   :height: 400px
   :alt: alternate text

13)Accessing microservice:

Access your service that is deployed in kubernetes KOPS cluster with the external IP(LB IP) along with port specified within the service in a browser outside of the cluster. Further “Service Endpoints” for the service are assigned to be discoverable to other services that are intended to use this service. An Endpoints resource is a list of IP addresses and ports exposing a service. 
                                            
.. code-block:: bash

   $ kubectl get endpoints flask
   
14)To delete KOPS installation of Kubernetes Cluster :

.. code-block:: bash

   $ kops delete cluster --name=$(kube-cluster-name) --yes
