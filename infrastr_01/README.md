# Problema
Creare con un Docker container una `infrastruttura costituita da 3 macchine`:
una (`master`) che tramite `Ansible` potrà controllare, comandare, installare pacchetti e gestire le altre 2 macchine (`slave1` e `slave2`). 
Le 3 macchine devono essere collegate tra loro in una `rete Docker`. 
Nella macchina `master` deve essere configurato `Ansible` affinché possa lanciare i comandi alle altre 2 macchine. 


# Soluzione docker-compose
Per creare una infrastruttura Docker con una macchina `master` che utilizza `Ansible` per gestire due macchine `slave`, 
è più conveniente utilizzare `Docker Compose`. 
`Docker Compose` consente di gestire facilmente la configurazione della rete e dei container.

Ecco i passaggi dettagliati per configurare questa infrastruttura:

### 1. Creare il file docker-compose.yml
Nella directory principale del progetto, crea un file `docker-compose.yml`:

```yaml
version: '3'
services:
  master:
    build: master
    container_name: master
    networks:
      - mynet
    volumes:
      - ./master/ansible:/etc/ansible
  slave1:
    build: slave
    container_name: slave
    networks:
      - mynet
  slave2:
    build: slave
    container_name: slave2
    networks:
      - mynet

networks:
  mynet:
    driver: bridge
```

### 2. Creare la directory per il master e per le slave
Crea una directory per il progetto e all'interno di essa crea altre tre directory: `master`, `slave1` e `slave2`.
```bash
mkdir ansible-docker
cd ansible-docker
mkdir master slave slave2
```


### 3. Creare i Dockerfile per master e slave:
- Dockerfile per il master (`master/Dockerfile`):
```dockerfile
FROM ubuntu:latest

# Install Ansible and SSH
RUN apt-get update && \
    apt-get install -y ansible openssh-client openssh-server && \
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
```

- Dockerfile per le slave (`slave/Dockerfile` e `slave2/Dockerfile`):
```dockerfile
FROM ubuntu:latest

# Install SSH server
RUN apt-get update && \
    apt-get install -y openssh-server && \
    apt-get clean

# Configure SSH server
RUN mkdir /var/run/sshd
RUN echo 'root:root' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# Install any additional packages you need
RUN apt-get install -y python3

CMD ["/usr/sbin/sshd", "-D"]
```

### 4.  Configurare Ansible nel master:
Crea i file di configurazione di Ansible nella directory **master/ansible**.

`master/ansible/ansible.cfg`: Configura Ansible per eseguire i comandi in modalità locale.
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
```

`master/ansible/hosts`: Definisci gli host su cui eseguire i comandi.
```ini
[slaves]
slave1 ansible_host=slave1
slave2 ansible_host=slave2
```

`master/ansible/playbook.yml`: Definisci i comandi da eseguire sulle macchine slave.
```yaml
---
- hosts: slaves
  tasks:
    - name: Ensure Python is installed
      apt:
        name: python3
        state: present
    - name: Test command
      command: echo "Hello from {{ inventory_hostname }}"
```

## 5. Copiare la chiave SSH pubblica del master nelle slave
Aggiungi al file `docker-compose.yml` i seguenti comandi per copiare la chiave pubblica SSH del master nelle slave:
```yaml
services:
  master:
    ...
    command: /bin/bash -c "while ! ssh-keyscan slave slave2 > /root/.ssh/known_hosts; do sleep 1; done && sshpass -p root ssh-copy-id root@slave && sshpass -p root ssh-copy-id root@slave2 && /usr/sbin/sshd -D"

```

## 6. Avviare i container
Nella directory principale del progetto, esegui il comando:
```bash
docker-compose up --build
```

## 7. Verificare l'infrastruttura
Una volta che i container sono in esecuzione, puoi accedere al container master ed eseguire il playbook di Ansible:
```bash
docker exec -it master bash
ansible-playbook /etc/ansible/playbook.yml
```

Questa configurazione ti permette di avere una macchina master che utilizza Ansible per gestire due macchine slave, 
tutte collegate tramite una rete Docker. 
Ansible può quindi inviare comandi, installare pacchetti e gestire le slave come desiderato.

