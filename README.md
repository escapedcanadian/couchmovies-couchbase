# couchmovies-couchbase
These instructions are for creating/modifying the Couchbase server image used for the Couchmovies demo.
Demo users should not need to follow these instructions as pre-built Docker images should already exist.
To run the demo, please clone the demo repo to your laptop and follow the instructions

```
git clone https://github.com/escapedcanadian/couchbase-demo 
```

**The remaining instructions in this file are only needed to modify/rebuild the Couchbase container for the demo.**


## Updating code
### Updating tweet feeder
The tweet feeder code is located in the ```feeder``` directory.
If you change this code, it needs to be rebuilt with maven

``` 
mvn compile package
```
Once this completes, you need to copy the jar file into the top directory, since the target directory is not preserved by ```.gitignore```.

```
cp target/tweet-feeder-1.0.jar ../tweet-feeder.jar
```

## Building a new couchmovies-couchbase image
### couchbase-demo container
***This section describes the reasons and process for building a demo container. You may be able to pull an existing one from [Docker hub](https://hub.docker.com/repository/docker/escapedcanadian/couchbase-demo).  If a suitable version exists, you may proceed with [building the demo image](#Pull-and-run-a-base-demo-docker-image)***

'Official' Couchbase Docker images on Docker Hub are designed to start as a clean node everytime they you use ```docker run``` This occurse becuase the Dockerfile used to define these images declares ```VOLUME /opt/couchbase/var```. The cluster configuration and any loaded datafiles are stored on this volume and are not included into the container image when it is saved using ```docker commit```.

The goal is to get a Docker conatiner, runing a single Couchbase server node as its own cluster, preloaded with data. Trying to get this to work in a single Docker container is problematic, because the existing containers primary run thread is the server, and we need the server running in order to populate it. We considered/prototyped the following options to achieve this ...
 
1. **Have all of the data loaded as part of the run script (e.g ```entrypoint.sh```) when the container is first run**

  This option takes considerable time for the user to run when they first run the container. Many of the Couchbase CLI commands return asynchronously (before the change may be completely effected) the scripts require a lot of ```sleep``` command. Many failure conditions can occur and this method is unreliable for mass distribution.
  
2. **Have a specific container whose role is solely to populate the data**
 
 This option has the benefit of making it easier to plug in a new 'stock' version of the Couchbase docker image, but has most of the problems of the previous solution. 
 
3. **Convert the anonymous volume in the 'stock' Couchbase docker file to a named volume, and share the volume as well as the container**
 
 By definition, changing the type of volume in the 'stock' image, means it is no longer the stock image.  And sharing the named volume adds additional complexity.
 
4. **Remove the ```VOLUME``` statement from the 'stock' Couchbase Dockerfiles and pushlish 'demo' containers in Docker Hub**
 
 This option is what was eventually chosen because it provided the simplest and most foolproof method for the end user to download and run.  Incidently, we did create an anonymous ```VOLUME``` statement in the new ```Dockerfile```, in order to create a temporary space (```/opt/demo/temp```) to load data zip files and build scripts that would not needed in the final demo image.



Start by determining which version of Couchbase Server you wish to run. Look in the [couchbase-demo repository](https://github.com/escapedcanadian/couchbase-demo) for a branch that matches the version of Couchbase you want to use.  This branch identifier will be used as the ```<tag>``` in the following commands.

If there is not a branch for the version you wish, create one, copying the ```Dockerfile``` and ```scripts``` directory from the [Couchbase Docker git repo](https://github.com/couchbase/docker/tree/master/enterprise/couchbase-server), for the verion you want. Currently, the only modification to ```Dockerfile``` is to replace the final line

```
VOLUME /opt/couchbase/var
```
with

```
VOLUME /opt/demo/temp
```
Then you must build your couchbase-demo version and push it to Docker Hub.



### Pull and run a base demo docker image

```
pull docker escapedcanadian/couchbase-demo:<tag>
docker run -d --name couchmovies_couchbase_build -p 8091-8096:8091-8096 -p 11210-11211:11210-11211 escapedcanadian/couchbase-demo:<tag>

```
### Run data population scripts
Open a shell in the docker container created above.

```
docker exec -it couchmovies_couchbase_build bash
```
Inside of the docker container shell, run the following ...

```
apt-get update && apt-get -y  install git unzip
git clone https://github.com/escapedcanadian/couchmovies-couchbase /opt/demo/temp/couchmovies
cd /opt/demo/temp/couchmovies/build
./populateData
```  
At this point, ***it is important to wait for the indicies to complete building***. Use the admin UI on ```localhost:8091```. The credentials are Administrator:password

### Copy the tweet feeder code
The tweet feeder reads tweets from the tweetsource bucket and writes them (at a prescribed rate) into the tweettarget bucket.  This allows the analytics demo to demonstrate 'real-time analytics' on changing data.

The following script will move reqiured scripts into the root directory of the docker container so that they can be easily run using the ```docker-compose exec``` command.

Inside of the docker shell openned above, run ...

```
cd /opt/demo/temp/couchmovies/feeder
./installForDocker
```

### Tag and push the image to the Docker repo
Make sure you exit from the container shell and/or run the following from shell on your laptop

```
docker commit couchmovies_couchbase_build escapedcanadian/couchmovies-couchbase:<tag>
docker stop couchmovies_couchbase_build
docker container rm couchmovies_couchbase_build
docker tag escapedcanadian/couchmovies-couchbase:<tag> escapedcanadian/couchmovies-couchbase:latest
docker push escapedcanadian/couchmovies-couchbase:<tag>
docker push escapedcanadian/couchmovies-couchbase:latest
```
