# How to Serve a Flask App with Amazon Lightsail Containers

To get started, you&#39;ll need an [AWS account](https://portal.aws.amazon.com/billing/signup) and must install [Docker](https://docs.docker.com/engine/install/), the [AWS Command Line Interface (CLI) tool](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and the [Amazon Lightsail plugin](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/amazon-lightsail-install-software) on your system. Follow the provided links if you don&#39;t have some of those.

## Create the application

1. Create a new project directory and switch to that directory.

   ```bash
   $ mkdir lightsail-containers-flask && cd lightsail-containers-flask
   ```

2. Create a new file, app.py, and paste the following code into the file. Save the file.

   ```python
   from flask import Flask
   app = Flask(__name__)

   @app.route("/")
   def hello\_world():
      return "Hello, World!"

   if __name__ == "__main__":
      app.run(host="0.0.0.0", port=5000)
   ```

   This minimal Flask application contains a single function hello_world that is triggered when the route "/" is requested. When run, this application binds to all IPs on the system ("0.0.0.0") and listens on port 5000 (this is the default Flask port).

   If you have Python installed, you can test this application locally:

   ```bash
   $ python app.py
   ```

   Navigate to http://localhost:5000 in your browser. You should see "Hello, World!"

   Note: This is a development configuration, in production you&#39;d serve this application using Gunicorn or some other app server.

3. Create a new file, requirements.txt. Edit the file and add the following. Save the file.
   ```
   flask===1.1.2
   ```
   requirements.txt files are used to specifying what Python packages are required by the application. For this minimal Flask application there is only one required package, Flask.

4. Create a new Dockerfile. Edit the file and add the following. Save the file.

   ```
   # Set base image (host OS)
   FROM python:3.8-alpine

   # By default, listen on port 5000
   EXPOSE 5000/tcp

   # Set the working directory in the container
   WORKDIR /app

   # Copy the dependencies file to the working directory
   COPY requirements.txt .

   # Install any dependencies
   RUN pip install -r requirements.txt

   # Copy the content of the local src directory to the working directory
   COPY app.py .

   # Specify the command to run on container start
   CMD ["python", "./app.py"]
   ```
   The Python alpine image ensures the resulting container is as compact and small as possible. The command to run when the container starts is the same as if run from the command line: python app.py

   Your project directory should contain the following files:
   ```bash
   $ tree
   .
   ├── app.py
   ├── Dockerfile
   └── requirements.txt

   0 directories, 3 files
   ```
## Build the container

   Build the container using Docker. Execute the following command from the same directory as the Dockerfile:
   ```bash
   $ docker build -t flask-container .
   ```
   This command builds a container using the Dockerfile in the current directory and tags the container "flask-container".

   Once the container build is done, test the Flask application locally by running the container:
   ```bash
   $ docker run -p 5000:5000 flask-container
   ```
   The Flask app will run in the container and will be exposed to your local system on port 5000. Browse to [http://localhost:5000](http://localhost:5000/) or use "curl" from the command line and you will see "Hello, World!".
   ```bash
   $ curl localhost:5000
   
   Hello, World!
   ```

## Create a container service

1. Create a Lightsail container service with the [create-container-service](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/create-container-service.html) command.
   
   ```bash
   $ aws lightsail create-container-service --service-name flask-service \ --power small --scale 1
   ```
   The power and scale parameters specify the capacity of the container service. For a minimal flask app, little capacity is required.

   The output of the create-container-service command indicates the state of the new service is "PENDING".
   ```
   {
       "containerService": {
           "containerServiceName": "flask-service",
           ...
           "state": "PENDING",
   ```
   
   Use the [get-container-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/get-container-services.html) command to monitor the state of the container as it is being created.
   
   ```bash
   $ aws lightsail get-container-services --service-name flask-service
   ```
   
   Wait until the container service state changes to "ACTIVE" before continuing to the next step.

2. Push the application container to Lightsail with the [push-container-image](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/push-container-image.html) comand.
   ```bash
   $ aws lightsail push-container-image --service-name flask-service -- \ label flask-container --image flask-container

   ...

   Refer to this image as ":flask-service.flask-container.X" in deployments.
   ```
   Note: the X in ":flask-service.flask-container.X" will be a numeric value. You will need this number in the next step.

## Deploy the container

1. Create a new file, containers.json. Edit the file and add the following. Replace the X in ":flask-service.flask-container.X" with the numeric value from the previous step. Save the file.
   ```
   {
       "flask": {
           "image": ":flask-service.flask-container.X",
           "ports": {
               "5000": "HTTP"
           }
       }
   }
   ```
   The containers.json file describes the settings of the containers that will be launched on the container service. In this instance, the containers.json file describes the flask container, the image it will use and the port it will expose.

2. Create a new file, public-endpoint.json. Edit the file and add the following. Save the file.
   ```
   {
       "containerName": "flask",
       "containerPort": 5000
   }
   ```
   The public-endpoint.json file describes the settings of the public endpoint for the container service. In this instance, the public-endpoint.json file indicates the flask container will expose port 5000. Public endpoint settings are only required for services that require public access.

   After creating containers.json and public-endpoint.json files, your project directory should look like this:

   ```bash
   $ tree
   .
   ├── app.py
   ├── containers.json
   ├── Dockerfile
   ├── public-endpoint.json
   └── requirements.txt

   0 directories, 5 files
   ```
3. Deploy the container to the container service with the AWS CLI using the [create-container-service-deployment](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/create-container-service-deployment.html) command.

   ```bash
   $ aws lightsail create-container-service-deployment \
   --service-name flask-service \
   --containers file://containers.json \
   --public-endpoint file://public-endpoint.json
   ```
   The output of the create-container-servicedeployment command indicates that the state of the container service is now "DEPLOYING".
   ```
   {
       "containerServices": [{
           "containerServiceName": "flask-service",
           ...
           "state": "DEPLOYING",
   ```
   Use the [get-container-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/get-container-services.html) command to monitor the state of the container until it changes to "RUNNING" before continuing to the next step.
   '''bash
   $ aws lightsail get-container-services --service-name flask-service
   '''
   The get-container-service command also returns the endpoint URL for container service.
   ```
   {
       "containerServices": [{
           "containerServiceName": "flask-service",
           ...
           "state": "RUNNING",
           ...
           "url": "https://flask-service...
   ```
   After the container service state changes to "RUNNING", navigate to this URL in your browser to verify your container service is running properly. Your browser output should show "Hello, World!" as before.

   Congratulations. You have successfully deployed a containerized Flask application using Amazon Lightsail containers.

## Cleanup

To cleanup and delete Lightsail resources, use the [delete-container-service](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/delete-container-service.html) command.
```bash
$ aws lightsail delete-container-service --service-name flask-service
```
The delete-container-service removes the container service, any associated container deployments, and container images.
