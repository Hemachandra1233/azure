
# Springboot with MySQL Deployment on Kubernetes

This repository contains a sample Spring Boot application with a MySQL database, deployed to Kubernetes.



## Overview:
This repository contains the project code for a simple RESTful API. The objective is to design a system API that will handle user data and deploy it to the Kubernetes. The API provides CRUD functionality for an User resource.

We have an user table that has the following fields:

```
user ID: integer
user name: string
country: string
```
The requirements are:
* Provide CRUD (create, read, update, delete) operations
* Insert a new user record
* Get all user records
* Get an user record given a specific user ID
* Delete an user record given a specific user ID
* Apply response status codes for applicable operations
  * 200 for successful GET. For GET All operations, an empty result is still a success
  * 201 for successful POST
  * 204 for successful DELETE, PATCH, or PUT
  * 404 when resource is not found. Use this on GET by ID, PATCH/PUT and DELETE
  * 500 for server errors

## Technologies Used

* Java 8
* Spring Boot 2.2.6.RELEASE
* Apache Maven
* Docker
* Kubernetes

## Getting Started

### Dependencies

* Generate the Spring Starter Project on spring initializer.
* Make sure you have Spring Web, Spring Data JPA, and MySQL Driver Dependencies added in pom.xml.
* When you want to use MySQL database, you must define the connection attributes in the application.properties file.
* Add below properties in properties file.

  ```
    spring.jpa.hibernate.ddl-auto=update
    spring.datasource.url=jdbc:mysql://${MYSQL_HOST:localhost}:3306/db_example
    spring.datasource.username=springuser
    spring.datasource.password=ThePassword
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    spring.jpa.show-sql: true
  ```
Here, ```spring.jpa.hibernate.ddl-auto``` can be ```none```, ```update```, ```create```, or create-drop. See the Hibernate documentation for details.

* ```none```: The default for MySQL. No change is made to the database structure.

* ```update```: Hibernate changes the database according to the given entity structures.

* ```create```: Creates the database every time but does not drop it on close.

* ```create-drop```: Creates the database and drops it when SessionFactory closes.

### Run the Application Locally using the Command Line

To run the Spring Boot application:

* Open a terminal and navigate to the root directory of the project.
* Run ```./mvnw spring-boot:run``` (on Unix-like systems) or ```mvnw spring-boot:run ``` (on Windows) to start the application.
* The application will start up and listen on ```http://localhost:8080```.

## Run Application on Kubernetes Cluster
### Step One: Setting up MySQL Database
- Use the `springboot-k8s-mysql\src\main\resources\mysqldb-root-credentials.yml` file to create a secret for root user:

    ``` kubectl apply -f mysqldb-root-credentials.yml ```

- Use the `springboot-k8s-mysql\src\main\resources\mysqldb-credentials.yml` file to create a secret for a user:
 
    ``` kubectl apply -f mysqldb-credentials.yml ```
- To verify that the secrets are created , list the secrets you created in cluster with the `kubectl get secrets` command:

    ```
    $ kubectl get secrets
    NAME                               TYPE                 DATA   AGE
    db-credentials                     Opaque               2      9d
    db-root-credentials                Opaque               1      9d
    ...
    ```

- Use the `springboot-k8s-mysql\src\main\resources\mysql-configmap.yml` file to create a ConfigMap:
 
    ``` kubectl apply -f mysql-configmap.yml ```

- To verify that the configmap is created , list the ConfigMaps you created in cluster with the `kubectl get configmaps` command:

    ```
    $ kubectl get cm
    NAME                  DATA   AGE
    db-conf               2      9d
    ```

- Use the `springboot-k8s-mysql\src\main\resources\mysql-deployment.yml` file to create a Deployment:
 
    ``` kubectl apply -f mysql-deployment.yml ```
