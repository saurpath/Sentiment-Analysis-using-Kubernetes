# Sentiment Analysis using Kubernetes
###### Author: Saurabh Pathak (spathak2@andrew.cmu.edu)

### Summary
The application takes one sentence as input and using Text Analysis, calculates the emotion of the sentence.
[Application overview](/Sentiment-Analysis/Images/Overview.gif)

### Steps Required:
- Building container images for each service of the Microservice application.
- Deploy the images to DockerHub.
- Deploy the images from DockerHub to Google container registry.
- Deploy the containers into a Kubernetes Managed Cluster, along with load-balancers and service.



## Detailed steps:

### 1. Environment setup
1. Clone application code from https://github.com/rinormaloku/k8s-mastery
2. Create a github repository "Sentiment-Analysis-using-Kubernetes". 
3. Install maven, npm, docker, java and python.
4. Setup docker environment
    ```ruby
    export DOCKER_USER_ID=saurpath
    docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD"
    ```

### 2. Build docker container images for each microservice and push them to DockerHub.

1. Frontend
    ```ruby
    cd Sentiment-Analysis/sa-frontend
    npm install
    npm run build
    docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-frontend .
    docker push $DOCKER_USER_ID/sentiment-analysis-frontend
    ```
2. WebApp
    ```ruby
    cd Sentiment-Analysis/sa-webapp
    mvn install
    docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-web-app .
    docker push $DOCKER_USER_ID/sentiment-analysis-web-app
    ```

3. Logic
    ```ruby
    cd Sentiment-Analysis/sa-logic
    docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-logic .
    docker push $DOCKER_USER_ID/sentiment-analysis-logic
    ```

### 3. Push images from Dockerhub to Google container registry.
1. Pull images from Dockerhub.
    ```ruby
    docker pull saurpath/sentiment-analysis-logic
    docker pull saurpath/sentiment-analysis-web-app
    docker pull saurpath/sentiment-analysis-frontend
    ```
2. Tag the images
    ```ruby
    docker tag saurpath/sentiment-analysis-frontend gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend:latest
    docker tag saurpath/sentiment-analysis-web-app gcr.io/extracredit-project1/saurpath/sentiment-analysis-web-app:latest
    docker tag saurpath/sentiment-analysis-logic gcr.io/extracredit-project1/saurpath/sentiment-analysis-logic:latest
    ```
3. Push the images to Google container registry.
    ```ruby
    docker push gcr.io/extracredit-project1/saurpath/sentiment-analysis-web-app
    docker push gcr.io/extracredit-project1/saurpath/sentiment-analysis-logic
    docker push gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend
    ```

### 4. Deploy the containers on Google Kubernetes Engine.
1. **Enable Kubernetes Engine API.**
2. **Navigate to Kubernetes Engine -> Clusters and Create a new cluster.**
    a. Choose GKE Standard
    b. Name: extra-credit
    c. Click Create.
3. **This would deploy a standard cluster of 3 pods.**
4. **Deploy the containers:**
    a. Frontend
    + Click Deploy
    + Choose the frontend image gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend
    + Continue
    + Set the application name as frontend.
    + Deploy.
    
    b. WebApp
    + Click Deploy
    + Choose the web-app image gcr.io/extracredit-project1/saurpath/sentiment-analysis-web-app
    + Continue
    + Set the application name as webapp.
    + Deploy.
    
    c. Logic
    + Click Deploy
    + Choose the logic image gcr.io/extracredit-project1/saurpath/sentiment-analysis-logic
    + Continue
    + Set the application name as logic.
    + Deploy.

5. **Edit the YAML files for port configuration:**
    a. Frontend: Add the service port under specs as
    ```ruby
    spec:
      containers:
        name: sentiment-analysis-frontend-sha256-1
        ports:
        - containerPort: 80
          protocol: TCP
    ```
    b. WebApp: Add the service ports and logic-service URL. [created in next step] 
    ```ruby
    spec:
      containers:
      - env:
        - name: SA_LOGIC_API_URL
          value: http://logic-service
        name: sentiment-analysis-web-app-sha256-1
        ports:
        - containerPort: 8080
          protocol: TCP
    ```
    c. Logic: Add the service port
    ```ruby
    spec:
      containers:
        name: sentiment-analysis-logic-sha256-1
        ports:
        - containerPort: 5000
          protocol: TCP
    ```

5. **Create load balancers for Frontend and WebApps:**
    a. Click on frontend pod -> Actions -> Expose.
    b. Set name as frontend-lb.
    c. Map the port 80 (public) to Target port (container) 80.
    d. Choose Service type as Load balancers and click Expose.
    e. Repeat for webapp pod with name as webapp-lb and target port as 8080.

6. **Expose the logic pod as a service**
    a. Click on logic pod -> Actions -> Expose.
    b. Set name as logic-service. This is where kube-dns will route the requests from the webapp to.
    c. Map the port 80 to Target port 5000.
    d. Choose Service type as Node Port and click Expose. 

7. **Route the requests from frontend to webapp pod.**
    a. Get the IP address of the webapp load-balancer from KE -> Services & Ingress.
    b. In our case, it is http://35.225.133.13:80.
    c. Edit the App.js file locally at Sentiment-Analysis/sa-frontend/src/
    ```ruby
     analyzeSentence() {
        fetch('http://35.225.133.13:80/sentiment', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({sentence: this.textField.getValue()})
        })
            .then(response => response.json())
            .then(data => this.setState(data));
    }
    ```
    d. Build the dockerfile again with the updated changes and tag it as minikube.
    ```ruby
    docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-frontend:minikube .
    ```
    e. Push the updated image to DockerHub.
    ```ruby
    docker push $DOCKER_USER_ID/sentiment-analysis-frontend:minikube
    ```
    f. Fetch the image from Docker Hub.
    ```ruby
    docker pull saurpath/sentiment-analysis-frontend:minikube
    ```
    g. Tag and push the image to Google Container Registry.
    ```ruby
    docker tag saurpath/sentiment-analysis-frontend:minikube gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend:minikube
    docker push gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend:minikube
    ```
    h. Copy the update image name from Google container registry.
    ```ruby
    gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend@sha256:ca69bd33e3b8b0f3d6c8d1e55cdf584b18c250ee520a029751e1169cced3c445
    ```
    i. Update the webapp pod to use the new image by editing the YAML file.
    ```ruby
    spec:
      containers:
      - image: gcr.io/extracredit-project1/saurpath/sentiment-analysis-frontend@sha256:ca69bd33e3b8b0f3d6c8d1e55cdf584b18c250ee520a029751e1169cced3c445
    ```

## That's it! 
Navigate to the IP address pointed by the frontend-lb to use the Application.

Frontend load-balancer IP address: http://http://34.135.254.171:80

![IP address screenshot](/Sentiment-Analysis/Images/Cluster-IP-address.png)

Application Output:

![Application screenshot](/Sentiment-Analysis/Images/Output.gif)


----------
### Container Images on Docker Hub.
1. Frontend: https://hub.docker.com/r/saurpath/sentiment-analysis-frontend
2. WebApp: https://hub.docker.com/r/saurpath/sentiment-analysis-web-app
3. Logic: https://hub.docker.com/r/saurpath/sentiment-analysis-logic
----------
