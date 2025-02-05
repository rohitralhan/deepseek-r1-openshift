
# DeepSeek on OpenShift with OpenWebUI and Ollama

<p align="center"><img src="https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/images/combined.png" /></p>

## Introduction

### DeepSeek

DeepSeek R1 is a cutting-edge large language model (LLM) for advanced AI applications. It provides high accuracy, efficiency, and scalability, making it an excellent choice for developers and enterprises looking to integrate AI into their workflows. With billions of parameters, DeepSeek R1 offers superior performance in natural language understanding and generation tasks.

### Ollama

Ollama is a powerful tool designed to manage and run AI models efficiently. It simplifies the deployment of large language models like DeepSeek R1 70B by handling the underlying execution and resource management.

### OpenWeb UI

OpenWeb UI provides an accessible, user-friendly interface for interacting with AI models running on Ollama. It allows users to send requests, analyze responses, and fine-tune model interactions without needing extensive command-line expertise.

Together, Ollama and OpenWeb UI enable seamless integration of DeepSeek R1 into OpenShift, making it easy for developers and non-technical users to harness AI capabilities in a scalable, containerized environment.

## Prerequisites

Before getting started, ensure you have:

-   Access to an OpenShift environment with cluster administrator privileges.
-   OpenShift CLI (oc) installed.
-   Sufficient computational resources (GPU recommended for optimal performance).
-   Basic knowledge of Kubernetes and OpenShift.
-   Choose a DeepSeek R1 model based on your hardware capabilities for this demo we used DeepSeek R1 70B
----------

## Step 1: Setting Up the OpenShift Environment

1.  Log in to your OpenShift cluster:  
    oc login --server=<OPENSHIFT_API_URL> --token=<YOUR_ACCESS_TOKEN>
2.  Create a new project:  
    oc new-project deepseek-ai
3.  Verify the project is set up correctly:  
    oc project deepseek-ai
    
----------

