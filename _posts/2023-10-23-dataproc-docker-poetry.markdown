---
layout: post
title: "Creating small Docker images for Dataproc Serverless Pyspark projects managed with Poetry"
date: 2023-10-23 13:00:00 +0200
categories: dataproc docker poetry pyspark
---
Dataproc Serverless on GCP is Google's solution for running Apache Spark workloads without provisioning and managing a Dataproc cluster. It currently supports batch and interactive workloads (using Jupyter notebooks), has up-to-date versions of Apache Spark (up to 3.4 at the time of writing), and has some very useful features such as autoscaling.

It supports all of the common languages for writing your Spark applications: Pyspark, R, Spark SQL and Java/Scala. This blog post assumes you are writing your applications in Pyspark (Python).

# Poetry
If you've never heard of Poetry, check out the official [website](https://python-poetry.org/). Poetry is a dependency manager for Python projects, and can be very useful if your Pyspark project relies on external packages. Poetry is not the focus of this blog post however. Just search the internet with a search term like `poetry python` and you'll find lots of good resources on the subject. A good introduction to using Poetry for a Pyspark project can be found [here](https://mungingdata.com/pyspark/poetry-dependency-management-wheel/).

# Dataproc Serverless custom Docker image
Dataproc Serverless runs your Spark workloads in Docker containers. There's a default container image, but it's not very useful if you have external Python dependencies. That's why custom images are also supported. Google's [documentation](https://cloud.google.com/dataproc-serverless/docs/guides/custom-containers) has some useful information to get you started. They list the following requirements for the Docker image:
- Any OS image can be used as a base image, but Debian 11 images (such as `debian:11-slim`) are preferred.
- The image must contain the packages `procps` and `tini`
- The image requires a `spark` user with a UID and GID of `1099`

The documentation also has some other interesting information. One thing to note is that you must not include the Spark binaries and Java Runtime Environment in your container image, because Dataproc will mount these at runtime to your container. This is nice, because it will keep our custom Docker image quite small. Another important point for us, is that by default Dataproc mounts Conda to `/opt/dataproc/conda`. But, we are not using Conda for dependency management, we're using Poetry. Luckily, Dataproc allows us to specify which Python environment to use in the container image, by specifying the `PYSPARK_PYTHON` environment variable.

There's an example custom Dockerfile in the [documentation](https://cloud.google.com/dataproc-serverless/docs/guides/custom-containers#dockerfile), which shows some interesting details. For example, besides `procps` and `tini`, it also installs the package `libjemalloc2`, so I'm assuming that this is also a required package which Google simply forgot to list. We will be using this example for the first version of our Docker image.

# Custom Dockerfile, first version
{% highlight Dockerfile %}
FROM python:3.9-slim

# Suppress interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# (Required) Install utilities required by Spark scripts.
RUN apt update && apt install -y procps tini libjemalloc2

# Enable jemalloc2 as default memory allocator
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2

# Instantiating environment variables
ENV SPARK_EXTRA_JARS_DIR=/opt/spark/jars/
ENV SPARK_EXTRA_CLASSPATH='/opt/spark/jars/*'
ENV PYTHONPATH=/src
RUN mkdir -p "${SPARK_EXTRA_JARS_DIR}"
RUN mkdir -p "${PYTHONPATH}"

WORKDIR /

COPY pyproject.toml /pyproject.toml
COPY poetry.lock /poetry.lock

ENV PYSPARK_PYTHON=/usr/local/bin/python3.9
ENV PATH=${PYSPARK_PYTHON}:${PATH}

RUN python3.9 -m pip install --upgrade pip \
    && python3.9 -m pip install poetry==1.6.1
RUN poetry config virtualenvs.create false \
    && poetry install 

# (Optional) Add extra Python modules.
COPY ./jobs/ "${PYTHONPATH}"/jobs

# (Required) Create the 'spark' group/user.
# The GID and UID must be 1099. Home directory is required.
RUN groupadd -g 1099 spark
RUN useradd -u 1099 -g 1099 -d /home/spark -m spark
USER spark
{% endhighlight %}

Let's break down this Dockerfile to understand what is going on:
- First of all, even though Google suggests to use a Debian 11 base image, we are using `python:3.9-slim` here, which is based on the latest stable version of Debian. But it's very convenient for us to use this image, because it comes with a Python environment pre-installed.
- We copy the `pyproject.toml` and `poetry.lock` files from the root of our repository into the Docker image. These files are needed by Poetry to install the external dependencies exactly as they are specified.
- We use `pip` to install Poetry in the Docker image
- By setting `virtualenvs.create` to `false`, Poetry will not create a virtual environment, but instead install all dependencies in the systems Python environment. This is fine as long as you do this inside a Docker container, and for us it makes the usage of the Docker image simpler, but it is normally recommended to always create virtual environments for your projects
- Because Poetry installed all the dependencies in the system Python environment, we can simply set `PYSPARK_PYTHON` to `/usr/local/bin/python3.9`
- This example assumes the Pyspark project repository has all the Pyspark code in a `/jobs` directory, which we will copy as a whole to `$PYTHONPATH/jobs`, in this case `/src/jobs`

Build the image and push it to Google Artifact Registry. You can then test your Spark job from the command line:
{% highlight sh %}
gcloud dataproc batches submit \
--batch <batch name> \
--project <your GCP project name> \
--region <your GCP region> \
pyspark file:///src/jobs/path/to/your/spark/app.py \
--container-image <location of your custom image> \
--subnet default \
--version 2.0 \
--properties <your spark properties>
{% endhighlight %}

This first version of our Docker image should work fine, but we can improve on it. Can we make the image smaller in size? We are installing Poetry into the image, but only to be able to run the Poetry commands to install the dependencies. After that's done, Poetry doesn't serve any purpose for our Spark applications. Is there a way to use Poetry as a dependency manager, without having Poetry in the final Docker image, using up unnecessary disk space? Inspired by [this](https://stackoverflow.com/questions/53835198/integrating-python-poetry-with-docker/64642121#64642121) StackOverflow post, let's get to work on our second version.
# Custom Dockerfile, second version
{% highlight Dockerfile %}
FROM python:3.9-slim AS base

# Suppress interactive prompts
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONFAULTHANDLER=1 \
    PYTHONHASHSEED=random \
    PYTHONUNBUFFERED=1

WORKDIR /

FROM base AS builder

ENV DEBIAN_FRONTEND=noninteractive \
    PIP_DEFAULT_TIMEOUT=100 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    POETRY_VERSION=1.6.1

RUN python3.9 -m pip install --upgrade pip \
    && python3.9 -m pip install poetry==${POETRY_VERSION}

RUN python3.9 -m venv /venv

COPY pyproject.toml poetry.lock ./

RUN  poetry export -f requirements.txt | /venv/bin/pip install  -r /dev/stdin

FROM base AS final

ENV DEBIAN_FRONTEND=noninteractive

# (Required) Install utilities required by Spark scripts.
RUN apt-get update && apt-get install -y --no-install-recommends procps tini libjemalloc2

# Enable jemalloc2 as default memory allocator
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2

# Instantiating environment variables
ENV SPARK_EXTRA_JARS_DIR=/opt/spark/jars/
ENV SPARK_EXTRA_CLASSPATH='/opt/spark/jars/*'
ENV PYTHONPATH=/src
RUN mkdir -p "${SPARK_EXTRA_JARS_DIR}"
RUN mkdir -p "${PYTHONPATH}"

COPY --from=builder /venv /venv

ENV PYSPARK_PYTHON=/venv/bin/python
ENV PATH=${PYSPARK_PYTHON}:${PATH}

# (Optional) Add extra Python modules.
COPY ./jobs/ "${PYTHONPATH}"/jobs

# (Required) Create the 'spark' group/user.
# The GID and UID must be 1099. Home directory is required.
RUN groupadd -g 1099 spark
RUN useradd -u 1099 -g 1099 -d /home/spark -m spark
USER spark
{% endhighlight %}

The Dockerfile is very similar to the one from the original StackOverflow post. So let's see what's going on.
- First of all, this Dockerfile utilizes a [multi-stage build](https://docs.docker.com/build/building/multi-stage/), with two separate stages. Stage one creates a Python virtual environment with all of the project's dependencies, and stage two copies that virtual environment into the final image
- Instead of installing all dependencies into the system Python environment, this time we are creating a virtual environment. This makes it very simple to copy the environment into the final image. All we then have to do, is to point `PYSPARK_PYTHON` to `/venv/bin/python`.
- We don't use Poetry to create the virtual environment. Instead, we use Poetry's `export` command and pipe it into `pip`, to install the virtual environment
- The final image will not have Poetry, saving us a couple of hundred MB of disk space

Build the image and push it to Artifact Registry again. In the GCP UI you can see the image size and compare it to the previous version. Hopefully, the new image will be significantly smaller!

Run into any issues? See mistakes in the examples? Please let me know by dropping me an e-mail.
