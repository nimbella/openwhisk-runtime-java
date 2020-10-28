#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# build go proxy from source
FROM golang:1.12 AS builder_source
RUN env CGO_ENABLED=0 go get github.com/apache/openwhisk-runtime-go/main && mv /go/bin/main /bin/proxy

# or build it from a release
FROM golang:1.12 AS builder_release
ARG GO_PROXY_RELEASE_VERSION=1.12@1.15.0
RUN curl -sL \
  https://github.com/apache/openwhisk-runtime-go/archive/{$GO_PROXY_RELEASE_VERSION}.tar.gz\
  | tar xzf -\
  && cd openwhisk-runtime-go-*/main\
  && GO111MODULE=on go build -o /bin/proxy

# Use AdoptOpen JDK8, OpenJ9 release version 0.15.1
FROM adoptopenjdk/openjdk8-openj9:x86_64-ubuntu-jdk8u222-b10_openj9-0.15.1

# select the builder to use
ARG GO_PROXY_BUILD_FROM=release

RUN rm -rf /var/lib/apt/lists/* \
    && apt-get clean \
    && apt-get update \
    && apt-get -y --no-install-recommends upgrade \
    && apt-get -y --no-install-recommends install locales python vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && locale-gen en_US.UTF-8

ENV LANG="en_US.UTF-8" \
	LANGUAGE="en_US:en" \
	LC_ALL="en_US.UTF-8" \
	VERSION=8 \
	UPDATE=222 \
	BUILD=10

RUN locale-gen en_US.UTF-8 ;\
    mkdir -p /javaAction/action /usr/java/src /usr/java/lib

WORKDIR /javaAction

COPY --from=builder_source /bin/proxy /bin/proxy_source
COPY --from=builder_release /bin/proxy /bin/proxy_release
RUN mv /bin/proxy_${GO_PROXY_BUILD_FROM} /bin/proxy

# Add maven
ARG MAVEN_VERSION=3.6.3
ARG USER_HOME_DIR="/root"
ARG BASE_URL=https://apache.osuosl.org/maven/maven-3/${MAVEN_VERSION}/binaries
RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
 && curl -fsSL -o /tmp/apache-maven.tar.gz ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
 && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
 && rm -f /tmp/apache-maven.tar.gz \
 && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn
ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"

# Add nim
RUN curl https://apigcp.nimbella.io/downloads/nim/nim-install-linux.sh | bash

ADD https://search.maven.org/remotecontent?filepath=com/google/code/gson/gson/2.8.5/gson-2.8.5.jar /usr/java/lib/gson-2.8.5.jar
ADD lib/src/Launcher.java /usr/java/src/Launcher.java
RUN cd /usr/java/src ;\
    javac -cp /usr/java/lib/gson-2.8.5.jar Launcher.java ;\
    jar cvf /usr/java/lib/launcher.jar *.class
ADD bin/compile /bin/compile
ENV OW_COMPILER=/bin/compile
ENV OW_SAVE_JAR=exec.jar
ENTRYPOINT /bin/proxy
