# Hochverf-gbarkeits-Load-Balancer
# 4VM's 2 Load Balancer ("HA Proxy", "nginx") 2 Webserver
### SSH Keys erstellen um sich ohne Passwordeingabe anzumelden.
___
RSA Schlüssel erstellen<br>
`ssh-keygen -t rsa`

Den Privaten Schlüssel auf den Server pushen<br>
`ssh-copy-id janne@janne-test-01.cloud.rto.de`

Remote Shell öffnen<br>
`ssh janne@janne-test-01.cloud.rto.de`

Neues Konsolenfenster öffnen um wieder auf den Home rechner zu sein
Dann ssh key zum zweiten server pushen<br>
`ssh-copy-id janne@janne-test-02.cloud.rto.de`

Anschließend kann man sich verbinden<br>
`ssh janne@janne-test-02.cloud.rto.de`

### Webserver aufsetzen
___

Installiere nginx<br>
`sudo yum install nginx`

nginx freischalten und starten<br>
`sudo systemctl enable nginx`

`sudo systemctl start nginx`

Ports mit der Firewall freigeben<br>
`sudo firewall-cmd --permanent --add-service=http`

`sudo firewall-cmd --permanent --add-service=https`

Firewall neu starten<br>
`sudo firewall-cmd --reload`

__Wenn man den Server jetzt aufruft sieht man die nginx default webseite__

Installiere Nano zur editierung<br>
`sudo yum install nano`

Gebe der Startdatei Schreibrechte<br>
`sudo chmod 666 /usr/share/nginx/html/index.html`

Jetzt kann man mit folgenden Befehl die default html datei öffnen und editieren.<br>
`nano /usr/share/nginx/html/index.html`

Anschließend den Server neu starten<br>
`sudo systemctl restart nginx`

__Nun ist wichtig das man den Cache lehrt,
sonst werden die änderungen nicht angezeigt.__

__Wiederhole das für Server 3__

<br>

### Reverse Proxy mit load balancer
___

__Gehe wieder zu Server 1__

HA Proxy installieren<br>
`sudo dnf -y install haproxy`

Installiere Nano<br>
`sudo yum install nano`

Die Datei editierbar machen<br>
`sudo chmod 644 /etc/haproxy/haproxy.cfg`

Mit nano öffnen und folgenden Code einfügen

```
#  See the full configuration options online.
#  https://www.haproxy.org/download/1.8/doc/configuration.txt

global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    # log zum localen loopback
    log         127.0.0.1 local2

    # Gibt an wo Daten von HA Proxy gespeichert werden sollen
    chroot      /var/lib/haproxy

    # Hier wird die prozess id gespeichert
    pidfile     /var/run/haproxy.pid

    # Maximale anzahl an Verbindungen
    maxconn     4000
    user        haproxy
    group       haproxy

    # Führe den Prozess im Hintergrund aus.
    daemon

    # Legt die Verschlüsselung fest.
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    # Legt dass Kommunikationsprotokol fest
    mode                    http
    # Alle Protokolmeldungen werden in einem log file zusammengefasst
    log                     global
    # Protokolierung von HTTP informationen.
    option                  httplog
    # Lehre Verbindungen sollen nicht protokoliert werden
    option                  dontlognull
    # HTTP-Verbindung soll am ende der Anfrage geschlosen werden
    option http-server-close
    # Dient der weiterleitung der ursprünglichen client ip-adrtesse except schließt die funktion aus
    option forwardfor       except 127.0.0.0/8
    # Ausgefallende Server werden nach dem Neustart wieder für den Lastenausgleich verwendet.
    option                  redispatch
    # Anzahl der Wiederholungsversuche auf den Server zu connecten
    retries                 3
    # Maximale Zeit die eine Anfrage von einem Client dauern darf.
    timeout http-request    10s
    # Maximale Zeit in der eine HTTP anfrage in einer Warteschlange stehen kann bevor sie abgelehnt wird.
    timeout queue           1m
    # Wartezeit für Verbindungsdauer zum Backend Server
    timeout connect         10s
    # So lange wartet HA Proxy auf Daten vom Client nachdem eine Verbindung hergestellt wurde.
    timeout client          1m
    # So lange wartet HA Proxy auf Daten vom Backend Server nachdem eine Verbindung hergestellt wurde.
    timeout server          1m
    # Maximale Zeit die eine Keep-Alive verbindung offen bleibt.
    timeout http-keep-alive 10s
    # Maximale Wartezeit für die Health Check anfrage
    timeout check           10s
    # Legt die Anzahl der Maximalen Verbindungen fest
    maxconn                 3000

frontend http-in
    # listen 80 port
    bind *:80
    # set default backend
    default_backend    backend_servers
    # Ist notwending um die ursprünglinge Client IP-Adresse an den Backend Server weiterzuleiten.
    option             forwardfor

# define backend
backend backend_servers
    # balance with roundrobin
    balance            roundrobin
    # define backend servers
    server             node01 192.168.13.131:80 check
    server             node02 192.168.13.251:80 check
```

    
Starte HA Proxy mit diesem Befehl<br>
`systemctl enable --now haproxy`

