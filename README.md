Symfony Demo Application
========================

The "Symfony Demo Application" is a reference application created to show how
to develop applications following the [Symfony Best Practices][1].

You can also learn about these practices in [the official Symfony Book][5].

Requirements
------------

  * PHP 8.2.0 or higher;
  * PDO-SQLite PHP extension enabled;
  * and the [usual Symfony application requirements][2].

Installation
------------

There are 3 different ways of installing this project depending on your needs:

**Option 1.** [Download Symfony CLI][4] and use the `symfony` binary installed
on your computer to run this command:

```bash
symfony new --demo my_project
```

**Option 2.** [Download Composer][6] and use the `composer` binary installed
on your computer to run these commands:

```bash
# you can create a new project based on the Symfony Demo project...
composer create-project symfony/symfony-demo my_project

# ...or you can clone the code repository and install its dependencies
git clone https://github.com/symfony/demo.git my_project
cd my_project/
composer install
```

**Option 3.** Click the following button to deploy this project on Platform.sh,
the official Symfony PaaS, so you can try it without installing anything locally:

<p align="center">
<a href="https://console.platform.sh/projects/create-project?template=https://raw.githubusercontent.com/symfonycorp/platformsh-symfony-template-metadata/main/symfony-demo.template.yaml&utm_content=symfonycorp&utm_source=github&utm_medium=button&utm_campaign=deploy_on_platform"><img src="https://platform.sh/images/deploy/lg-blue.svg" alt="Deploy on Platform.sh" width="180px" /></a>
</p>

Usage
-----

There's no need to configure anything before running the application. There are
2 different ways of running this application depending on your needs:

**Option 1.** [Download Symfony CLI][4] and run this command:

```bash
cd my_project/
symfony serve
```

Then access the application in your browser at the given URL (<https://localhost:8000> by default).

**Option 2.** Use a web server like Nginx or Apache to run the application
(read the documentation about [configuring a web server for Symfony][3]).

On your local machine, you can run this command to use the built-in PHP web server:

```bash
cd my_project/
php -S localhost:8000 -t public/
```

Tests
-----

Execute this command to run tests:

```bash
cd my_project/
./bin/phpunit
```

[1]: https://symfony.com/doc/current/best_practices.html
[2]: https://symfony.com/doc/current/setup.html#technical-requirements
[3]: https://symfony.com/doc/current/setup/web_server_configuration.html
[4]: https://symfony.com/download
[5]: https://symfony.com/book
[6]: https://getcomposer.org/


## Containerize Application
**Important Information:** This is implementation is done assuming the database integrated is MySQL database

### Docker
1. Build docker image:  
    ```
    docker build . -t lingoda:v1
    ```

2. Test container by starting it:
    ```
    docker run -p 8000:8000 lingoda:v1
    ```
  
3. Re-tag image:
    ```
    docker tag lingoda:v1 forevertga/lingoda:v1
    ```

3. Push image to Iamge registry (Dockerhub):
  `Note:` Ensure to first create the repository at the image registry e.g., on Dockerhub 
    ```
    docker push forevertga/lingoda:v1
    ```

### Kubernetes
# Task 1 
1. Create a secret to allow access to the image resgistry from the k8s cluster:
    ```
    export DOCKER_REGISTRY_SERVER=https://index.docker.io/v1/
    export DOCKER_USER=myregistryname
    export DOCKER_PASSWORD=myregistrytoken
    export DOCKER_EMAIL=YOUR_EMAIL

    kubectl -n <namespace> create secret docker-registry regcred\
    --docker-server=$DOCKER_REGISTRY_SERVER\
    --docker-username=$DOCKER_USER\
    --docker-password=$DOCKER_PASSWORD\
    --docker-email=$DOCKER_EMAIL

    or 

    kubectl create secret docker-registry regcred\
    --docker-server=$DOCKER_REGISTRY_SERVER\
    --docker-username=$DOCKER_USER\
    --docker-password=$DOCKER_PASSWORD\
    --docker-email=$DOCKER_EMAIL
    ```
    `Note:` the above creates a Secret in the specified namespace. This Secret should be created in the same namespace where the apps will be deployed.


2. Deploy the app using the manifest file:
    ```
    kubectl apply -f kubernetes/deployment.yaml
    ```

3. Expose the app to make it accessible and functional:
    ```
    kubectl apply -f kubernetes/service.yaml
    ```

    When running locally you will have to port forward to access it via localhost -->  http://localhost:8001
    ```
    kubectl port-forward pod/lingoda-6c9568565b-gq85d 8001:8000
    ```

# Task 2
4. This part needs migration files to be added and since the demo app does not have migration files, only uncomment the initContainer portion of the code to run migration, the task 2 works perfectly here when uncommented provided the migration files are provided to be registered during deployment:
    ```
    initContainers:
       - name: migration
         image: forevertga/lingoda:v1
         command: ['sh', '-c', 'composer require symfony/runtime && php bin/console doctrine:migrations:migrate --no-interaction']
    ```

# Task 3
5. Create HPA - Scaling issues

    When the load increases, we want to also have the pods automatically increase to match demand. Therefore if there is increased load, horizontal scaling deploys more pods to handle the said load. Similarly, if the load decreases, and the number of pods is above the configured minimum, the hpa will work to ensure that it scales down.

    First, we need to install metric server (if it does not already exist) in the cluster. This exposes metrics through the Kubernetes API. To install Metric Server, download the manifest using:
    ```
    wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```
    Update the manifest file and include the arg: `--kubelet-insecure-tls`

    Then apply the manifest to the cluster:
    ```
    kubectl apply -f kubernetes/components.yaml
    ```

    Check the status of the metric-server using the following command:
    ```
    kubectl -n kube-system get deployments metrics-server
    ```

    Once the metric server is in a ready state, create the autoscaler
    ```
    kubectl apply -f kubernetes/hpa.yaml
    ```
    The above autoscaler maintains an average cpu utilization(across all pods) of 50%. Get the status of the autoscaler using:
    ```
    kubectl get hpa
    ```


    
    To test the above, we can simulate increased load by running the following in a separate terminal:
    ```
    kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://lingoda:8001; done"
    ```

    Watch the hpa in a different terminal:
    ```
    kubectl get hpa lingoda --watch
    ```


    # Todo - improvement 
    - Create a single Makefile to implement Task 1, 2 and 3
    - Use base64 encoded values for the database credentials for production
    - Create Ingress for external link when deployed on AWS, GCP or Azure
    - Improve on security 
    - Add monitoring with Prometheus/Grafana to manage the Scaling Issues
    - Improve on the demo app to include migrations files but this has to be project specific as the current demo app has a default database used and since I didn't focus on the application codebase, my focus what on the infrastructure