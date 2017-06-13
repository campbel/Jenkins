# Multi-Container App
This tutorial will guide you through building, testing & deploying a multi-container docker application with a Jenkins pipeline.
## Prerequisites
Before starting make sure you have a dockerhub account and know your login information.
## 1. Initial Setup
Start by forking the example-voting app project from my repositories https://github.com/campbel/example-voting-app and then cloning it locally.
## 2. Create a Jenkinsfile for the pipeline job
Create a Jenkinsfile at the root of the project. Inside should be three stages **build**, **test** & **push**.
```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                
            }
        }
        stage('Test') {
            steps  {
                
            }
        }
        stage('Push') {
            steps {
                
            }
        }
    }
}
```
## 3. Add the build step to the Jenkinsfile
To build a multi-container Docker application we will need to use docker-compose. Docker-compose is a command line tool most often used in conjunction with docker-comopse.yml files. The docker-compose file is a configuration for how the containers should be built and run and how they relate to one another. It also includes configuration for volumes and networks and when used in conjunction with Docker Swarm, secrets.

Start by manually running a build.
```
docker-compose build
```
This command will execute a build for any service with a "build" parameter specified. Notice how redis and postgresql are not built, instead these images will be pulled. By default docker-compose will use the docker-compose file named docker-compose.yml. A different file can be specified with the `-f` flag.

Now let's add the command to our build step in our Jenkinsfile
```
stage('Build') {
    steps {
        sh docker-compose build       
    }
}
```
## 4. Run the application
Before we move onto testing let's try running the application manually.
```
docker-compose up
```
This will boot up all the containers (pulling them as needed) and stream their combined logs to your terminal. You'll see output for the five containers, redis, db, vote, result & worker. Browse to http://localhost:8000 and http://localhost:8001 to interact with the application. Make a vote on the vote page and see the results appear on the result page.

Next open a new terminal window and execute `docker network ls`. Here you should see a network named `examplevotingapp_default`. This network is important because it is what allows the containers in the application to talk to eachother. To demonstrate this run the following container.

```
docker run -it --rm --net examplevotingapp_default alpine sh
```
You will now be at the command line of an alpine linux container that is attached to the same network. Now let's install some utilities to demonstrate communicating with other containers in the network.
```
apk update && apk add curl
```
Next run a few commands to communicate with another container in the network.
```
nslookup vote
ping vote
curl http://vote
```
These three commands will be communicating with the **vote** container as defined in the docker-compose.yml file. Inface the name vote comes from the key for that service definition in that file. If we were to rename the service to foo or bar we could then re-run the application and communicate with the container by running `curl http://foo`.

So now you should have a basic understanding of how our multi-container application is able to communicate across process. Next we'll use this to execute tests against our application.

## 5. Add the test step to the Jenkinsfile
In the previous step we showed how containers connected to the same docker network could communicate to eachother by their service name. This is pretty nifty for our application, but it can also be used by our tests. The strategy for testing will be to create a new container that contains just our applications tests, boot up the application and run the container inside the same network.

First take a look at the docker-compose.test.yml file. This docker-compose file will be "merged" with our default compose file to create an environment for testing. This "merging" can be demonstrated with the following command.
```
docker-compose -f docker-compose.yml -f docker-compose.test.yml config
```
Notice how the output of the command is a merging of the two docker-compose files. This is what happens for any docker-compose command that has two compose files passed to it. So when we run `docker-compose -f docker-compose.yml -f docker-compose.test.yml up` both the application and the test container will be started. Try running this now.

You'll notice that much like before the application started, but now you'll see a new entry from the `test` service showing the output from the test and a message that it has exited. However, this isn't particularly useful to us because we want to receive the exit code from the test service once it's finished executing. Luckily docker-compose has a solution for this with the `--exit-code-from` flag. What this flag will do is watch a specific service and propagate it's exit code back up and it will also shutdown the application which is a nice bonus. So the full test command is now:
```
docker-compose -f docker-compose.yml -f docker-compose.test.yml up --exit-code-from test
``` 
Try running this command now and verify the application comes up, the test container runs and it shuts down as the test container exits.