Öffne die Firewall<br>
`sudo firewall-cmd --add-service=http`

`sudo firewall-cmd --runtime-to-permanent`

Log Konfigurationen anpassen<br>
`sudo chmod 677 /etc/rsyslog.conf`

`nano /etc/rsyslog.conf`

line 31, 32 : uncomment and add a line<br>
`module(load="imudp") # needs to be done just once`

`input(type="imudp" port="514")`

`$AllowedSender UDP, 127.0.0.1`

line 47 : change like follows<br>

`*.info;mail.none;authpriv.none;cron.none;local2.none    /var/log/messages`

`local2.*                                                /var/log/haproxy.log`

Restart system<br>
`systemctl restart rsyslog`

**Nun funktioniert der Reverse Proxy**

**Setze einen zweiten Reverse Proxy mit dieser Anleitung auf**

**Wenn sich dieser Proxy so verhält, wie der andere kann
weiter konfiguriert werden.**

### Hochverfügbarkeit mit Keepalived

Gehe zu VM 4<br>

Instaliere Keepalived<br>
`sudo dnf install keepalived`

Editiere die Config Datei<br>
`sudo nano /etc/keepalived/keepalived.conf`

```
vrrp_script chk_haproxy {
   script "killall -0 haproxy"   # Dienst prüfen
   weight 2                      # 2 Punkte hinzufügen wenn OK
}

vrrp_instance Instance0 {
   interface enp1s0              # Zu überwachendes Interface
   state MASTER
   virtual_router_id 51          # ID der Route
   unicast_src_ip 192.168.13.212
   unicast_peer {
       192.168.13.54
   }
   priority 101                  # 101 - Master, 100 - Backup
   advert_int 1
   authentication {
       auth_type PASS
       auth_pass 12345
   }
   virtual_ipaddress {
       192.168.12.200            # Die virtuelle IP Adresse
   }
   track_script {
       chk_haproxy
   }
}
```

Starte Keepalived<br>
`sudo systemctl restart keepalived`
<br>
Überprüfe ob der Service läuft<br>
`sudo systemctl status keepalived`
<br><br>
__Wenn der Service läuft, wechsle zu VM 1__
<br>
Instaliere Keepalived<br>
`sudo dnf install keepalived`

Öffne die Config Datei und füge den Code ein<br>
`sudo nano /etc/keepalived/keepalived.conf`

```
vrrp_script chk_haproxy {
   script "killall -0 haproxy"   # Dienst prüfen
   weight 2                      # 2 Punkte hinzufügen wenn OK
}

vrrp_instance Instance0 {
   interface enp1s0              # Zu überwachendes Interface
   state MASTER
   virtual_router_id 51          # ID der Route
   unicast_src_ip 192.168.13.54
   unicast_peer {
           192.168.13.212
   }
   priority 100                  # 101 - Master, 100 - Backup
   advert_int 1
   authentication {
       auth_type PASS
       auth_pass 12345
   }
   virtual_ipaddress {
       192.168.12.200            # Die virtuelle IP Adresse
   }
   track_script {
       chk_haproxy
   }
}
```

Starte Keepalived<br>
`sudo systemctl restart keepalived`
<br>
Überprüfe ob der Service läuft<br>
`sudo systemctl status keepalived`
<br><br>
__Nun sollte bei ausfall des einen Loadbalancers der andere einspringen__

### Meine IP im lokalen DNS eintragen

Öffne die locale DNS Datei<br>
`sudo nano /etc/hosts`
<br>
Schreibe in eine neue Zeile [IP] [Domain] etwa so<br>
`192.168.12.200 janne.rto.io`
<br><br>
__Anschließend sollte man den Loadbalancer über die angegebende Domain aufrufen können.__
