# Raspberry Jenkins Build Farm

Progetto Ansible separato per preparare una piccola farm Jenkins su tre Raspberry Pi
con Debian 13 (Trixie) Minimal. Il repository **non modifica** `ansible-ninux-openwrt`
né altri repository applicativi.

## Architettura

- `raspberry-01`: Jenkins Controller e anche agente di compilazione dedicato.
- `raspberry-02`: agente SSH.
- `raspberry-03`: agente SSH.
- La macchina Debian dalla quale esegui Ansible è il control node: Jenkins non viene
  installato su questa macchina.

Gli agenti sono processi separati dal controller. La capacità non viene sommata come
se fosse un unico computer: Jenkins distribuisce i job fra i nodi.

> Nota hardware: Raspberry Pi 3 con microSD e 512 MB/1 GB RAM sono molto limitati.
> Jenkins raccomanda 4 GB+ RAM e 50 GB+ disco per un piccolo team. Il controller su
> Pi 3 è quindi una configurazione sperimentale/laboratorio; le compilazioni OpenWrt
> complete potrebbero fallire per RAM, spazio, temperatura o usura della microSD.
> Valuta almeno un SSD USB e, idealmente, un controller più potente.

## Prerequisiti

1. Debian 13/Trixie (o Debian recente) sulla macchina di controllo.
2. Ansible Core installato sulla macchina di controllo.
3. Accesso SSH `root` ai tre Raspberry con la chiave privata già presente sul control
   node. Il progetto non copia la chiave privata e non la inserisce nel repository.
4. SSH server già attivo sui Raspberry. `raw` permette di installare Python senza
   Python preesistente, ma Ansible deve comunque poter aprire la connessione SSH.
5. I Raspberry devono avere connettività Internet per installare i pacchetti.

Se la chiave non è quella predefinita, imposta `ansible_ssh_private_key_file` in
`inventory/group_vars/all.yml` oppure usa `ssh-agent`.

## 1. Installa Ansible sul control node

Esempio su Debian:

```bash
sudo apt update
sudo apt install -y ansible-core git
ansible-galaxy collection install -r collections/requirements.yml
```

## 2. Estrai il progetto e configura l'inventory

Modifica `inventory/hosts.yml`: gli IP presenti sono **esempi**.

```yaml
all:
  children:
    jenkins_controller:
      hosts:
        raspberry-01:
          ansible_host: 192.168.1.101
    jenkins_agents:
      hosts:
        raspberry-02:
          ansible_host: 192.168.1.102
        raspberry-03:
          ansible_host: 192.168.1.103
  vars:
    ansible_user: root
    ansible_port: 22   # porta SSH dei Raspberry (es. 2400 se non standard)
```

L'utente SSH predefinito è `root`. La stessa `ansible_port` viene usata anche da
Jenkins per collegarsi agli agenti, quindi deve essere la porta reale di `sshd` e
l'utente `jenkins-agent` deve poter accedere via SSH su quella porta. Le variabili comuni si trovano in
`inventory/group_vars/all.yml`. Non inserire password, chiavi private o token in Git.

## 3. Verifica la connessione

```bash
ansible all -m ping
```

Se la chiave non è quella predefinita: `ansible all -m ping --private-key ~/.ssh/nome_chiave`.

Al primo collegamento può essere necessario accettare le host key SSH. Per una LAN
fidata è preferibile verificare le fingerprint e popolare `known_hosts` prima
dell'esecuzione. Non disabilitare permanentemente la verifica host in ambienti non
fidati.

## 4. Esegui il provisioning

Prima esecuzione:

```bash
ansible-playbook playbooks/site.yml
```

Il playbook:
- esegue bootstrap con `raw` (installa Python, sudo e pacchetti base);
- raccoglie i facts hardware e controlla architettura/OS;
- installa Java 21 su tutti i nodi;
- installa Jenkins LTS sul controller (porta e opzioni JVM tramite override systemd);
- installa i plugin con il Plugin Installation Manager ufficiale (il pacchetto
  Debian non include `jenkins-plugin-cli`);
- prepara l'utente `jenkins-agent` e le chiavi SSH per gli agenti;
- registra le host key SSH dei nodi in `/var/lib/jenkins/.ssh/known_hosts`, così
  Jenkins verifica gli agenti invece di accettare qualsiasi chiave;
- configura l'agente locale del controller e gli agenti remoti tramite il plugin
  Jenkins SSH Build Agents;