Next all we need to do is add this command to our Jenkinsfile for the **Test** stage.
```
stage('Test') {
    steps {
        sh docker-compose -f docker-compose.yml -f docker-compose.test.yml up --exit-code-from test     
    }
}
```
## 6. Add the push stage to the Jenkinsfile
Lucky for us this is much simpler than step 5, however, if you are going to move forward with actually pushing the images you'll need to do some cleanup. First thing you'll want to do is go through all of the docker-compose files in the project and switch the image names to be for your dockerhub account (change *campbel* to your dockerhub ID).

Once that is done, you'll be happy to hear that docker-compose contains a `push` command.
```
docker-compose push
```
This command will push any container with both an "image" property and a "build" property in the docker-compose file. This is nice because we don't want it trying to push redis or postgresql. Try running the command now and see the vote, worker and result images being pushed to dockerhub. Once that is complete, let's add the stage to our Jenkinsfile.

```
stage('Push') {
    steps {
        sh docker-compose push    
    }
}
```
## 7. Setup the Jenkins server
Start the Jenkins server that exists at the root of this repository, follow the instructions from the README as needed.

Once the Jenkins server is running and you've signed in, we'll need to login to our dockerhub account.
`docker exec -it <jenkins_container_id> docker login -u <dockerhub_username> -p <dockerhub_password>`
Get the jenkins_container_id by running `docker ps` and finding the Jenkins container. This command will execute a docker login from within the Jenkins container. Allowing us to push our container image to Dockerhub. You should see `Login Succeeded`.

## 8. Check-in your application & Build it
You should now be ready to check-in your multi-container docker application and trigger a Jenkins build.`

Once the code is checked-in and synced with Github head to http://localhost:8080/blue/pipelines and follow the instructions for setting up a Jenkins pipeline build with Github. It will have you create an access token with Github for Jenkins use.

After the Job as been created it should execute automatically.

## 9. Add the deploy stage to the Jenkinsfile
For extra credit let's use Docker Swarm to deploy the multi-container application. Docker Swarm is the built-in orchestration engine for Docker containers. It is super simple to setup and easy to configure given we've already been exposed to docker-compose. You'll notice up until now I haven't referenced the `docker-compose.production.yml` file. This file is used for deploying to a swarm. This file has two key differences from the previous docker-compose file. First at the top you'll notice this is version 3. Version 3 compose files are specifically used for Docker swarm. While docker-compose will run a version 3 file it will ignore some fields in the spec. The most important of those fields being the "deploy" field. This property is used for describing deployment specific settings for the containers. For more information on docker-compose version 3, visit the documentation https://docs.docker.com/compose/compose-file/.

To run this locally you'll first need to initialize your docker swarm.
```
docker swarm init
```
And you're done, it was that easy. You now have access to docker swarm commands such as `docker node ls` and `docker deploy`.

When deploying a multi-container application to a swarm, we use the built-in docker deploy command, no need for docker-compose. This command is pretty straight forward, we pass deploy a docker-compose v3 file and a name and it will create a stack for the application.
```
docker deploy --compose-file docker-compose.production.yml voteapp
```
You should see some output saying the services and network was created. To verify the stack is up run:
```
docker stack ls
```
and then...
```
docker service ls
```

Notice the replicas are set 1/1. This means we told docker to run one instance of our service and one instance of our service is running. If we wanted to scale up we would simply alter this value and then run the deploy again. Let's do that now. Modify the replias in the docker-compose.production.yml file to different values and re-run the deploy command. Running `docker service ls` again should show the values changed. Once we hook this up to our pipeline modifying the number of instances of a service we have running will be as simple as checking-in changes to the docker-compose.production.yml file.

Now let's add the deploy stage to the Jenkinsfile.
```
stage('Push') {
    steps {
        sh docker deploy --compose-file docker-compose.production.yml voteapp   
    }
}
```
# Conclusion
We've shown how it can be just as easy to create a multi-container build and deploy pipeline as it was to create a single-container pipeline. We've also shown how you can use docker swarm to deploy your application to an environment that provides multi-node and scaling capabilities. Before you end it's recommended that you turn off swarm mode with `docker swarm leave -f`
