# Docker Image Security workshop

## 1. Challenge: Find the most vulnerability-free image

Demo OS-level vulnerabilities made possible by a vulnerability in a library installed on the operating system, and used by an application: [https://github.com/lirantal/goof-container-breaking-in](https://github.com/lirantal/goof-container-breaking-in)

In this challenge you should find and fix vulnerabilities in docker images that you test and build in a local environment, or from your CI.

Starting point is this goofy project called `docker-goof` which uses an old version of Node.js runtime as a docker image tag.

1. Clone the project: [https://github.com/snyk/docker-goof](https://github.com/snyk/docker-goof)
2. Inside it you will find a `Dockerfile` with the image to use
3. Find how many vulnerabilities (low, medium, and high) the image has. Are there any interesting vulnerabilities?

<details><summary>Hint</summary>
<br/>

We'll get started with scanning a docker image using the Snyk CLI which is free to use and scan.

### Install the Snyk CLI or use Docker Scan

You can use the open source Snyk CLI to scan the image.

> Snyk is now a part of Docker Desktop as [docker scan](https://docs.docker.com/engine/scan/).

You can install `Snyk` or use `docker scan`:

* [Install Snyk on Windows](https://support.snyk.io/hc/en-us/articles/360003812538#UUID-ac39f35d-8608-e949-613d-24333ced4d42)
* [Install Snyk on macOS](https://support.snyk.io/hc/en-us/articles/360003812538#UUID-876089c6-d195-a81e-4c7a-21354f788306)
* [Install Snyk on Linux](https://github.com/snyk/snyk/releases) and other self-contained executables for Windows and macOS.

OR just use `docker scan`.

See [Snyk CLI install instructions](https://support.snyk.io/hc/en-us/articles/360003812458-Getting-started-with-the-CLI) to get started for the CLI, or the [docker scan docs](https://docs.docker.com/engine/scan/).

### Scan a docker image for security vulnerabilities

To scan an image you'll need to pull the docker image to your host locally and then scan it:

```sh
$ docker pull <image>
$ snyk test --docker <image>
```

or 

```sh
$ docker pull <image>
$ docker scan <image>
```

<br/>
</details>

4. Did you find vulnerabilities in the Node.js runtime as well? give an example of one of them and which version of Node.js will fix it.
5. How would you fix these vulnerabilities?
6. Find which official Node.js image you could switch to in order to lower your vulnerabilities footprint without disrupting the product and development teams too much.

<details><summary>Hint</summary>
<br/>

If the Snyk CLI is provided with a Dockerfile it will give you a remediation advice so you can make a conscious decision of which image you could move to in order to lower the security vulnerabilities foot-print.

What happens if you provide the Snyk CLI with the Dockerfile as well?

```sh
snyk test --docker <image> --file=Dockerfile
```

or 

```sh
docker scan --file Dockerfile <image>
```


<br/>
</details>

<details><summary>Solution</summary>
<br/>

```sh
snyk test --docker node:10.4.0 --file=Dockerfile
```

or 

```sh
docker scan --file Dockerfile node:10.40.0
```

<br/>
</details>

Don't forget -
Lowest vulnerabilities & ease of upgrade wins the challenge!

### Registry workflow

Find and fix vulnerabilities in docker images in a Docker Hub registry or others.

Bonus challenge - how do you monitor your docker images on a container registry and do you have an actionable advice as to which image and tag you should change to in order to lower the footprint of your image's security vulnerabilities?

## 2. Challenge: Don't steal my (sort of) immutability!

Your team needs to update to the latest version of Node.js 10 LTS to get fixes and keep up to date with security vulnerabilities.

You do the obvious:

```sh
docker pull node:10
```

1. Can you think of some downsides to pulling the image this way?

### Use fixed tags for immutability

When you pulled the image you specified the image name and a tag. But what's the implications of using a tag like that?

> Each Docker image can have multiple tags, which are variants of the same images. The most common tag is _latest_, which represents the latest version of the image. Image tags are not immutable, and the author of the images can publish the same tag multiple times.
> This means that the base image for your Docker file might change between builds. This could result in inconsistent behavior because of changes made to the base image.

_source: [10 Docker Image Security Best Practices](https://snyk.io/blog/10-docker-image-security-best-practices/)_

The above best practices document stresses that we should be as specific as possible in our tags, and ideally we'd pull in the image by its SHA256 reference.

2. Pull in the image based on its SHA256

<details><summary>Hint 1</summary>
<br/>

If you pulled the `node:10` image, take a look at the output

<br/>
</details>

<details><summary>Hint2</summary>
<br/>

You're looking for the `Digest` key in the output of `docker pull`

<br/>
</details>

<details><summary>Solution: how to pull the image by its SHA256</summary>
<br/>
   
```
docker pull node@sha256:bdc6d102e926b70690ce0cc0b077d450b1b231524a69b874912a9b337c719e6e
```

<br/>
</details>

## 3. Challenge: Really bad practices by default

Reminder: if you're coming here from previous challenges, did you remember to turn Docker's Content Trust policy off? Right, you need to do that:

```
export DOCKER_CONTENT_TRUST=0
```

How do you help your devops and developer engineers to validate the proper configuration of a `Dockerfile` when they are building them? Aaaaaaaaa-utomation!

We'll visit a couple of static code analysis tools to help us find out issues in a `Dockerfile`, or what us in the JavaScript land like to call - linters. There are a couple I recommend, and you're welcome to try both of them:

1. [Dockle](https://github.com/goodwithtech/dockle) - Puts a focus on scanning the image through its layers.
2. [Hadolint](https://github.com/hadolint/hadolint) - Statically analyze the Dockerfile

For this challenge we'll work with the project in the `bad-defaults/` directory.
Use any of the linters above and test the Docker file/docker image.

Only after you found issues with either or both of the above linters should you continue to the issues below in order to fix them.

### Container is running as root

1. Build the bad defaults image

<details><summary>Hint</summary>
<br/>
   
```
docker build -t best-practices .
```

<br/>
</details>

2. Check if the container is running with the root user. If so, that's not good.

<details><summary>Solution</summary>
<br/>
   
```
docker run -it --rm best-practices:latest sh
```

Inside the container run:

```
whoami
```

<br/>
</details>

3. Fix the issue so that the container uses a non-privileged user

<details><summary>Solution</summary>
<br/>
Update the Dockerfile to make use of the built-in `node` user:
   
```
...
USER node
CMD node index.js
```

Now build the container and run again to check which is user is being used.

<br/>
</details>

## 4. Challenge: All your secrets belong to me!

Did the `Dockerfile` smell somewhat fishy to you in the previous challenge?
It was smelly of secrets and tokens!

I am specifically referring to this entry in the `Dockerfile`:

```
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc
```

1. First assignment, can you think about why this is bad?
   After all, we're building the image in an internal environment and only we pass that secret token at build time. No one else sees it.

<details><summary>Hint 1</summary>
<br/>

The `.npmrc` file contains sensitive information, such as a token which is used for read/write for private packages on a registry. If the container is compromised, users will be able to access it.

Can you think of a simple vulnerability in an application that will allow a malicious attacker to easily get to the `.npmrc` file?

You can build it yourself and try:

```
export NPM_TOKEN=<npm token>
docker build -t best-practices --build-arg NPM_TOKEN=$NPM_TOKEN .
```

Now login to the container and validate the value of `.npmrc`:

```
docker run -it --rm best-practices sh
```

<br/>
</details>

2. Can you think of some other ways to fix this issue of secrets leaking? Below are hints for 2 "solutions". They might prove like a good idea for you but we explain why they shouldn't be followed.

<details><summary>Bad practice 1</summary>
<br/>

You remember to remove the token, such as:

```
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc
RUN npm install
RUN rm .npmrc
```

Nice, but not really good.
Every `RUN` creates another layer, all of which are later inspect-able and leave a trace. This means that if the image itself is ever leaked or made public then sensitive data exists inside it in the form of the `.npmrc` file.

<br/>
</details>

<details><summary>Bad practice 2</summary>
<br/>

You understand the concept of Docker layers so you put all of this into one command to make sure there's no trace, something like this:

```
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && \
    npm install && \
    rm .npmrc
```

However, Docker has this thing called commits history which it uses to save metadata about the way the image was built and this is why you should never really use environment variables such as build arguments for sensitive storage such as passwords, API keys, tokens.

Read more about [Docker history here](https://docs.docker.com/engine/reference/commandline/history/).

<br/>
</details>

<details><summary>Hint: for proper solution</summary>
<br/>

What if you could create a Docker image without the `.npmrc` file in it?

<br/>
</details>

<details><summary>Solution</summary>
<br/>

Let's use multi-stage builds to fix it!

Update the `Dockerfile` so that the first image is used as a base to install all of our npm dependencies and build what is required. To do that, update the FROM instruction as follows:

```
FROM bitnami/node:latest AS build
```

As well as remove the `CMD` instruction which isn't needed.
Then have another section in the Dockerfile for the "production" image, which should use the app directory which is now ready for use from the previous build image.

Following is an example:

```
FROM bitnami/node:latest
RUN mkdir ~/project
COPY --from=build /app/~/project ~/project
WORKDIR ~/project
CMD node index.js
```

An example of a full multi-stage Node.js docker image build Dockerfile:

```
FROM node:12 AS build
RUN mkdir ~/project
COPY app/. ~/project
WORKDIR ~/project
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc
RUN npm install

FROM node:12-slim
RUN mkdir ~/project
COPY app/. ~/project
COPY --from=build /~/project/node_modules ~/project/node_modules
WORKDIR ~/project
CMD node index.js
```

<br/>
</details>

# Author

**Docker image security best practices workshop** © [Liran Tal](https://github.com/lirantal), Released under [CC BY-SA 4.0](./LICENSE) License.

# License

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License</a>.
