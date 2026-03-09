# Deploy Microservices in OpenShift

In this lab, you will deploy a full-stack JavaScript application as microservices in OpenShift. You will learn how OpenShift can build container images directly from source code using Source-to-Image (S2I), how to connect microservices using environment variables, and how to add health checks so OpenShift can monitor your application.

This lab has three parts:

1. **Deploy the front end** from a pre-built container image
2. **Build and deploy the back end** from source code using S2I
3. **Connect the services** using environment variables and add health monitoring

## Why Microservices?

Traditional monolithic applications bundle everything — the user interface, business logic, and data access — into a single deployable unit. Microservices split these responsibilities into independent services that can be developed, deployed, and scaled separately.

The application you will deploy is a URL shortener with a React front end and a Node.js back end. Each component runs in its own Pod with its own Service, and they communicate over HTTP. This separation means you can update the front end without touching the back end, scale them independently, and use different technologies for each.

## Project Setup

Create a dedicated project for this lab:

```bash
oc new-project microservices
```

## Part 1: Deploy the Front End

The front end is a React application that has already been built and published as a container image on Docker Hub. This is the simplest deployment model — you point OpenShift at an existing image, and it creates all the necessary resources.

### Deploy the front-end image

The `oc new-app` command can create an application from a container image, a Git repository, or a template. Here you will use it with a container image:

```bash
oc new-app nodeshift/urlshortener-front
```

OpenShift pulls the image from Docker Hub and creates three resources:
- An **ImageStream** that tracks the image
- A **Deployment** that manages the Pod
- A **Service** that provides internal networking

### Expose the front end

The front end is running inside the cluster, but no one can reach it from outside. Create a Route to expose it:

```bash
oc expose svc/urlshortener-front --port=8080
```

The `--port=8080` flag tells the Route which Service port to target, since this Service exposes multiple ports.

### Verify the front end

Get the Route URL:

```bash
oc get route urlshortener-front
```

Open the URL from the `HOST/PORT` column in your browser. You should see the URL shortener front end with:
- A list of redirection URLs (currently empty)
- A form to add new URLs (not yet functional)
- An **About** page showing the status of the application components

On the About page, all the status indicators are red because the front end has no back end to talk to. You will fix this in Part 3.

## Part 2: Build and Deploy the Back End

Instead of using a pre-built image, you will build the back end directly from source code using OpenShift's **Source-to-Image (S2I)** toolkit.

### What is Source-to-Image?

S2I is a build process built into OpenShift that:
1. Detects the programming language from your source code
2. Selects an appropriate builder image (Node.js, Python, Java, etc.)
3. Combines your source code with the builder image to produce a runnable container
4. Pushes the resulting image to the internal registry

This means developers can deploy applications without writing a Dockerfile. OpenShift handles the entire build process.

### Build the back end from GitHub

The back-end source code is in the `/back` directory of a GitHub repository. Use `oc new-app` with the repository URL, specifying both the builder image and the source directory:

```bash
oc new-app --image-stream="openshift/nodejs:20-ubi9" https://github.com/nodeshift-blog-examples/urlshortener --context-dir=back
```

Let's break down this command:
- **--image-stream="openshift/nodejs:20-ubi9"** — Use the Node.js 20 builder image based on Red Hat Universal Base Image 9. OpenShift has multiple Node.js versions available; this flag selects a specific one.
- **https://github.com/nodeshift-blog-examples/urlshortener** — The Git repository containing the source code.
- **--context-dir=back** — Only use the `/back` subdirectory of the repository.

You should see output indicating that a build has started:

```
--> Found image ... in image stream "openshift/nodejs" under tag "20-ubi9" for "openshift/nodejs:20-ubi9"

    Node.js 20
    ----------
    Node.js 20 available as container is a base platform for building and running various Node.js 20 applications and frameworks.

    * A source build using source code from https://github.com/nodeshift-blog-examples/urlshortener will be created
      * The resulting image will be pushed to image stream tag "urlshortener:latest"
      * Use 'oc start-build' to trigger a new build

--> Creating resources ...
    imagestream.image.openshift.io "urlshortener" created
    buildconfig.build.openshift.io "urlshortener" created
    deployment.apps "urlshortener" created
    service "urlshortener" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/urlshortener' to track its progress.
```

### Monitor the build

Follow the build logs in real time:

```bash
oc logs -f buildconfig/urlshortener
```

