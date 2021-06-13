# Docker Getting Started Tutorial

This tutorial has been written with the intent of helping folks get up and running
with containers and is designed to work with Docker Desktop. While not going too much 
into depth, it covers the following topics:

- Running your first container
- Building containers
- Learning what containers are running and removing them
- Using volumes to persist data
- Using bind mounts to support development
- Using container networking to support multi-container applications
- Using Docker Compose to simplify the definition and sharing of applications
- Using image layer caching to speed up builds and reduce push/pull size
- Using multi-stage builds to separate build-time and runtime dependencies

# Part 7: Multi-Container apps  

## Index  
*   [Container networking](#Container-networking)  
*   [Start MySQL](#Start-MySQL)  
*   [Connect to MySQL](#Connect-to-MySQL)  
*   [Run your app with MySQL](#Run-your-app-with-MySQL)  
*   [Recap](#Recap)  

## Container networking  
Remember that containers, by default, run in isolation and don't know anything about other processes or containers on the same machine. So, how do we allow one container to talk to another? The answer is **networking**. Now, you don't have to be a network engineer. Simply remember this rule...  
> <span style="color: #147ac8"> Note </span>  
If two containers are on the same network, the can talk to each other. If they aren't, the can't

## Start MySQL  
There are two ways to put a container on a network:  
1.   Assign it at start.  
2.  Connect an existing container.  

For now, we will create the network first and attach the MySQL container at startup.  

1.  Create the network.  
```console
    docker network create todo-app  
```  
2.  Start a MySQL container and attach it to the network. We're also going to define a few environment variables that the database will use to initialize the database (see the "Environment Variables" section in the [MySQL Docker Hub listing](https://hub.docker.com/_/mysql/))  

```console
    docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:5.7
```

You'll also see we specified the ```--network-alias``` flag. We'll come back to that in just a moment. 

> <span style="color: #147ac8"> Tip </span>  
You'll notice we're using a volume named ```todo-mysql-data``` here and mounting it at ```var/lib/mysql```, which is where MySQL stores its data. However, we never ran a ```docker volume create``` command. Docker recognizes we want to use a named volume and creates one automatically for us.  

3.  To confirm we have the database up and running, connect to the database and verify it connects.  
```console
    docker exec -it <mysql-container-id> mysql -u root -p
```  
When the password prompt comes up, type in **secret**. In the MySQL shell, list the databases and verify you see the ```todos``` database.  

```mysql
    mysql> SHOW DATABASES;
```  
You should see output that looks like this:  

```mysql
 +--------------------+
 | Database           |
 +--------------------+
 | information_schema |
 | mysql              |
 | performance_schema |
 | sys                |
 | todos              |
 +--------------------+
 5 rows in set (0.00 sec)
```  
Now we have our ```todos``` database and it's ready for us to use!  

## Connect to MySQL  
Now that we know MySQL is up and running, let's use it! But, the question is... how? If we run another container on the same network, how do we find the container (remember each container has its own IP adress)?  

To figure it out, we're going to make use of the nicolaka/netshoot container, which ships with a lot of tools that are useful for troubleshooting or debugging networking issues.  

1.  Start a new container using the nicolaka/netshoot image. Make sure to connect it to the same network.  
```console
    docker run -it --network todo-app nicolaka/netshoot
```  

2.  Inside the container, we're going to use the ```dig``` command, which is a useful DNS tool. We're going to look up the IP adress for the hostname ```mysql```.  
```console
    dig mysql
```  
And you'll get an output like this...  
```console
 ; <<>> DiG 9.14.1 <<>> mysql
 ;; global options: +cmd
 ;; Got answer:
 ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32162
 ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

 ;; QUESTION SECTION:
 ;mysql.				IN	A

 ;; ANSWER SECTION:
 mysql.			600	IN	A	172.23.0.2

 ;; Query time: 0 msec
 ;; SERVER: 127.0.0.11#53(127.0.0.11)
 ;; WHEN: Tue Oct 01 23:47:24 UTC 2019
 ;; MSG SIZE  rcvd: 44
```  
In the "ANSWER SECTION", you will see an ```A``` record for ```mysql``` that resolves to ```172.23.0.2```(your IP adress will most likely have a different value). While ```mysql``` isn't normally a valid hostname, Docker was able to resolve it to the IP adress of the container that had that network alias (remember the ```--network-alias``` flag we used earlier?).  

What this means is... our app only simply needs to connect to a host named ```mysql``` and it'll talk to the database! It doesn't get much simpler than that.  


## Run your app with Mysql  
The todo app supports the setting of a few environment variables to specify MySQL connection settings. The are:  
*   ```MYSQL_HOST``` - the hostname for the running MySQL server  
*   ```MYSQL_USER``` - the username to use for the connection  
*   ```MYSQL_PASSWORD``` - the password to use for the connection  
*   ```MYSQL_DB``` - the database to use once connected  

> <span style="color:#147ac8">Setting connection Settings via Env Vars</span>  
While using env vars to set connection settings is generally ok for development, it is **HIGHLY DISCOURAGED** when running applications in production. Diogo Monica, the former lead of security at Docker, [wrote a fantastic blog post](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/) explining why.  
A more secure mechanism is to use the secret support provided by your container orchestration framework. In most cases, these secrets are mounted as files in the running container. You'll see many apps (including the MySQL image and the todo app) also support env vars with ```_FILE``` suffix to point to a file containing the variable.  
As an example, setting the ```MYSQL_PASSWORD_FILE``` var will cause the app to use the contents of the referenced file as the connection password. Docker doesn't do anything to support these env vars. Your app will need to know to look for the variable and get the file contents.  

With all of that explained, let's start our dev-ready container!  

1.  We'll specify each of the environment variables above, as well as connect the container to our app network.  

```console
    docker run -dp 3000:3000 \
    -w /app -v "$PWD:/app" \
    --network todo-app \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=todos \
    node:12-alpine \
    sh -c "yarn install && yarn run dev"
```  

2.  If we look at the logs for the container (```docker logs <container-id>```), we should see a message indicating it's using the mysql database.  

```console
 # Previous log messages omitted
 $ nodemon src/index.js
 [nodemon] 1.19.2
 [nodemon] to restart at any time, enter `rs`
 [nodemon] watching dir(s): *.*
 [nodemon] starting `node src/index.js`
 Connected to mysql db at host mysql
 Listening on port 3000
```  

3.  Open the app in your browser and add a few items to your todo list.  

4.  Connect to the mysql database and prove that the item are being written to the database. Remember, the password is **admin**.  
```console
    docker exec -it <mysql-container-id> mysql -p todos
```
And if you check the ```todos``` database, it has now the ```todo_items``` table, which stores all the items we've added, run the following:  
```mysql
     mysql> select * from todo_items;
    +--------------------------------------+--------------------+-----------+
    | id                                   | name               | completed |
    +--------------------------------------+--------------------+-----------+
    | c906ff08-60e6-44e6-8f49-ed56a0853e85 | Do amazing things! |         0 |
    | 2912a79e-8486-4bc3-a4c5-460793a575ab | Be awesome!        |         0 |
    +--------------------------------------+--------------------+-----------+
```

## Recap  
At this point, we have an application that now stores its data in an external database running in a separate container. We learned a little bit about container networking and saw how service discovery can be performed using DNS.  

But there's a good chance you are starting to feel a little overwhelmed with everything you need to do to start up this application. We have to create a network, start containers, specify all of the environment variables, expose ports, and more! That's a lot to remember and it's certainly making things harder to pass to someone else.  

In the next section, we'll talk about Docker Compose. With Docker Compose, we can share our applications stacks in a much easier way and let other spin them up with a single (and simple) command!  


# Part 8: Use Docker Compose  
 
## Index  
*   [Install Docker Compose](#Install-Docker-Compose)  
*   [Create the Compose file](#Create-the-Compose-file)  
*   [Define the app service](#Define-the-app-service)  
    *   [Define the MySQL service](#Define-the-MySQL-service)  
*   [Run the application stack](#Run-the-application-stack)  
*   [Tear it all down](#Tear-it-all-down)  

[Docker Compose](https://docs.docker.com/compose/) is a tool that was developed to help define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, can sping everything up or tear it all down.  

The *big* advantage of using Compose is you can define your application stack in a file, keep it at the root of your project repo (it's now version controlled), and easily enable someone else to contribute to your project. Someone would only need to clone your repo and start the compose app. In fact, you might see quite a few projects on Github/GitLab doing exactly this now.  

So, how do we get started? 

## Install Docker Compose  
If you installed Docker/Toolbox for either Windows or Mac, you already have Docker Compose!  
Play-with-Docker instances already have Docker Compose installed as well. If you are on a Linux machine, you will need to [install Docker Compose](https://docs.docker.com/compose/install/).  

After installation, you should be able to run the following and see version information.  

```console
    docker-compose version
```

## Create the Compose file  
1.  At the root of the app project, create a file named ```docker-compose.yml```.  

2.  In the compose file, we'll start off by defining the schema version. In most cases, it's best to use the latest supported version.  

```yml
version: "3.7"
```

3.  Next, we'll define the list of services (or containers) we want to run as part of our application.  

```yml
version: "3.7"

services:
```  
And now, we'll start migrating a service at a time into the compose file.  

## Define the app service  
Just to remember, this was the command we were using to define our app container.  

```console
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=admin \
  -e MYSQL_DB=todos \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
``` 

1.  First, let's define the service entry and the image for the container. We can pick any name for the service. The name will automatically become a network alias, which will be useful when defining our MySQL service.  

```yml
 version: "3.7"

 services:
   app:
     image: node:12-alpine
```  

2.  Tipically, you will see the ```command``` close to the ```image``` definition, although there is no requirement on ordering. So, let's go ahead and move that into our file.  
```yml
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
```  

3.  Let's migrate the ```-p 3000:3000``` part of the command by defining the ```ports``` for the service.  

```yml
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
```  

4.  Next, we'll migrate both the working directory (```-w /app```) and the volume mapping (```-v "$(pwd):/app"```) by using the ```working_dir``` and ```volumes``` definitions.  

```yml
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
     working_dir: /app
     volumes:
       - ./:/app
```  

5.  Finally, we need to migrate the environment variable definitions using the ```environment``` key.  

```yml
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
     working_dir: /app
     volumes:
       - ./:/app
     environment:
       MYSQL_HOST: mysql
       MYSQL_USER: root
       MYSQL_PASSWORD: admin
       MYSQL_DB: todos
```  

## Define the MySQL service  
Now, it's time to define the MySQL service. The command that we used for that container was the following:  

```console
docker run -d \
--network todo-app --network-alias mysql \
-v todo-mysql-data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=admin \
-e MYSQL_DATABASE=todos \
mysql:5.7
```  
1.  We will first define the new service and name it ```mysql``` so it automatically gets the network alias.  
We'll go ahead and specify the image to use as well.  

```yml
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
```  

2.  Next, we'll define the volume mapping. When we ran the container with ```docker run```, the named volume was created automatically. Howeverm that doesn't happen when running with Compose. We need to define the volume in the top-level ```volumes:``` section and then specify the mountpoint in the service config. By simply providing only the volume name, the default options are used.  

```yml
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
     volumes:
       - todo-mysql-data:/var/lib/mysql

 volumes:
   todo-mysql-data:
```  

3.  Finally we only need to specify the environment variables.  

```yml
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
     volumes:
       - todo-mysql-data:/var/lib/mysql
     environment:
       MYSQL_ROOT_PASSWORD: admin
       MYSQL_DATABASE: todos

 volumes:
   todo-mysql-data:
```  

At this point, our complete ```docker-compose.yml``` should look like this:  

```yml
version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: admin
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```  

## Run the application stack  
Now that we have our ```docker-compose.yml``` file, we can start it up!  

1.  Make sure no other copies of the app/db are running first (```docker ps``` and ```docker rm -f <ids>```).  

2.  Start up the application stack using the ```docker-compose up``` command. We'll add the ```-d``` flag to run everything in the background.  

```console
docker-compose up -d
```  

When we run this, we should see output like this:  

```console
 Creating network "app_default" with the default driver
 Creating volume "app_todo-mysql-data" with default driver
 Creating app_app_1   ... done
 Creating app_mysql_1 ... done
```  
You'll notice that the volume was created as well as a network! By default, Docker Compose automatically creates a network specifically for the application stack (which is why we didn't define one in the compose file).  

3.  Let's look at the logs using ```docker-compose logs -f``` command. You'll see the logs from each of the services interleaved into a single stream. This is incredibly useful when you want to watch for timing-related issues. The ```-f``` flag "follows" the log, so will give you live output as it's generated.  

If you don't already, you'll see output that looks like this...  

```console
 mysql_1  | 2019-10-03T03:07:16.083639Z 0 [Note] mysqld: ready for connections.
 mysql_1  | Version: '5.7.27'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
 app_1    | Connected to mysql db at host mysql
 app_1    | Listening on port 3000
```  

The service name is displayed at the beginning of the line (often colored) to help distinguish messages. If you want to view the logs for a specific service, you can add the service name to the end of the logs command (for example, ```docker-compose logs -f app```).  

> <span style="color:#147ac8">Tip: Waiting for the DB before starting the app</span>  
When the app is starting up, it actually sits and waits for MySQL to be up and ready before trying to connect to it. Docker doesn't have any built-in support to wait for another container to be fully up, running, and ready before starting another container. For Node-based projects, you can use the [wait-port](https://github.com/dwmkerr/wait-port) dependency. Similar projects exist for other languages/frameworks.  

4.  At this point, you should be able to open your app and see it running. And hey! We're down to a single command!  

## Tear it all down  
When you're ready to tear it all down, simply run ```docker-compose down```. The containers will stop and the network will be removed.  

<blockquote style="color: #ff7f7a !important;background: #0C141E;">
    <strong>Warning</strong>
    <p>Removing Volumes</p>
    <p>By default, named volumes in your compose file are NOT removed when running ```docker-compose down```. If you want to remove the volumes, you will need to add the ```--volumes``` flag.</p>
</blockquote>  
Once torn down, you can switch to another project, run ```docker-compose up``` and be ready to contribute to that project! It really doesn't get much simpler than that!  

## Recap  
In this section, we learned about Docker Compose and how it helps us dramatically simplify the defining and sharing of multi-service applications. We created a Compose file by translating the commands we were using into the appropiate compose format.  

At this point, we're starting to wrap up the tutorial. However, there are a few best practices about image building we want to cover, as there is a big issue with the Dockerfile we've been using. So, let's take a look!  

