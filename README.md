# Package a project
In this short article, I will show you how to package a java spring boot application together with a MySQL DB with docker.

**QUICK NOTE** : when I talk about the `application.properties` file, please view my other example of how secure a spring app. There this file has content related to connecting to a MySQL server. In this example, I set up a MySQL server, but my app doesn't connect to it just to keep it simple.

## Quick recap

Terminology:

-  *Container* - Like a virtual machine but smarter. i.e it can be used like a standalone server but is small and you can run multiple on your laptop. A container should encapsulate **one** service, like MySQL, Nginx, spring etc.
- *Docker* - A multi-tool for managing containers.
- *Dockerfile* - A config file that spec out a container. It contains information on what the container should do.
- *Docker image* - With a dockerfile we can build an image. It is a bit like compiling the dockerfile to an executable. We can later run this image and it will create a container for us.

Some things to note. An image does not persist in memory since it will not be a running instance. A container can store stuff between sessions. However, this is usually a bad practice. We would like the container to behave like **static code**. If we need to flush the instance and start another container from the same original image, it should more or less pick up where the old container left. This is one of the best things about containers.


*docker-compose* is like a sub-tool inside of docker, it orchestrates everything for us in a really easy and nice way. It can build a whole system(multiple interconnected containers) from one simple yml file.

You will not store an image inside your git repo. We can publish images to hubs. docker etc or something similar. But we want to keep it simple. Often when we look at different projects on GitHub we find that they either contain a Dockerfile or a docker-compose.yml file.

## Your goal
Your goal with using docker should be to make it easier for other developers to use your code. Be it develop upon it or deploy it. In this instance - as an assignment - you should make it as easy as possible for us TA:s to start, reset and test your app. 

The only commands we will run on our side are:

```bash
docker build -t some-tag-name .
docker run -t some-tag-name 
```

This should start your application, setup a MySQL server and just work-out-of-the-box. We will not do anything else, the app should start from this.

Instead of doing it like this, I would recommend you use docker-compose. Then the only thing we TA:s would need to do is 

```bash
docker compose up
```

The same thing here. From this command, everything should work out of the box. We TA:s should not need to enter any other commands.


## How to do it
I will use **docker-compose** in this example.

I have provided a folder named `demo`, this is a root directory of a GitHub repo. In short, it contains a simple spring boot app(This example does not connect to a DB, but pretend that it does. Please see my other example on *"how to connect a react frontend to spring security"* to see a live example).

### The docker-compose file
So this is the file content of the docker-compose file. It contains the specs for setting up two containers, one running OpenJDK and one MySQL.

```yml
services:
  db:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3306:3306'
    volumes:
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql

  spring:
    depends_on:
      - db
    image: 'openjdk:latest'
    container_name: back-end-server
    ports:
      - "8080:8080"
    volumes:
      - ./SpringApp:/javaSource
    command: sh -c "cd /javaSource; ./mvnw spring-boot:run"
```

The `DB` service is just fantastic, it sets up a whole MySQL server without any hassle. The volumes attribute gives the container an entry SQL script, i.e this script runs on boot. So in this file, you can add your schemas and table structure, add test data etc. Just keep in mind that it needs to be in this format.

### The magic for running spring

So the magic with using docker in this example comes from specifying the image `openjdk: latest`. This image setups an environment for us with some basic tools needed to run the app. What is awesome is that this environment will be consistent on your machine as it will be on mine. Therefore, when I will run `docker compose up` I will not be missing any software and the app will start without problems. But this is of course dependent on you have included everything in your git repo correctly.

The important parts to note are these : 

```yml 
    ports:
      - "8080:8080"     <--- This exposes the container port 8080
    volumes:
      - ./SpringApp:/javaSource     <--- [1]
    command: sh -c "cd /javaSource; ./mvnw spring-boot:run" <---- [2]
```

[1]This adds the spring app source to this container

[2] This command first changes the pointer to our java source, and then it runs the maven script for building and running the app as if it were inside  IntelliJ.

This part `./mvnw spring-boot: run` can be switched out for whatever solution you have for running your app. If you want you can build the project first, export it as a jar, then you can switch it out for `sh -c "cd /javaSource; java -jar yourFile.jar"`. This should work. 

## What more can you do with `docker compose`?

If you are not serving the frontend from spring. Then you could add the following service to your docker compose file : 

```yml
  node:
    depends_on:
      - back-end
    image: 'node'
    container_name: front-end-server
    volumes:
      - ./frontend/react-app:/frontend
    ports:
      - "3000:3000"
    command: sh -c "cd /frontend ; npm run start"
```

But then you would need to have some settings inside the webserver allowing `CORS` or set up a proxy like Nginx so the browser doesn't complain.

## The one thing I have left out
As you can see in this example `docker compose` is quite flexible and not that hard to use to get an app up and running. But there is one thing I have not included in this example and that is how we can dynamically change the IP for which DB to use in the spring app. So the problem is that we have to manually go in and change the `application.properties` file to the correct values and then recompile the whole app and reboot the container. There are ways to fix this, 

> but I want to keep it simple (stupid) - KISS

So in the `application.properties` as the DB server, use your "docker host IP". Run this command to view it(on Linux)

```bash
> ip addr

--- cut ---

6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:df:4d:68:c8 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0

--- cut ---

```

so in my case, it is `172.17.0.1` this might be the standard IP for docker to use(IDK, haven't had the opportunity or time to look it up). If you have anything else than this, please mark this clearly in your README file and make sure that we TA:s can rebuild the project and change the `application.properties` if needed.

Here is an example on a `application.properties` file 

```
server.port=8000
spring.jpa.generate-ddl=true
spring.datasource.url=jdbc:mysql://172.17.0.1:3306/db
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.jpa.database=mysql
```

# That is about it
If something isn't clear, please get in touch ASAP and let us sort it out.

Sicne this is a git repo: Feel free to make pull requests if you see something that is missing.
