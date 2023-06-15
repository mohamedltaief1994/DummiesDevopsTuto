In this tuto we will create a jenkins agent with some prerequist ( jdk, node, kubectl) and we will bind docker socket to it.

First you need to build the image 
```bash
docker build .  -t jenkins-agent
```

Second we have two option you can run the image directely with dokcer 
```bash
docker run --init -v /var/run/docker.sock:/var/run/docker.sock jenkins-agent -url $URL $SECRET $NAME
```

or you can deployed to kubernetes with 

```bash
kubectl aplly -f deploy.yaml
```
notice you have to change $URL, $NAME and $SECRET environment variable with your jenkins config.