- To verify that the Deployment is created and pods are in ready state, list the deployments  you created in the cluster with the `kubectl get deployments` command(if you don't specify a `--namespace`, the `default` namespace will be used. The same below):

 ```console
    $ kubectl get deployments
    NAME          READY            UP-TO-DATE     AVAILABLE     AGE
    mysql          1/1                  1            1           1d
    ...
```
- To verify that the service is created and, list the services you created in cluster with the `kubectl get svc` command:
    ```
    $ kubectl get svc
    NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    mysql                     ClusterIP   None             <none>        3306/TCP         9d
    ```

- Check whether pods created by deployments are running by executing the pods. Use below commands :

    ``` kubectl get pods ```

   ``` kubectl exec -it <podName> /bin/bash ```

   ``` mysql -u username -p password ```

   ``` show databaes; ```

   This should show databases present.

Note: For mysql database, mysql-5.7 image is pulled from the docker hub.

### Step Two: Setting up SpringBoot Application
#### Dockerizing SpringBoot Application:
- Configure `application.properties` file inorder to connect to MySQL database running on kubernetes cluster.
- Build `-jar` file of your application to build Docker image by below command.

    `mvn clean package -DskipTests`

- The project includes a dockerfile to build an image.

- Below docker file is used to build a docker image of our spring-boot application.


    ```
    FROM adoptopenjdk/openjdk11:alpine-jre
    ADD target/springboot-k8s-mysql-0.0.1-SNAPSHOT.jar app.jar
    ENTRYPOINT ["java","-jar","app.jar"]
    ```
- Build the above docker file by navigating to the location where Dockerfile exists by using below command

    ``` docker build .```

#### Setting up Spring Boot on Kubernetes  

- Use the `springboot-k8s-mysql\src\main\resources\deployment.yml` file to create a Deployment and service configured in it:
    
    ``` kubectl apply -f deployment.yml ```

- To verify that the Deployment is created and pods are in ready state, list the deployments  you created in the cluster with the `kubectl get deployments` command(if you don't specify a `--namespace`, the `default` namespace will be used. The same below):

 ```console
    $ kubectl get deployments
    NAME                         READY            UP-TO-DATE     AVAILABLE     AGE
    springboot-k8s-mysql          3/3                  3            3           1d
    ...
```

- To verify that the application pods are created and running, list the pods you created in cluster with the `kubectl get pods` command:

    ```
    $ kubectl get pods
    NAME                                       READY   STATUS    RESTARTS       AGE
    springboot-k8s-mysql-64ccd68695-2wbvf      1/1     Running   8 (62m ago)    9d
    springboot-k8s-mysql-64ccd68695-59nbl      1/1     Running   7 (62m ago)    9d
    springboot-k8s-mysql-64ccd68695-t27kv      1/1     Running   9 (62m ago)    9d
    ....
    ```

- Use the `springboot-k8s-mysql\src\main\resources\ingress-app.yml` file to create a ingress API object that manages external access to the services in a cluster, typically HTTP.
 
    ``` kubectl apply -f ingress-app.yml ```

- To verify that the Ingress object is created, list the ingress  you created in the cluster with the `kubectl get ingress` command(if you don't specify a `--namespace`, the `default` namespace will be used. The same below):

 ```console
    $ kubectl get ingress
    NAME                     CLASS           HOSTS         ADDRESS     PORTS       AGE
    example-ingressl         nginx      hello-world.info   localhost    80         1d
    ...
```
- Check whether all services, pods, secrets and ConfigMaps are created by using below commands

    ```kubectl get services```

    ```kubectl get pods```

    ```kubectl get secrets```

    ```kubectl get configmap```

## Test Your Application:

- Use below command to get list of users:

    `curl -X GET http://hello-world.info/users`

- Use below command to add a user:

    `curl -X POST http://hello-world.info/addUser`

- Use below command to retrieve a user by ID:

    `curl -X GET http://hello-world.info/findUser/{id}`

- Use below command to delete a user by ID:

    `curl -X DELETE http://hello-world.info/deleteUser/{id}`

Note: All Kubernetes Manifest files are located in resources section of application folder.





