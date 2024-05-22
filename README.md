https://www.youtube.com/watch?v=KQzDiPwclHg&t=11s

Da WSL se ho installato Ansible posso già provare a pingare me stesso
```bash
ansible localhost -m ping # gli dico di usare il MODULO ping
```
Ci sono due elementi essenziali:
- `INVENTORY`: elenco degli host a cui inviare i comandi
- `PLAYBOOK`:  elenco dei comandi da lanciare sugli host


# INVENTORY
- https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#intro-inventory
- Contiene le info sulla infrastruttura di macchine. Può essere un file o una cartella di file (per non usare un file enorme magari)
- Lo scrivo in YAML che è più leggibile
- Lo scrivo in INI se diventa troppo lungo

Di default il file è `/etc/andible/hosts`

Contiene l'elenco degli host (macchine), e l'elenco dei Groups (gruppi di host), e delle variabili (per dare delle proprietà a 1 o + host/group)

```bash
ansible [-i inventory_path] <host> -m ping [-k] # inventario opzionale; -k mi chiede la passw SSH
ansible <hosts> -m module_name -a module_parameters
ansible         -m shell       -a "free -m"          host1 # esempio => magari mi abituo a mettere 
```
Per testarlo, con Vagrant potrei tirarmi su un paio di macchine, serve un Vagrantfile

Creo il file hosts prendendo spunto da hosts2
```bash
ansible -i hosts -m ping --private-key demokey -u vagrant
```

# PLAYBOOK
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html#about-playbooks
- https://ansible.readthedocs.io/projects/lint/rules/command-instead-of-shell/
- Nota: scrivere `ansible.builtin.copy` oppure `copy` nel playbook è la stessa cosa

Nella realtà io userò sempre un `playbook` per le istruzioni da dare agli host, non metto le istruzioni inline

Playbook = elenco dei task (eseguiti sempre) e degli handler (eseguiti se cambia qualcosa) da lanciare sugli host
```bash
ansible-playbook <hosts> <my_playbook.yml>
ansible-playbook -i inventory.yml playbook.yml

ansible-lint    # tool per fare il lint dei playbook: se voglio posso installarlo
ansible-lint -L # elenco dei comandi disponibili
ansible-lint playbook1.yml -v # verifica VERBOSA del playbook
```

### Esempio semplice
```yaml
---                           # delimitatore che indica l'inizio di uno YAML
- name: Basic playbook        # nome facoltativo
  hosts: localhost            # target host dove devo eseguire il playbook
  tasks:                      # task da eseguire nel playbook
    - name: Print a message
      debug:
        msg: "Hello world!!!"
```

### Esempio più complesso
```yaml
---                           
- name: Install Apache webserver        
  hosts: webservers              # elenco target host
  become: yes                    # dice ad Ansible si usare sudo per lanciare i comandi
  tasks:                      
    - name: Install Apache
      apt:                       # modulo Ansible da usare: installatore di pkg
        name: httpd              # pacchetto da installare con apt
        state: present           # il pacchetto deve essere installato, se non è già presente
    - name: Start Apache
      service:                   # modulo Ansible da usare: gestione servizi sugli host target
        name: httpd
        state: started           # il servizio deve essere avviato
        enabled: true            # il servizio deve essere avviato all'avvio della macchina
```
```ini
[webservers]
192.168.33.10
192.168.33.20

[dbservers]
192.168.1.100
```



# Moduli (quelli che eseguo come task o handlers)
- `copy`: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html
- `lineinfile` verifica che ci sia una particolare linea di codice nel file
- `file` fa operazioni sulle proprietà dei file
- `ansible localhost -m setup`


### ES 04: due slave
- https://www.youtube.com/watch?v=iTplOLc64gs&t=580s

Ansible di default si collega in SSH su macchine che hanno già la chiave precondivisa, così da non dover trasmettere la passw di root ad ogni comando. 
Potrei specificare user+passw di ogni macchina su un file in chiaro ma è poco sicuro. Conviene creare la chiave in ogni macchina remota così che Ansible possa collevarsi via SSH senza pasword. 

Se faccio `ssh root@192.168.1.100` mi viene chiesta la chiave e la password di root.
```bash
ssh-keygen # sulla macchina Ansible salvo nel path di default /.ssh senza passphrase
ssh-copy-id root@192.168.1.100 # per copiarla sulla macchina remota
```
Da adesso mi posso collegare in SSH senza dover inserire nessuna password.
```bash
mkdir /etc/ansible/group_vars
# devo dire ad Ansible dov'è il file della chiave per quando si lanciano dei comandi alla macchina remota
vim servers
```

Dentro `servers`: 
```yaml
---
ansible_ssh_user: root
```
Ansible di default cerca di collegarsi con lo stesso utente con cui lancio i comandi, root nel mio caso (in questo caso è inutile quindi il file `servers`)
```bash
ansible -m ping linuxservers # pingo gli host del gruppo linuxservers
ansible -m ping all          # pingo tutti gli host
```
Mi risponderà il primo host ma non il secondo, perchè il secondo non ha ancora la chiave


### ES 05: credenziali in chiaro sulla macchina Ansible (da NON fare)
Sul nuovo `hosts.ini` aggiungo le credenziali di accesso
Su `ansible.cfg` devo aggiungere/decommentare `host_key_checking=False`, poi dalla macchina ansible faccio:
```bash
ansible -m ping linuxservers # pingo gli host del gruppo linuxservers
ansible -m ping all          # pingo tutti gli host
```
Anche se rimuovo le chiavi dalle macchine host, il ping và perchè le stò indicando nella macchina ansible


### ES 06: creo un file sulla macchina Ansible e lo sparo alle altre
```bash
touch playbook.yml # gli metto le istruzioni
vim test.txt       # ci scrivo qualcosa giusto per prova
ansible-playbook playbook.yaml
```


### ES 07: creo un utente "utente" su tutte le macchine e poi lo elimino
```bash
mkdir /etc/ansible/keypairs
ssh-keygen -t rsa # salvo nel path /etc/ansible/keypairs/client1 per comodità
ssh-keygen -t rsa # salvo nel path /etc/ansible/keypairs/client2 
ls /etc/ansible/keypairs # ci son 4 chiavi: 2 private e 2 public. Devo spedire le public alle macchine
```

Nelle macchine client creo le chiavi:
```bash
vim .ssh/authorized_keys # ci copio dentro il contenuto di client1.pub
chmod 700 .ssh/
chmod 00 .ssh/authorized_keys
```

Dalla macchina Ansible io ora posso fare:
```bash
ssh -i /etc/ansible/keypairs/client1 root@192.168.1.100 # qui uso la chiave privata
ssh -i /etc/ansible/keypairs/client2 root@192.168.1.200 

ansible -m ping all
```

Nei playbook la password và scritta hashata. Per farlo basta fare su una macchina qualsiasi
```bash
useradd utente # creo utente
passwd utente  # creo passw: digito "password" per esempio
sudo cat /etc/shadow # in fondo c'è la password hashata (copio fino a prima dei due punti :)
```
Ora posso provare il playbook
```bash
ansible-playbook useradd.yml # 
```
Nelle client provo a fare
```bash
su - utente # mi loggo come utente, dovrebbe essere del gruppo wheel, che fa parte di root
sudo su # dovrebbe poter fare sudo su con password "password"
```

https://www.youtube.com/watch?v=YNmNw7oQpuw 12:36