- applica la configurazione NFS se abilitata.

Per limitare l'esecuzione:

```bash
ansible-playbook playbooks/site.yml --limit raspberry-01
ansible-playbook playbooks/site.yml --tags bootstrap
ansible-playbook playbooks/site.yml --tags hardware
```

## 5. Accesso a Jenkins

Apri `http://IP_RASPBERRY_01:8080` (porta impostabile con `jenkins_http_port`).
La password iniziale si trova sul controller:

```bash
ansible raspberry-01 -m command -a 'cat /var/lib/jenkins/secrets/initialAdminPassword'
# oppure: ssh -p PORTA_SSH root@IP_RASPBERRY_01 'cat /var/lib/jenkins/secrets/initialAdminPassword'
```

Il progetto non crea un account amministratore con password predefinita: completa la
procedura iniziale Jenkins e crea un account personale. Proteggi la porta 8080 sulla
LAN e non esporla direttamente a Internet.

## 6. NFS (opzionale)

In `inventory/group_vars/all.yml`:

```yaml
storage_nfs:
  enabled: false
  server: "192.168.1.10"
  export: "/volume1/nfs-firmware"
  mountpoint: "/mnt/nfs-firmware"
  directories:
    - downloads
    - artifacts
    - cache
```

Imposta `enabled: true` solo dopo aver verificato server, export e permessi. Il ruolo
monta l'export su tutti i nodi. I workspace Jenkins restano locali per evitare
concorrenza e latenza su NFS. La directory NFS viene predisposta, ma la condivisione
della cache `ccache` fra build concorrenti va progettata e verificata separatamente.

## Hardware discovery e executor

I facts Ansible rilevano modello, architettura, CPU, memoria e filesystem. Il ruolo
calcola una stima prudenziale e salva un report in
`/var/lib/jenkins-buildfarm/hardware.json` (un executor ogni 2 core e ogni 2 GB di
RAM, minimo 1). Il playbook si ferma se lo spazio libero su `/` è inferiore a
`hardware_min_free_mb`. La raccomandazione è solo informativa:
`jenkins_agent_executors` può essere impostato globalmente oppure per host
nell'inventory. Il valore predefinito è intenzionalmente conservativo (1 executor
per agente). Il nodo built-in del controller ha `jenkins_controller_executors`
executor (predefinito 0): le build girano sugli agenti, compreso quello locale di
`raspberry-01`.

## Integrazione con il repository OpenWrt

Questo progetto **non clona, modifica o committa** il repository
`https://github.com/mikysal78/ansible-ninux-openwrt`. Il job Jenkins può essere
configurato successivamente per effettuare checkout del repository originale e
richiamare i suoi playbook. Nessun file di quel repository viene gestito da questo
progetto.

## Verifiche e troubleshooting

```bash
ansible all -m setup -a 'filter=ansible_architecture'
ansible all -m command -a 'java -version'
ansible raspberry-01 -m command -a 'systemctl status jenkins'
ansible all -m command -a 'df -h'
```

- `UNREACHABLE`: verifica IP, porta SSH (`ansible_port`), SSH attivo, utente root,
  chiave e firewall.
- Agente offline in Jenkins: controlla che `jenkins-agent` possa accedere via SSH
  sulla porta configurata (eventuali `AllowUsers`/`AllowGroups` in `sshd_config`) e
  che la host key non sia cambiata; in quel caso riesegui il playbook con
  `--tags controller` per aggiornare `known_hosts`.
- `Permission denied`: controlla la chiave pubblica in `authorized_keys` e i
  permessi di `/root/.ssh`.
- `armhf`/`armv7l`: il playbook si ferma prima dell'installazione Jenkins; usa Debian
  arm64/aarch64. Il SoC Pi 3 è capace di 64 bit, ma l'immagine installata può essere
  a 32 bit.
- Jenkins non parte o il sistema va in OOM: Pi 3 può essere sottodimensionato;
  controlla `journalctl -u jenkins -n 100` e `free -h`.

## Sicurezza

- La chiave SSH privata resta sul control node.
- Nessun segreto è versionato.
- La chiave agente è generata sul controller e la parte privata è leggibile solo
  dall'utente Jenkins.
- L'accesso iniziale a Jenkins va completato manualmente via browser per creare
  l'utente amministratore; le installazioni di sistema e la configurazione degli
  agenti sono gestite da Ansible.
