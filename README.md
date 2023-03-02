# K8s Scaling and Updating

1. Export your namespace

    export MY_NAMESPACE=sn-labs-$USERNAME

2. Build and push the image

    docker build -t us.icr.io/$MY_NAMESPACE/hello-world:1 . && docker push us.icr.io/$MY_NAMESPACE/hello-world:1


## Deploy the application to Kubernetes

1. Run your image as a Deployment

    kubectl apply -f deployment.yaml

2. List Pods until the status is “Running”

    kubectl get pods

3. In order to access the application, we have to expose it to the internet via a Kubernetes Service.

    kubectl expose deployment/hello-world

This creates a service of type ClusterIP. Cluster IPs are only accesible within the cluster. To make this externally accessible, we will create a proxy.

4. Create a proxy

    kubectl proxy

5. Ping the app

    curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy

## Scaling the application using a ReplicaSet

1. Use the scale command to scale up your Deployment. 

    kubectl scale deployment hello-world --replicas=3

2. Get Pods to ensure that there are now three Pods instead of just one. In addition, the status should eventually update to “Running” for all three.

    kubectl get pods

3. Ping your application multiple times to ensure that Kubernetes is load-balancing across the replicas

    for i in `seq 10`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy; done

You should see that the queries are going to different Pods because of the effect of load-balancing.

4. Similarly, you can use the scale command to scale down your Deployment.

    kubectl scale deployment hello-world --replicas=1

## Perform rolling updates

1. Upadate app.js, build and push this new version to Container Registry:

    docker build -t us.icr.io/$MY_NAMESPACE/hello-world:2 . && docker push us.icr.io/$MY_NAMESPACE/hello-world:2

2. List images in Container Registry to see all the different versions of this application that you have pushed so far.

    ibmcloud cr images

3. Update the deployment to use this version instead

    kubectl set image deployment/hello-world hello-world=us.icr.io/$MY_NAMESPACE/hello-world:2

4. Get a status of the rolling update

    kubectl rollout status deployment/hello-world

5. You can also get the Deployment with the wide option to see that the new tag is used for the image

    kubectl get deployments -o wide

6. Ping your app:

    curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy

7. It’s possible that a new version of an application contains a bug. In that case, Kubernetes can roll back the Deployment like this

    kubectl rollout undo deployment/hello-world

8. Get a status of the rolling update

    kubectl rollout status deployment/hello-world

9. Get the Deployment with the wide option to see that the old tag is used.

    kubectl get deployments -o wide

10. Ping your app:

    curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy


## Using a ConfigMap to store configuration

ConfigMaps and Secrets are used to store configuration information separate from the code so that nothing is hardcoded. It also lets the application pick up configuration changes without needing to be redeployed. To demonstrate this, we’ll store the application’s message in a ConfigMap so that the message can be updated simply by updating the ConfigMap.

1. Create a ConfigMap that contains a new message

    kubectl create configmap app-config --from-literal=MESSAGE="This message came from a ConfigMap!"

2. Modify app.js with 

    res.send(process.env.MESSAGE + '\n')

This change indicates that requests to the app will return the environment variable MESSAGE

3. Build and push a new image that contains your new application code.

    docker build -t us.icr.io/$MY_NAMESPACE/hello-world:3 . && docker push us.icr.io/$MY_NAMESPACE/hello-world:3

4. Apply the new Deployment configuration.

    kubectl apply -f deployment-configmap-env-var.yaml

5. Ping your application again to see if the message from the environment variable is returned.

    curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy

If you see the message, “This message came from a ConfigMap!”, then great job!

6. Because the configuration is separate from the code, the message can be changed without rebuilding the image. Using the following command, delete the old ConfigMap and create a new one with the same name but a different message

    kubectl delete configmap app-config && kubectl create configmap app-config --from-literal=MESSAGE="This message is different, and you didn't have to rebuild the image!"

7. Restart the Deployment so that the containers restart.

    kubectl rollout restart deployment hello-world

8. Ping your app:

    curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy

## Autoscale the hello-world application using Horizontal Pod Autoscaler

1. lease add the following section to the deployment.yaml file under the template.spec.containers section for increasing the CPU resource utilization

              name: http
        resources:
          limits:
            cpu: 50m
          requests:
            cpu: 20m

2. Apply the deployment

    kubectl apply -f deployment.yaml

3. Autoscale the hello-world deployment 

    kubectl autoscale deployment hello-world --cpu-percent=5 --min=1 --max=10

4. You can check the current status of the newly-made HorizontalPodAutoscaler

    kubectl get hpa hello-world

5. Please ensure that the kubernetes proxy is still running in the 2nd terminal. If it is not, please start it again by running:

    kubectl proxy

6. Open another new terminal and enter the below command to spam the app with multiple requests for increasing the load

    for i in `seq 100000`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy; done

7. Run the below command to observe the replicas increase in accordance with the autoscaling

    kubectl get hpa hello-world --watch

You will see an increase in the number of replicas which shows that your application has been autoscaled.

Stop this command by pressing CTRL + C.

8. Run the below command to observe the details of the horizontal pod autoscaler

    kubectl get hpa hello-world

You will notice that the number of replicas has increased now

9. Stop the proxy and the load generation commands running in the other 2 terminal by pressing CTRL + C

10. Delete the Deployment

    kubectl delete deployment hello-world

11. Delete the Service

    kubectl delete service hello-world
