# syntax = docker/dockerfile:1.0-experimental
# ------------------------------------------------------------------------
#
# Copyright 2018 WSO2, Inc. (http://wso2.com)
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License
#
# ------------------------------------------------------------------------

# set base Docker image to AdoptOpenJDK Alpine Docker image
FROM adoptopenjdk/openjdk11:jdk-11.0.8_10-alpine
LABEL maintainer="WSO2 Docker Maintainers <dev@wso2.org>"

# set Docker image build arguments
# build arguments for user/group configurations
ARG USER=wso2carbon
ARG USER_ID=802
ARG USER_GROUP=wso2
ARG USER_GROUP_ID=802
ARG USER_HOME=/home/${USER}
# build arguments for WSO2 product installation
ARG WSO2_SERVER_NAME=wso2is
ARG WSO2_SERVER_VERSION=5.11.0-m8
ARG WSO2_SERVER_REPOSITORY=product-is
ARG WSO2_SERVER=${WSO2_SERVER_NAME}-${WSO2_SERVER_VERSION}
ARG WSO2_SERVER_HOME=${USER_HOME}/${WSO2_SERVER}
ARG WSO2_SERVER_DIST_URL=https://github.com/wso2/${WSO2_SERVER_REPOSITORY}/releases/download/v${WSO2_SERVER_VERSION}/${WSO2_SERVER}.zip
# build arguments for external artifacts
ARG DNS_JAVA_VERSION=2.1.8
ARG K8S_MEMBERSHIP_SCHEME_VERSION=1.0.7
ARG MYSQL_CONNECTOR_VERSION=8.0.17
# build argument for MOTD
ARG MOTD='printf "\n\
 Welcome to WSO2 Docker Resources \n\
 --------------------------------- \n\
 This Docker container comprises of a WSO2 product, running with its latest GA release \n\
 which is under the Apache License, Version 2.0. \n\
 Read more about Apache License, Version 2.0 here @ http://www.apache.org/licenses/LICENSE-2.0.\n"'
ENV ENV=${USER_HOME}"/.ashrc"

# create the non-root user and group and set MOTD login message
RUN \
    addgroup -S -g ${USER_GROUP_ID} ${USER_GROUP} \
    && adduser -S -u ${USER_ID} -h ${USER_HOME} -G ${USER_GROUP} ${USER} \
    && echo ${MOTD} > "${ENV}"

# create Java prefs dir
# this is to avoid warning logs printed by FileSystemPreferences class
RUN \
    mkdir -p ${USER_HOME}/.java/.systemPrefs \
    && mkdir -p ${USER_HOME}/.java/.userPrefs \
    && chmod -R 755 ${USER_HOME}/.java \
    && chown -R ${USER}:${USER_GROUP} ${USER_HOME}/.java

# copy init script to user home
COPY --chown=wso2carbon:wso2 docker-entrypoint.sh ${USER_HOME}/

# install required packages
RUN apk add --no-cache netcat-openbsd

# add the WSO2 product distribution to user's home directory
RUN \
    wget --no-check-certificate -O ${WSO2_SERVER}.zip "${WSO2_SERVER_DIST_URL}" \
    && unzip -d ${USER_HOME} ${WSO2_SERVER}.zip \
    && chown wso2carbon:wso2 -R ${WSO2_SERVER_HOME} \
    && rm -f ${WSO2_SERVER}.zip

RUN apk update \
 && apk add jq \
 && rm -rf /var/cache/apk/* \
 && apk add wget \
 && rm -rf /var/cache/apk/*

# copy initializer binary to user home
COPY --chown=wso2carbon:wso2 initializer ${USER_HOME}/

# read artifact.json file and copy/remove artifacts based on the content.
COPY --chown=wso2carbon:wso2 artifacts.json .
RUN \
--mount=type=secret,id=GIT_TOKEN GIT_TOKEN=$(cat /run/secrets/GIT_TOKEN); \
jq -c '.[]' artifacts.json | while read i; do \
echo i; \
action=$(echo $i | jq -r '.action'); \
artifact_location=$(echo $i | jq -r '.artifact_location'); \
full_artifact_name=$(echo $i | jq -r '.full_artifact_name'); \
copy_destination=$(echo $i | jq -r '.copy_destination'); \
artifact_repo=$(echo $i | jq -r '.artifact_repo'); \
if [ $action = "remove" ] ; then \
    echo Removing artifact ${WSO2_SERVER_HOME}$artifact_location; \
    rm -rf ${WSO2_SERVER_HOME}$artifact_location; \
    echo Removing finished; \
else \
    if [ $artifact_repo = "github" ] ; then \
        echo Downloading $full_artifact_name; \
        wget --auth-no-challenge --header='Accept:application/octet-stream' https://$GIT_TOKEN:@api.github.com/repos/$artifact_location -O ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
        chown wso2carbon:wso2 ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
    else \
        echo Downloading from external artifact_location $full_artifact_name; \
        wget $artifact_location -O ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
        chown wso2carbon:wso2 ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
    fi ; \
    extract=$(echo $i | jq -r '.extract'); \
    if [ $extract = "true" ] ; then \
        extracted_filename=${full_artifact_name%.*} ; \
        echo extracting as ${WSO2_SERVER_HOME}$copy_destination$extracted_filename; \
        unzip -d ${WSO2_SERVER_HOME}$copy_destination$extracted_filename  ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
        chown -R wso2carbon:wso2 ${WSO2_SERVER_HOME}$copy_destination$extracted_filename; \
        rm -r ${WSO2_SERVER_HOME}$copy_destination$full_artifact_name; \
    fi ; \
fi ; \
done;

# set the user and work directory
USER ${USER_ID}
WORKDIR ${USER_HOME}

# set environment variables
ENV JAVA_OPTS="-Djava.util.prefs.systemRoot=${USER_HOME}/.java -Djava.util.prefs.userRoot=${USER_HOME}/.java/.userPrefs" \
    WORKING_DIRECTORY=${USER_HOME} \
    WSO2_SERVER_HOME=${WSO2_SERVER_HOME}

# expose ports
EXPOSE 4000 9763 9443

# initiate container and start WSO2 Carbon server
ENTRYPOINT ["/home/wso2carbon/docker-entrypoint.sh"]
