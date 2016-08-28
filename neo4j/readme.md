# Installing Neo4j with Docker

Here are a few notes about creating a Docker image of Neo4j with the APOC and spatial libraries. The purpose is not to create a step-by-step guide, but to illustrate the basic stages of creating and deploying a derived Neo4j Docker image. I should say, by the way, that I'm a complete beginner at Docker. So take things that I say with a grain of salt.

## The Problem

My goal is to deploy a version of Neo4j with the [APOC](https://github.com/neo4j-contrib/neo4j-apoc-procedures) (or Awesome procedures for Neo4j 3.0) and the [spatial library](https://github.com/neo4j-contrib/spatial) plugins. Both plugins support GIS functionality not yet present in Neo4j (though [some functions have already been integrated](http://gist.asciidoctor.org/?dropbox-14493611%2Fcypher_spatial.adoc)). For various reasons, it turned out to be difficult to use these plugins locally or without incurring higher charges using hosted services. So creating a Docker image with these plugins in place seemed a natural option. 

## What is Docker?

[Docker](https://www.docker.com/) is a technology for packing applications into containers. There are many advantages to using containers, including bundling application components together, restricting access to system resources for improved security, and making applications portable across systems. My reason for considering Docker in this instance is primarily to make it easy to set up Neo4J with the configuration our application requires. The idea would be to create a Docker image that anyone could deploy if they want to use our mapping application. If you're interested in learning more about Docker in general, I'd recommend Jeff Nickoloff's [Docker in Action](https://www.manning.com/books/docker-in-action) (Manning Press, 2016).

Docker is a command line program but there is also a platform for sharing Docker images called [Docker Hub](https://hub.docker.com/). The analogy is like git and Github. Lots of companies, [including Neo4j](https://hub.docker.com/_/neo4j/), already provide official Docker images on Docker Hub. However, none of the official images include either APOC or the spatial library. So what to do? Raising the question on the Neo4j Slack channel for Docker, Michael Hunger suggested, "you could also create a derived image which just pulls the plugins as part image." Seemed like an excellent suggestion apart from the fact that I had no idea how to create a derived image.

## Creating a Dockerfile

After spending some time on Docker Hub looking at other people's derived images, it appeared to be less complicated than I feared. Docker images are built using text files called `Dockerfile`s. As far as I can tell, there is no extension for these files. In any case, Docker Hub allows you to read these Dockerfiles, many of which turn out to be maintained on Github. To extend a base image, you just need to start the Docker file with a `FROM` command, making sure to specify the tag (or version) of the Docker image you want to serve as the starting point. You should also use the `MAINTAINER` keyword to indicate who has created the derived image.

```
FROM neo4j:3.0.4
MAINTAINER Clifford Anderson <anderson.clifford@gmail.com>
```

The rest of the Dockerfile specifies the additional steps you'd like to have happen before starting your application. These are essentially the commands that you'd run if you were installing these plugins on your own system. In my case, this basically means using CURL to download the JAR files that I need and `mv` to move them into the plugins directory of Neo4j. As you see, I used the `ENV` command to create two variables since the URLs to the JAR files are rather long.

```
ENV APOC_URI https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases/download/3.0.4.1/apoc-3.0.4.1-all.jar
ENV GIS_URI https://github.com/neo4j-contrib/m2/blob/master/releases/org/neo4j/neo4j-spatial/0.19-neo4j-3.0.3/neo4j-spatial-0.19-neo4j-3.0.3-server-plugin.jar?raw=true

RUN mv plugins /plugins \
    && ln --symbolic /plugins

RUN curl --fail --silent --show-error --location --output apoc-3.0.4.1-all.jar $APOC_URI \
    && mv apoc-3.0.4.1-all.jar /plugins

RUN curl --fail --silent --show-error --location --output neo4j-spatial-0.19-neo4j-3.0.3-server-plugin.jar  $GIS_URI \
    && mv neo4j-spatial-0.19-neo4j-3.0.3-server-plugin.jar /plugins
```

The last two steps are to expose the ports I need to run the application and then to start it up. Frankly, I'm not sure why I need to expose the ports in my derived image since they're already exposed in the base image, but, for whatever reason, declaring them again seemed to be required. Finally, I issue a `CMD` to start up Neo4j.

```
EXPOSE 7474 7473 7687

CMD ["neo4j"]
```

I tested this Dockerfile locally with [Docker for Mac](https://www.docker.com/products/docker) and [Kitematic](https://kitematic.com/)/the [Docker Toolbox](https://www.docker.com/products/docker-toolbox). Obviously, I didn't want to push this image to Docker Hub if it didn't work. But my local Docker image worked great and so I was ready to move the image to Docker Hub.

## Using "Automated Builds" on Docker Hub

My inclination when trying to share this image was to push it to Docker Hub. That's certainly one way of sharing images, but it's not the only way. You can also share your image through what's called an "Automated Build." Essentially, you do so by connecting Docker Hub with your Github (or Bitbucket) account and creating a repository with your Dockerfile in it. By pointing your Docker Hub repository to the location of your Dockerfile (using "Build Settings") on Github, you can trigger a build of your image on Docker Hub. As you can imagine, there are lots of bells and whistles available, but the basic process of pointing Docker Hub to a Dockerfile on Github and then triggering a build was surprising straightforward. There are [good instructions about how to proceed](https://docs.docker.com/docker-hub/builds/) at Docker Hub.

## Deploying to Digital Ocean

Finally, I wanted to test the image by deploying it to [Digital Ocean](https://www.digitalocean.com/). Digital Ocean makes it very quick and easy to create cloud servers called "Droplets." You can also provision a Droplet with Docker already installed, meaning that there's no set up cost on the server. After creating a Droplet with 2 GBs to make sure that Neo4j had sufficient memory to perform well, I logged into the Droplet console as root and issued a command to Docker to pull my new image.

```
docker run \
    --publish=7474:7474 --publish=7687:7687 \
    --volume=$HOME/neo4j/data:/data \
    andersoncliffb/neo4j-with-apoc-and-spatial:latest
```

This command is identical to [the command to pull the official Neo4j image](https://neo4j.com/developer/docker/), apart from substituting my repository on Docker Hub. Docker began to pull the image on my Droplet and, within a matter of minutes, I was up and running with a correctly configured Neo4j server in the cloud. 

## Concluding Thoughts

Creating derived images on Docker Hub provides a easy way to extend the official images for specific purposes. While I chose to make my image public, I could also see the value of creating a private repository that would also import data into Neo4j. If you want to try out my image, it's available at https://hub.docker.com/r/andersoncliffb/neo4j-with-apoc-and-spatial/.

Thanks to Michael Hunger and Stefan Armbruster for suggestions along the way. Of course, the mistakes (or misleading ideas), are all mine.
