# Simple App
This tutorial will guide you through creating a simple dockerized application to run through a Jenkins pipeline build.
## Prerequisites
Before starting make sure you have a dockerhub account and know your login information.
## 1. Initial Setup
Start by creating a new repository in Github. Then add three files to the root of the repo.
```
  /Dockerfile
  /Jenkinsfile
  /main.go
```
## 2. Create a simple Go application.
Let's create a simple application that serves up static text at the root of a web server. In Go this simple...

**main.go**
```
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":80", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "I'm writing an application for my Jenkins pipeline!")
}
```

We can test the application if we have go installed by running `go run main.go`. If you don't have Go installed, no worries, you'll be able to test once we put the application in a container.

## 3. Create a Dockerfile for building our application
The Dockerfile will tell Docker how to build our image. We start by specifying the base image we want to work from, in this case golang and then the actions we want to perform to build the application and setup our environment.

**Dockerfile**
```
FROM golang:1.8.3-alpine3.6
ADD main.go /go/src/app/main.go
RUN go install app
CMD app
```
We can now test the application in Docker by building the container and then running it. 
1. `docker build -t <dockerhub_username>/myapp .`
2. `docker run -it --rm -p 80:80 <dockerhub_username>/myapp`
3. Visit localhost:80 in your browser and you should see your applications response.

## 4. Create a Jenkinsfile for building our application in a pipeline Job.
The Jenkinsfile will tell the pipeline Job how to build our application.

**Jenkinsfile**
```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh "docker build -t <dockerhub_username>/myapp ."
            }
        }
        stage('Push') {
            steps {
                sh "docker push <dockerhub_username>/myapp"
            }
        }
    }
}
```
The job will be seperated into two jobs, the **Build** job and the **Push** job. The build job will build the image from source and the push job will push the image to Dockerhub.

## 5. Setup the Jenkins server
Start the Jenkins server that exists at the root of this repository, follow the instructions from the README as needed.

Once the Jenkins server is running and you've signed in, we'll need to login to our dockerhub account.
`docker exec -it <jenkins_container_id> docker login -u <dockerhub_username> -p <dockerhub_password>`
Get the jenkins_container_id by running `docker ps` and finding the Jenkins container. This command will execute a docker login from within the Jenkins container. Allowing us to push our container image to Dockerhub. You should see `Login Succeeded`.

## 6. Check-in your application & Build it
You should now be ready to check-in your simple docker application and trigger a Jenkins build.`

Once that is done head to http://localhost:8080/blue/pipelines and follow the instructions for setting up a Jenkins pipeline build with Github. It will have you create an access token with Github for Jenkins use.

After the Job as been created it should execute automatically.

## 7. Add unit tests
Let's add a unit test to our go application. This can be done by adding a file called **main_test.go**
```
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHandler(t *testing.T) {
	recorder := httptest.NewRecorder()
	request := httptest.NewRequest("", "/", nil)

	handler(recorder, request)

	if recorder.Code != http.StatusOK {
		t.Error("expected status 200 OK")
	}
}

```
If you have go installed you can run the tests with `go test`, otherwise go to step 8 and we'll execute them in the container as part of the build.

## 8. Execute unit tests with the build
There are a couple options for unit testing when it comes to Docker. Both options have their advantages and it might take experimenting with both to see what works best for you. For this tutorial pick one of the options below.
### (i) Modify the Jenkinsfile
When we run the container we can pass an optional command to execute. We can use this to run our tests, for example: `docker run -it --rm <image_name> go test app`. Then we can add this as a new stage in our Jenkinsfile.

**Jenkinsfile**
```
.
.
        stage('Test') {
            steps {
                sh "docker run -it --rm <dockerhub_username>/myapp go test app"
            }
        }
.
.
```
Check these changes in and run the job again.
### (ii) Modify the Dockerfile
Alternatively we can add another RUN command to our Dockerfile that will execute the unit tests as part of a Docker build. The downside to this is we can't build our container without running our unit tests, so if they take a long time to execute this solution may not be ideal.

**Dockerfile**
```
FROM golang:1.8.3-alpine3.6
ADD main.go /go/src/app/main.go
ADD main_test.go /go/src/app/main_test.go
RUN go install app
RUN go test app
CMD app
```
If we choose this solution, we won't need to modify our Jenkinsfile.

# Conclusion
That's it, I hope you found this useful. Please provide feedback in the issues section or send a pull request to fix any errors you may have found.