You will see OpenShift clone the repository, install Node.js dependencies with `npm install`, and build the container image. This takes a few minutes. When the build completes, you will see `Push successful`.

Press `Ctrl+C` if the log stream does not exit automatically after the build completes.

### Configure the back-end port

The Node.js application uses an environment variable called `PORT` to determine which port to listen on. The S2I builder expects applications to listen on port 8080. Set this explicitly:

```bash
oc set env deployment/urlshortener PORT=8080
```

This triggers a new rollout of the Deployment with the updated environment variable. OpenShift performs a rolling update — the new Pod starts before the old one is removed, ensuring zero downtime.

### Expose the back end

Create a Route for the back end:

```bash
oc expose svc/urlshortener
```

There is no need to specify the port because the S2I builder configured the Service to use port 8080 by default.

### Test the back end

Get the back-end Route URL:

```bash
oc get route urlshortener
```

Test the root endpoint:

```bash
curl -s http://$(oc get route urlshortener -o jsonpath='{.spec.host}')
```

You should see:

```
{"msg":"Hello"}
```

Test the health endpoint:

```bash
curl -s http://$(oc get route urlshortener -o jsonpath='{.spec.host}')/health
```

You should see:

```
{"server":true,"database":false}
```

The server is running (`true`), but there is no MongoDB database deployed (`false`). This is expected for this lab.

## Part 3: Connect the Services

Now you will connect the front end to the back end and add health monitoring.

### Add a health check

OpenShift can periodically call an endpoint on your Pod to verify the application is still responding. This is called a **liveness probe**. If the probe fails repeatedly, OpenShift restarts the Pod automatically.

Add a liveness probe that calls the `/health` endpoint every 10 seconds:

```bash
oc set probe deployment/urlshortener --liveness --get-url=http://:8080/health --initial-delay-seconds=10 --period-seconds=10
```

- **--liveness** — This is a liveness probe (checks if the application is alive).
- **--get-url=http://:8080/health** — Send an HTTP GET to port 8080 on the `/health` path. As long as it returns an HTTP 200 status code, the Pod is considered healthy.
- **--initial-delay-seconds=10** — Wait 10 seconds after the container starts before sending the first probe. This gives the application time to initialize.
- **--period-seconds=10** — Check every 10 seconds.

Verify the probe was added:

```bash
oc describe deployment/urlshortener | grep -A5 "Liveness"
```

You should see:

```
    Liveness:   http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
```

You can confirm the probe is working by checking the back-end Pod logs. You should see periodic `GET /health` requests:

```bash
oc logs deployment/urlshortener --tail=5
```

### Link the front end to the back end

The front-end application uses a `BASE_URL` environment variable to know where the back end is. You need to set this to the back-end Route URL.

Get the back-end Route and set it as an environment variable on the front-end Deployment in a single command:

```bash
oc set env deployment/urlshortener-front BASE_URL=http://$(oc get route urlshortener -o jsonpath='{.spec.host}')
```

This triggers a new rollout of the front-end Deployment with the `BASE_URL` configured.

### Verify the connection

Wait a few seconds for the new front-end Pod to start, then open the front end in your browser:

```bash
oc get route urlshortener-front -o jsonpath='{.spec.host}'
```

Navigate to the **About** page. The **Node.js API** status indicator should now show green with "Up and running." The **Database** and **Redirector** indicators remain red because those services are not deployed in this lab.

### Review all resources

View everything that was created in this project:

```bash
oc get all
```

You should see:
- Two Deployments (front end and back end)
- A completed Build Pod from the S2I process
- Two Services for internal communication
- Two Routes for external access
- A BuildConfig and ImageStream for the S2I-built back end

## Cleanup

Delete the project and all resources:

```bash
oc delete project microservices
```

## Congratulations

You have deployed a microservices application in OpenShift using two different methods:

- **Pre-built container images** — The front end was deployed from a Docker Hub image using `oc new-app`. This is the simplest approach when images are already available.
- **Source-to-Image (S2I)** — The back end was built directly from GitHub source code. OpenShift detected the Node.js runtime, built the container, and pushed it to the internal registry — no Dockerfile required.
- **Environment variables** connected the two services, demonstrating how microservices communicate in a loosely coupled way.
- **Liveness probes** allow OpenShift to continuously monitor the back end and automatically restart it if it stops responding.

These patterns — deploying from images, building from source, connecting services through configuration, and adding health monitoring — are the foundation of running production microservices in OpenShift.