## Step 2: Deploying Ollama on Red Hat OpenShift
All the files are on [github](https://github.com/rohitralhan/deepseek-r1-openshift). You can directly run it by simply cloning/downloading the files from github and running the **`oc apply -k .`** from the root folder. To delete the installation run **`oc delete -k .`** from the root folder. Alternatively, you can follow along to create the resources step by step.

Ollama is required to run DeepSeek R1 70B. We’ll create a persistent volume claim, deployment, service, and route for it. The deployment will require these three files:

-   pvc.yaml for persistent to download and store the DeepSeek R1 70B (about 45G)
-   deployment.yaml for the DeepSeek R1 70B model deployment.
-   service.yaml to expose the deployment as a service within the cluster.
-   route.yaml to create an OpenShift route for external access.

---

### Namespace creation yaml

    kind: Namespace
    apiVersion: v1
    metadata:
      name: "deepseek-r1"
      labels:
        name: "deepseek-r1"

### PVC YAML for Ollama

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: deepseek-r1-pvc
      namespace: deepseek-r1
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 100Gi
      storageClassName: freenas-nfs-csi
      volumeMode: Filesystem

### Deployment YAML for Ollama

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: deepseek-r1-deploy
      namespace: deepseek-r1
    spec:
      selector:
        matchLabels:
          name: deepseek-r1-deploy
      template:
        metadata:
          labels:
            name: deepseek-r1-deploy
            app: ollama-server
        spec:
          containers:
          - name: deepseek-r1-deploy
            image: ollama/ollama:latest
            env:
              - name: OLLAMA_MODELS
                value: /mnt/models-pvc/deepseek-70B
              - name: HOME
                value: /mnt/models-pvc/
              - name: OLLAMA_MODEL_NAME
                value: "deepseek-r1:70b"
            command: ["ollama"]
            args: ["serve"]
            ports:
            - name: http
              containerPort: 11434
              protocol: TCP
            volumeMounts:
            - mountPath: /mnt/models-pvc/
              name: model-volume
          restartPolicy: Always
          volumes:
          - name: model-volume
            persistentVolumeClaim:
              claimName: deepseek-r1-pvc

### Service YAML for Ollama

    apiVersion: v1
    kind: Service
    metadata:
      name: deepseek-r1-svc
      namespace: deepseek-r1
    spec:
      type: ClusterIP
      selector:
        name: deepseek-r1-deploy
      ports:
      - port: 80
        name: http
        targetPort: 11434
        protocol: TCP

### Route YAML for External Access

    kind: Route
    apiVersion: route.openshift.io/v1
    metadata:
      name: deepseek-r1-route
      namespace: deepseek-r1
    spec:
      to:
        kind: Service
        name: deepseek-r1-svc
      tls: null
      port:
        targetPort: http

  

You can either create and apply the YAML files using the command below or run above yaml from the `Import section` of the Red Hat OpenShift web console:

oc apply -f [namespace.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/namespace.yaml)
oc apply -f [pvc.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/ollama/pvc.yaml)
oc apply -f [deployment.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/ollama/deployment.yaml)
oc apply -f [service.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/ollama/service.yaml)
oc apply -f [route.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/ollama/route.yaml)

Confirm if the ollama pod is running:
oc get pods -n deepseek-r1

----------

## Step 3: Downloading and Running DeepSeek R1 70B

After starting Ollama, you have to download and run the DeepSeek R1 70B model. Running the command for `ollama run deepseek-r1:70b` will automatically trigger the model download if it isn't already available, and then run it.

#### Running DeepSeek R1 on Ollama:  

 1. Run this command `oc exec -it $(oc get pods -l app=ollama-server -o jsonpath="{.items[0].metadata.name}") -- /bin/sh -c "ollama run deepseek-r1:70b"`
 2. Once the model is downloaded it will run and present a prompt `>>>` type `/bye` to exit. This confirms that the model is up and running

![](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfwzZ7ov39vPMClZ0kkZXXd811gUs26yWfPdkceAAviwaUjLa1JRE2U7pJtLdY8XmTHDdqguOHhPHQRn9x7pk_1XKapQCI4HSRZLeUjfIBvkNxBaIsvz6v_S0XzWyGfNKzFYCLpKA?key=4hgiTo2KqihXQD2lOgQsMMyb)
<p align=center>Downloading DeepSeek-R1 </p>

 3. You can also run the following command to check that the model is up and running `oc exec -it $(oc get pods -l app=ollama-server -o jsonpath="{.items[0].metadata.name}") -- /bin/sh -c "ollama list"` you should see the output similar to the one below:
 
	``` 
	NAME  					ID  			SIZE  		MODIFIED
	deepseek-r1:1.5b  		a42b25d8c10a  	1.1 GB  	23 minutes ago 
	```

## Step 4: Deploying OpenWeb UI

OpenWeb UI provides an interface to interact with the model. We’ll create a persistent volume claim, deployment, service, and route for it. The deployment will require these three files:

-   pvc.yaml for persistent to download and store the DeepSeek R1 70B
-   deployment.yaml for the DeepSeek R1 70B model deployment.
-   service.yaml to expose the deployment as a service within the cluster.
-   route.yaml to create an OpenShift route for external access
    

### PVC YAML for OpenWeb UI
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openwebui-data
  namespace: deepseek-r1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```
  

### Deployment YAML for OpenWeb UI
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
  namespace: deepseek-r1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
      - name: open-webui
        image: ghcr.io/open-webui/open-webui:main
        ports:
        - name: openweb-http
          containerPort: 8080
          protocol: TCP
        env:
        - name: OLLAMA_BASE_URL
          value: "http://deepseek-r1-svc.deepseek-r1.svc.cluster.local"
        - name: WEBUI_SECRET_KEY
          value: "Secretkey"            
        volumeMounts:
        - name: webui-data
          mountPath: /app/backend/data
      volumes:
      - name: webui-data
        persistentVolumeClaim:
          claimName: openwebui-data
      restartPolicy: Always
```
### Service YAML for OpenWeb UI
```
apiVersion: v1
kind: Service
metadata:
  name: open-webui
  namespace: deepseek-r1
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
    name: http
  selector:
    app: open-webui
```
### Route YAML for External Access
```
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: open-webui
  namespace: deepseek-r1
spec:
  to:
    kind: Service
    name: open-webui
  tls: null
  port:
    targetPort: http
```
You can either create and apply the YAML files using the command below or run above yaml from the `Import section` of the Red Hat OpenShift web console:

oc apply -f [pvc.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/openweb-ui/pvc.yaml)
oc apply -f [deployment.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/openweb-ui/deployment.yaml)
oc apply -f [service.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/openweb-ui/service.yaml)
oc apply -f [route.yaml](https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/openweb-ui/route.yaml)

Check if the openweb-ui pod is running:
oc get pods -n deepseek-r1
 
----------

## Step 5: Accessing OpenWeb UI

1.  Retrieve the OpenWeb UI via the route URL:  
    **`echo "https://$(oc get route openweb-ui -o jsonpath='{.spec.host}')"`**
2.  Open the retrieved URL in your browser to access the WebUI, where you can interact with your deployed models through a graphical interface.    
3.  When accessing the WebUI for the first time, sign up with any credentials. The first user to register will receive administrator privileges, allowing full control over user management and settings.
4.  Select the DeepSeek-R1 model from the top left and start using the model via OpenWeb UI!

<p align="center"><img src="https://raw.githubusercontent.com/rohitralhan/deepseek-r1-openshift/refs/heads/main/images/openwebui-out.gif" /></p>

----------

## Conclusion

By following this guide, you have successfully:

-   Deployed Ollama on Red Hat OpenShift.    
-   Downloaded and started DeepSeek R1 70B.    
-   Set up OpenWeb UI for interaction with the model.    
-   Exposed the services using OpenShift Routes.    
-   Interact with the model using OpenWeb UI.    

This setup provides a scalable, accessible AI solution within an OpenShift environment. Enjoy!

----------

## Additional Resources

-   [DeepSeek Official Documentation](https://deepseek.com)
-   [OpenShift Documentation](https://docs.openshift.com)    
-   [Ollama GitHub](https://github.com/ollama)    
-   [OpenWeb UI Project](https://github.com/openwebui/openwebui)
----------
