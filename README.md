# Hochverf-gbarkeits-Load-Balancer
# 4VM's 2 Load Balancer ("HA Proxy", "nginx") 2 Webserver

### Webserver aufsetzen
___

__Auf VM 1 und 2 werden Webserver aufgesetzt__

Werde root user
`sudo su`

Installiere nginx<br>
`yum install nginx`

nginx freischalten und starten<br>
`systemctl enable nginx`

`systemctl start nginx`

Port mit der Firewall freigeben<br>
`firewall-cmd --permanent --add-service=http`

Firewall neu starten<br>
`firewall-cmd --reload`

__Wenn man den Server jetzt aufruft sieht man die nginx default webseite__

Installiere Nano zur editierung<br>
`yum install nano`

Gebe der Startdatei Schreibrechte<br>
`chmod 666 /usr/share/nginx/html/index.html`

Jetzt kann man mit folgenden Befehl die default html datei öffnen und editieren.<br>
`nano /usr/share/nginx/html/index.html`

Anschließend den Server neu starten<br>
`sudo systemctl restart nginx`

__Nun ist wichtig das man den Cache lehrt,
sonst werden die änderungen nicht angezeigt.__

__Wiederhole das für Server 2 mit einer abgeänderten index.html__

<br>

### Reverse Proxy mit load balancer und Keepalived
__Auf VM 3 und 4 werden Proxys eingerichtet__

Werde root user
`sudo su`

HA Proxy installieren<br>
`dnf -y install haproxy`

Installiere Nano<br>
`yum install nano`

Die Datei editierbar machen<br>
`chmod 644 /etc/haproxy/haproxy.cfg`

Den Inhalt der Datei löschen
`echo "" > /etc/haproxy/haproxy.cfg`

Mit nano öffnen und folgenden Code einfügen
`nano /etc/haproxy/haproxy.cfg`

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
    server             node01 192.168.13.54:80 check
    server             node02 192.168.13.131:80 check
```

    
Starte HA Proxy mit diesem Befehl<br>
`systemctl enable --now haproxy`

Öffne die Firewall<br>
`firewall-cmd --add-service=http`

`firewall-cmd --runtime-to-permanent`

Log Konfigurationen anpassen falls nötig<br>
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

**VM 4 aufsetzen**

**Wenn sich dieser Proxy so verhält, wie der andere kann
weiter konfiguriert werden.**

### Hochverfügbarkeit mit Keepalived
__Für VM 3 und 4__<br>
__VM 3 wird MASTER VM 4 BackUp__

Beginne bei Server 3<br>

Werde root user<br>
`sudo su`

Instaliere Keepalived<br>
`dnf install keepalived`

Löschen den Inhalt der Keepalived.conf Datei<br>
`echo "" > /etc/keepalived/keepalived.conf`

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
   unicast_src_ip 192.168.13.251
   unicast_peer {
       192.168.13.212
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
`systemctl restart keepalived`
<br>
Überprüfe ob der Service läuft<br>
`systemctl status keepalived`
<br><br>
__Wenn der Service läuft, wechsle zu VM 4__<br>
__Diese wird der BACK-UP Prroxy__<br>
<br>
Gib dir root rechte
`sudo su`

Instaliere Keepalived<br>
`dnf install keepalived`

Löschen den Inhalt der Keepalived.conf Datei<br>
`echo "" > /etc/keepalived/keepalived.conf`

Öffne die Config Datei und füge den Code ein<br>
`nano /etc/keepalived/keepalived.conf`

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
`systemctl restart keepalived`
<br>
Überprüfe ob der Service läuft<br>
`systemctl status keepalived`
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
