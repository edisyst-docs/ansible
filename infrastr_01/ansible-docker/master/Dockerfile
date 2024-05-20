FROM ubuntu:latest

# Install Ansible and SSH
RUN apt-get update && \
    apt-get install -y ansible openssh-client openssh-server sshpass && \
    apt-get clean

# Create Ansible directory
RUN mkdir -p /etc/ansible

# Generate SSH keys for the master
RUN ssh-keygen -q -N "" -f /root/.ssh/id_rsa

# Copy the public key to known hosts
RUN touch /root/.ssh/known_hosts

# Add ssh config to disable host key checking
RUN echo -e "Host *\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config

# Copy the Ansible configuration files
COPY ansible/ /etc/ansible/

CMD ["/usr/sbin/sshd", "-D"]
