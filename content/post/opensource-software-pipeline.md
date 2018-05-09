+++
author = "Nick Lanng"
categories = ["travis", "ci", "github", "docker", "dockerhub"]
date = 2017-09-10T21:16:00Z
description = ""
draft = false
image = "/images/2017/09/BLOG-COVER.png"
slug = "opensource-software-pipeline"
tags = ["travis", "ci", "github", "docker", "dockerhub"]
title = "Open-source Software Pipeline"

+++

The following is a demonstration of the build pipeline I use for my personal projects.


# Creating a repository
To start, I create a repository on [Github](https://github.com). This whole process supports a ton of different languages, but I'm going to write a simple console app in Go.


# Checkout repository to local file system
Go has a convenient command for checking out the repository and automatically putting it into the correct path in my `GOPATH`.

{{< highlight bash >}}
$ go get github.com/nicklanng/pipeline-blog

package github.com/nicklanng/pipeline-blog: no buildable Go source files in /Users/nicholas.lanng/work/go/src/github.com/nicklanng/pipeline-blog
{{< /highlight >}}


# Write some code
I create my `main.go` in the root of the repository, this app will just print a hello to the console.

{{< highlight go >}}
package main

import "fmt"

func main() {
        fmt.Println("Hello, blog readers!")
}
{{< /highlight >}}


# Set up CI system
Next, I will set up my build for this project. I like to use [Travis CI](https://travis-ci.org/) due to its great integration with Github and superb community.

I use the Travis CLI Ruby gem, as it provides handy functions and shortcuts. It requires at least Ruby version 1.9.3 (2.0.0 recommended).

##First time configuration only
{{< highlight bash >}}
$ ruby -v

ruby 2.4.1p111 (2017-03-22 revision 58053) [x86_64-darwin16]
{{< /highlight >}}

{{< highlight bash >}}
$ gem install travis -v 1.8.8 --no-rdoc --no-ri

Successfully installed travis-1.8.8
1 gem installed
{{< /highlight >}}

You need to login to Travis with your Github credentials. **The credentials aren't sent to Travis, but instead to Github and an authorization token is then sent on to Travis.**

{{< highlight bash >}}
$ travis login

We need your GitHub login to identify you.
This information will not be sent to Travis CI, only to api.github.com.
The password will not be displayed.

Try running with --github-token or --auto if you don't want to enter your password anyway.

Username: nicklanng
Password for nicklanng: ********************
Successfully logged in as nicklanng!
{{< /highlight >}}

##The only command you need to run each time

I navigate to the respository's local folder and run the following. A series of questions are asked of me.
{{< highlight bash >}}
$ travis init

Detected repository as nicklanng/pipeline-blog, is this correct? |yes|
Main programming language used: |Ruby| Go
.travis.yml file created!
nicklanng/pipeline-blog: enabled :)
{{< /highlight >}}

This command has now created the Travis CI configuration file in my repository and also told Travis CI to watch for changes and build on every push.

I give the `.travis.yml` file a quick check, in this case I want to target different version of Go.

{{< highlight yaml >}}
language: go
go:
- '1.9'
{{< /highlight >}}


# Place CI build status on Repo
Travis CI provides an image that will automatically update to show the current build status. I put this 'badge' into my `README.md` file, so that I can see the status of the build directly from the repository. It also acts as a link to the Travis CI page for the repository.

The Travis CI URL has the same slug as the Github repo, so I navigate to https://travis-ci.org/nicklanng/pipeline-blog.

I click on the badge that says '**build | unknown**', select **Markdown** in the second drop-down and copy and paste the supplied code into my `README.md`.

{{< highlight md >}}
# pipeline-blog
Repository used for a demonstration on my blog

[![Build Status](https://travis-ci.org/nicklanng/pipeline-blog.svg?branch=master)](https://travis-ci.org/nicklanng/pipeline-blog)
{{< /highlight >}}


# Push to Github and trigger a build
Its time to push all of this to Github and see my first build!

{{< highlight bash >}}
$ git add .
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   .travis.yml
	modified:   README.md
	new file:   main.go

$ git commit . -m "Added Travis CI integration"
[master 3b28bcf] Added Travis CI integration
 3 files changed, 13 insertions(+)
 create mode 100644 .travis.yml
 create mode 100644 main.go

$ git push
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (5/5), 645 bytes | 645.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0)
To https://github.com/nicklanng/pipeline-blog
   9024a72..3b28bcf  master -> master

{{< /highlight >}}

Go to the repository at https://github.com/nicklanng/pipeline-blog and we can see the badge is saying the build passed!

![](/content/images/2017/09/Screen-Shot-2017-09-10-at-20.40.36.png)

Click on the badge and we are taken to the Travis CI page, we can see that build was run against Go 1.9.

![](/content/images/2017/09/builds.png)


# Create a repository on Dockerhub
This step requires an account on [DockerHub](https://hub.docker.com).

**Dockerhub requires me to create an empty repository on the website first, before pushing to it from Travis.**

![](/content/images/2017/09/Screen-Shot-2017-09-10-at-21.15.42.png)


# Deploy a dev build to Dockerhub
I like to package my projects up in Docker containers, so next I will set up a docker build and push to Dockerhub on every commit. This image will be tagged with `dev` as it is the not stable enough to be a reliable release.



Create a `Dockerfile` in the project root.
{{< highlight docker >}}
FROM scratch

ADD ./bin/pipeline-blog-linux-amd64 /pipeline-blog

CMD ["/pipeline-blog"]
{{< /highlight >}}


Open up your `.travis.yml` and add the following.

{{< highlight yaml >}}
language: go

go:
  - '1.9'

sudo: required
services: docker

script:
  - GOARCH=amd64 GOOS=linux go build -o bin/pipeline-blog main.go
  - docker build -t $TRAVIS_REPO_SLUG .

after_success:
  - if [ "$TRAVIS_BRANCH" == "master" ] && [ "$TRAVIS_PULL_REQUEST" == "false" ]; then
    docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD";
    docker tag $TRAVIS_REPO_SLUG $TRAVIS_REPO_SLUG:dev;
    docker push $TRAVIS_REPO_SLUG:dev;
    fi

{{< /highlight >}}

There are a few environmental variables in this file.
```
$TRAVIS_BRANCH : the name of the branch that is being built
$TRAVIS_PULL_REQUEST : true if this build a testing a PR before merging
$TRAVIS_REPO_SLUG : the repo slug, 'nicklanng/pipeline-demo'

$DOCKER_USERNAME && DOCKER_PASSWORD : configured on the Travis website as explained in the next section
```

**I don't want to put my Dockerhub username and password in my repository, that's asking for trouble!** Travis provides the ability to create environmental variables for builds, and has the option to obscure them from the build output!

Now I check in my new file and changes and check for a build on Dockerhub!
{{< highlight bash >}}
$ git add .

$ git status
On branch master
Your branch is up-to-date with 'origin/master'.

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   .travis.yml
	new file:   Dockerfile

$ git commit . -m "Push to docker on successful dev build"
[master 1eb89a0] Push to docker on successful dev build
 2 files changed, 22 insertions(+), 2 deletions(-)
 create mode 100644 Dockerfile

$ git push
Counting objects: 4, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 682 bytes | 682.00 KiB/s, done.
Total 4 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/nicklanng/pipeline-blog
   3b28bcf..1eb89a0  master -> master
{{< /highlight >}}

Once the build has finished, we can pull down the Docker image and run it!
{{< highlight bash >}}
$ docker pull nicklanng/pipeline-blog:dev
dev: Pulling from nicklanng/pipeline-blog
7bfe62a404bb: Pull complete
Digest: sha256:a15aa2e88f3e9ad4adef9023a3bc095f840349dc1c131085e8af5f7d8fec6bde
Status: Downloaded newer image for nicklanng/pipeline-blog:dev

$ docker run nicklanng/pipeline-blog:dev
Hello, blog readers!
{{< /highlight >}}

Official documentation: https://docs.travis-ci.com/user/docker/


# Deploying a stable version

Github has a releases feature, that allows you to create a tag in the version history and release notes. We can use feature along with Travis to deploy tagged versions to Dockerhub.

Add the following to the end of the `.travis.yml` file.
{{< highlight yaml >}}
  - if [ $TRAVIS_TAG ]; then
    docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD";
    docker push $TRAVIS_REPO_SLUG;
    docker tag $TRAVIS_REPO_SLUG $TRAVIS_REPO_SLUG:$TRAVIS_TAG;
    docker push $TRAVIS_REPO_SLUG:$TRAVIS_TAG;
    fi
{{< /highlight >}}

When we create a tag, Travis has access to it through the`$TRAVIS_TAG` variable. In this case, we want to push with the tag `:latest` and the tag, eg `:0.1.0`.

Push this change to the repository.

{{< highlight bash >}}
$ git add .

$ git commit . -m "Push version tags and latest tag to Dockerhub on releases"
[master 8b787c0] Push version tags and latest tag to Dockerhub on releases
 1 file changed, 6 insertions(+)

$ git push
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 376 bytes | 376.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/nicklanng/pipeline-blog
   525fe82..8b787c0  master -> master
{{< /highlight >}}

Next, let's create a release in Github and check the tags in Dockerhub!
Go to https://github.com/nicklanng/pipeline-blog/releases and hit 'Create a new release'.

![](/content/images/2017/09/Screen-Shot-2017-09-10-at-22.02.35.png)

This actually creates a new branch, found not in the standard Branches section in Github, but in the Tags sections next to it. We can see the new branch building on Travis CI.

![](/content/images/2017/09/Screen-Shot-2017-09-10-at-22.03.18.png)

In Dockerhub, we can now see the `:latest`, `:dev` and `:0.1.0` tabs on the image's page!


I haven't spoken about testing or deployment of the software, but I'm hoping that with your current knowledge of your language of choice and whatever deployment mechanism you use - you might now have enough knowledge to get cracking.
