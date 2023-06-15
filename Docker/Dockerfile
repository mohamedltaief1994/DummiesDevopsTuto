From jenkins/inbound-agent
User root
#install kubectl 
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && install -o root -g root -m 0756 kubectl /usr/local/bin/kubectl
#install docker
RUN apt-get update && \
       apt-get -y install apt-transport-https \
       ca-certificates \
       curl \
       gnupg2 \
       software-properties-common && \
       curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
       add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
       $(lsb_release -cs) \
       stable" && \
       apt-get update && \
       apt-get -y install docker-ce

##ensures that /var/run/docker.sock exists
RUN touch /var/run/docker.sock

##changes the ownership of /var/run/docker.sock
RUN chown root:docker /var/run/docker.sock

##gives jenkins user permissions to access /var/run/docker.sock
RUN usermod -a -G docker jenkins
#install node 16
RUN curl -fsSL https://deb.nodesource.com/setup_16.x | bash - &&\
apt-get install -y nodejs
#install jdk graalvm
COPY graalvm-ce-java17-22.3.1/ /opt/graalvm-ce-java17-22.3.1/
COPY jdk-11.0.14/ /opt/jdk-11.0.14/
COPY maven/ /opt/maven/
ENV M2_HOME='/opt/maven'
ENV PATH="$M2_HOME/bin:$PATH"
#USER jenkins
