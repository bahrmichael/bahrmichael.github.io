---
layout: post
title: You can publish docker images without running docker - here's how
---

Over the last past days I've published and used a couple docker images without starting docker on my local machine.

*This guide is aimed at people who are already familiar with writing Dockerfiles.*

To get started you need a docker hub account and a github account. You don't need git or docker installed on your machine.

## The Github Repo

Head over to Github and create a repository. The name and description don't need to follow any conventions. I name my repos which are used for docker hub something like "docker-my-image-name".

Once the repository is set up, create a new file. You can do this through your browser. Name it Dockerfile and add the content you want. As an example we'll create one to deploy applications with the serverless framework.

```Dockerfile
FROM node:lts-slim

RUN npm install -g serverless
```

![Github Repo](https://dev-to-uploads.s3.amazonaws.com/i/oocbzm77l1ivjcq76kzv.png)

The Dockerfile is all you need. Feel free to add a readme or license.


## The Docker Hub Build

Now log in to Docker Hub and create a new repository there. Give it a nice name and an informative description.

![Docker Description](https://dev-to-uploads.s3.amazonaws.com/i/qdbhyanazz6is2312oa8.png)

Scroll down a bit and connect to the Github repository you created earlier. You might have to grant Docker Hub read access to your account.

![Docker Build](https://dev-to-uploads.s3.amazonaws.com/i/y6wfrch9b2e77vrck2lz.png)

Once you filled out all the settings, assuming you saved the Dockerfile in your repository click on "Create & Build". Otherwise click on "Create" and then go save the Dockerfile.

![Docker Progress](https://dev-to-uploads.s3.amazonaws.com/i/fkrdmf0somqpr6c9uxkx.png)

Depending on how busy the free platform is, your job might be queued for more than ten minutes. A little bit later your docker image will start building.

If nothing went wrong, your image is now available with `docker pull yourname/yourimage:latest`.