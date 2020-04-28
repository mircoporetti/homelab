FROM jenkins4eval/jenkins

USER root

RUN apt-get update -qq \
    && apt-get install -qqy apt-transport-https ca-certificates curl gnupg2 sof$
RUN curl -fsSL https://get.docker.com -o get-docker.sh
RUN sh get-docker.sh

RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.11.5/$
RUN chmod +x ./kubectl
RUN  mv ./kubectl /usr/local/bin/kubectl

RUN usermod -aG docker jenkins