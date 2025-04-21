Questa infrastruttura, parte dalla precedente (infrastr_02) e aggiunge al docker-compose.yml, i seguenti servizi:
- un'istanza di un database mysql, comprensiva di un suo volume (dove salvare i dati e i DB)
- un'istanza di phpmyadmin (per avere un'interfaccia dei DB del container mysql)
- probabilmente su master e slave dovrò installare anche il pacchetto mysql-client, per poter interagire con il DB mysql da dentro quei container (se installo Laravel sugli slave, dovrò installare anche il mysql)
 

Il pacchetto mysql lo installo con il comando:
```bash 
RUN apt-get update && apt-get install -y mysql-client
```

Avvio il docker-compose con il comando:
```bash
docker-compose up -d
docker compose up -d --build
```
