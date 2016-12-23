# Aufgabe 0
VMs in der Cloud aufsetzten, beispielsweise bei AWS.  
Folgendes Ansible-Playbook von [Github](https://github.com/pstauffer/kubernetes-setup) herunterladen und ausführen.

---

# Aufgabe 1

## Cluster aufsetzen
Key `team[A,B,C].pem` uf den Desktop kopieren.  
* [Team1](https://www.adesso.de/tausch/exchange.pl?g=ev3eyym5)
* [Team2](https://www.adesso.de/tausch/exchange.pl?g=8mbuakzg)
* [Team3](https://www.adesso.de/tausch/exchange.pl?g=hfph9mfp)
```bash
chmod 400 team[A,B,C].pem
```
Testen ob man auf den Server connecten kann:
```bash
rob@teama:~/Desktop$ ssh -i team[A,B,C].pem ubuntu@[MASTER:IP]
The authenticity of host '[MASTER:IP] ([MASTER:IP])' can't be established.
ECDSA key fingerprint is SHA256:1WuIoToQUhYCeco87+tanj5trGe+UUH4SYwh9pfzHTk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[MASTER:IP]' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-45-generic x86_64)
...
Last login: Wed Nov  9 08:46:56 2016 from 62.2.43.87
ubuntu@ip-172-30-0-70:~$ 
  ``` 
---

`kubectl` von Master Server auf den Desktop kopieren.
```bash
scp -i team[A,B,C].pem ubuntu@[MASTER:IP]:/usr/bin/kubectl .
```
`admin.conf` vom Master Server auf den Desktop kopieren.
```bash
ssh -i team[A,B,C].pem ubuntu@[MASTER:IP]
ubuntu@MASTER:~# sudo su -
root@MASTER:~# cp /etc/kubernetes/admin.conf /home/ubuntu
root@MASTER:~# chown ubuntu /home/ubuntu/admin.conf
root@MASTER:~# exit
ubuntu@MASTER:~# exit
scp -i team[A,B,C].pem ubuntu@[MASTER:IP]:/home/ubuntu/admin.conf .
```

---


Alias für `kubectl` erstellen:  
File `.alias` im Home-Verzeichnis des Users erstellen mit folgendem Inhalt:
```bash
alias kc='/home/rob/Desktop/kubectl --kubeconfig /home/rob/Desktop/admin.conf'
```
`.bashrc` des Home-Verzeichnisses um folgende Zeile erweitern:
```bash
. ~/.alias
```
Danach den `alias` in der aktuellen Bash ausführen:
```bash
. .alias
```
---
 
Testen ob man die Kubernetes Nodes sieht:
```bash
rob@teama:~/Desktop$ kc get no
NAME             STATUS    AGE
ip-172-30-0-68   Ready     2h
ip-172-30-0-69   Ready     2h
ip-172-30-0-70   Ready     2h
ip-172-30-0-71   Ready     2h
```

---

# Aufgabe 2

## Gogs im Cluster starten
Den Nodes Labels zuweisen:
```bash
kc label nodes ip-172-30-0-68 name=node68
```
Per SSH auf den entsprechenden Node zugreifen:
```bash
ssh -i "team[A,B,C].pem" ubuntu@[PUBLIC_IP]
```
Folgende Ordnerstruktur im Home-Verzeichnis anlegen:
```bash
gogs/data
```

---

Kubernetes-Deployment analog zu [GIST](https://gist.github.com/robertBrem/31b7ad46c8ee531c8dcd575989454825) erstellen.  
Deployment starten:
```bash
kc create -f deployments/gogs.yml
```
Überprüfen ob das Deployment funktioniert hat:
```bash
rob@teama:~/Desktop$ kc get pods
NAME                    READY     STATUS    RESTARTS   AGE
gogs-1417829598-rr88g   1/1       Running   0          <invalid>
```

---

Kubernetes-Service analog zu [GIST](https://gist.github.com/robertBrem/68706f161388b7307bb0) erstellen.  
Service starten:
```bash
kc create -f services/gogs.yml
```
* Gogs einrichten
* Repository einrichten
* Den REST Service ins Repository pushen

