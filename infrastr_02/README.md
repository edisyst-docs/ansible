Questa infrastruttura, parte dalla precedente (infrastr_01) e aggiunge le seguenti funzionalità:
- Sia su master che sugli slave viene creato un utente edoardo:edopassword che sarà l'utente che svolgerà le operazioni su ansible
- nel playbook verrà installato anche composer, per la cui installazione si utilizza anche il file ansible/scripts/install_composer.sh
- il playbook prevede di installare sulle slave apache2, mysql, php8, il progetto Laravel, e di deployarlo lì dentro
- stò provando anche il playbook del corso, per quanto possibile
