# Base Image

FROM ubuntu:latest

RUN apt-get update; \
    apt install -y openssh-client; \
    apt install -y python3-pip


RUN pip3 install --upgrade pip; \
    pip3 install "ansible==2.9.12"
