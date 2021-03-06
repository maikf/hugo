---
layout: post
title:  "vpnc und OpenVPN auf OpenWrt"
date:   2009-01-11 00:00:00
categories: Artikel
Aliases:
  - /Artikel/openwrt_vpn
---



<p>
In manchen Netzen braucht man vpnc, um ins Internet zu kommen. Wenn man nun
einem solchen Netz nicht vertraut, kann man auch ein OpenVPN durch den vpnc
fahren. Wenn man das nun auch gleich noch mehreren Leuten zur Verfügung stellen
möchte, ohne dass jeder diese Konfiguration machen muss, dann ist ein mit
OpenWrt betriebener Router eigentlich ideal. In dieser Anleitung zeige ich,
worauf man achten muss und wie man vorgeht.
</p>

<h2>OpenWrt Kamikaze selbst kompilieren</h2>

<p>
Nachdem ich eine vorkompilierte Version von OpenWrt ausprobierte, merkte ich,
dass der Platz zu schnell eng wurde. Außerdem muss man ohnehin das vpnc-Paket
selbst bauen, da es nicht mit OpenSSL-Unterstützung verbreitet werden darf (das
alte Lied von den inkompatiblen Lizenzen :-(). Da kann man sich dann auch
gleich ein ganzes zugeschnittenes Firmware-Image bauen.
</p>

<p>
Wir holen uns also die aktuelle Version aus dem Subversion und holen die
externen Pakete, die wir brauchen:
</p>

<pre>
x200 $ svn co https://svn.openwrt.org/kamikaze/trunk
x200 $ cd trunk
x200 $ ./scripts/feeds update
x200 $ ./scripts/feeds install openvpn vpnc
</pre>

<p>
Eventuell muss man die vorkompilierten Tools zum Konfigurieren neu kompilieren,
bei mir ging es zumindest nicht anders:
</p>

<pre>
x200 $ cd tools/config
x200 $ make clean
x200 $ make
x200 $ cd ../..
</pre>

<p>
Anschließend kann man via make menuconfig konfigurieren, welche Bestandteile
man im fertigen Image haben möchte. Ich habe in erster Linie ppp entfernt und
vpnc, openvpn und openssl aktiviert.
</p>

<p>
Meine <a href="/OpenWrt_vpn_config.txt">.config-Datei findest du hier</a>.
</p>

<h2>Patch an vpnc</h2>

<p>
Ich habe die Datei
<code>package/feeds/packages/vpnc/patches/001-cross.patch</code> durch folgende
ausgetauscht (damit vpnc mit OpenSSL- und somit Zertifikat-Support kompiliert
wird):
</p>

<pre>
--- a/Makefile.O	2009-01-09 16:31:04.391983895 +0100
+++ b/Makefile	2009-01-09 16:31:18.701204422 +0100
@@ -20,7 +20,7 @@
 # $Id: Makefile 312 2008-06-15 18:09:42Z Joerg Mayer $
 
 DESTDIR=
-PREFIX=/usr/local
+PREFIX=/usr
 ETCDIR=/etc/vpnc
 BINDIR=$(PREFIX)/bin
 SBINDIR=$(PREFIX)/sbin
@@ -47,18 +47,15 @@
 
 # Comment this in to obtain a binary with certificate support which is
 # GPL incompliant though.
-#OPENSSL_GPL_VIOLATION = -DOPENSSL_GPL_VIOLATION
-#OPENSSLLIBS = -lcrypto
+OPENSSL_GPL_VIOLATION = -DOPENSSL_GPL_VIOLATION
+OPENSSLLIBS = -lcrypto
 
 CC=gcc
-CFLAGS ?= -O3 -g
-CFLAGS += -W -Wall -Wmissing-declarations -Wwrite-strings
-CFLAGS +=  $(shell libgcrypt-config --cflags)
+CFLAGS += -W -Wall -Wmissing-declarations -Wwrite-strings -I$(STAGING_DIR)/usr/include -I$(STAGING_DIR)/include $(OFLAGS) '-DVERSION="$(shell cat VERSION)"'
 CPPFLAGS += -DVERSION=\"$(VERSION)\" $(OPENSSL_GPL_VIOLATION)
-LDFLAGS ?= -g
-LDFLAGS += $(shell libgcrypt-config --libs) $(OPENSSLLIBS)
+LDFLAGS = -L$(STAGING_DIR)/usr/lib -L$(STAGING_DIR)/lib -lgcrypt -lgpg-error $(OPENSSLLIBS)
 
-ifeq ($(shell uname -s), SunOS)
+ifeq ($(OS), SunOS)
 LDFLAGS += -lnsl -lresolv -lsocket
 endif
 ifneq (,$(findstring Apple,$(shell $(CC) --version)))
</pre>

<h2>Flashen und Konfiguration</h2>

<p>
Nach einem anschließenden make hat man nach einigen Minuten in
<code>bin/</code> ein Firmware-Image
(<code>openwrt-brcm-2.4-squashfs.trx</code>), welches man auf den Router
flasht.
</p>

<p>
Danach loggt man sich via Telnet ein, setzt ein Passwort und loggt sich über
SSH neu ein:
</p>

<pre>
x200 $ telnet 192.168.1.1
OpenWrt # passwd
x200 $ ssh root@192.168.1.1
</pre>

<p>
Nun ändere ich die IP-Adresse für das LAN, sodass diese nicht mit der auf dem
WAN kollidiert und aktiviere DHCP auf den LAN-Ports:
</p>

<pre>
OpenWrt # uci set network.lan.ipaddr="192.168.2.1"
OpenWrt # uci set dhcp.lan.force=1
OpenWrt # uci commit
OpenWrt # /etc/init.d/network restart
</pre>

<p>
Standardmäßig benutzt das WAN übrigens DHCP, wenn du eine statische
Konfiguration benötigst, musst du diese vornehmen (siehe <a
href="http://wiki.openwrt.org/OpenWrtDocs/KamikazeConfiguration">OpenWrt
Dokumentation</a>).
</p>

<h2>OpenVPN</h2>

<p>
Anschließend aktiviere ich die Änderungen und kopiere meinen Secret-Key auf den
Router:
</p>

<pre>
OpenWrt # /etc/init.d/network restart
x200 $ scp secret.txt root@192.168.2.1:/etc/openvpn_secret.txt
OpenWrt # chmod 600 /etc/openvpn_secret.txt
</pre>

<p>
Leider existiert momentan ein <a href="https://dev.openwrt.org/ticket/4439">Bug
im Initscript</a>, das bei OpenWrt mitkommt, sodass man manuell „secret” bei
append_params in /etc/init.d/openvpn anhängen muss, da er ansonsten die Option
nicht erkennt. Wenn du dein VPN nicht mit einem Secret-Key betreibst, betrifft
dich das nicht.
</p>

<p>
Dann geht es an die Konfiguration von OpenVPN, die in OpenWrt über
/etc/config/openvpn geschieht. Die Optionen entsprechen denen, die man OpenVPN
via Kommandozeile oder Konfigurationsdatei mitgeben kann. Lediglich müssen
Unterstriche statt Bindestriche verwendet werden:
</p>

<pre>
config openvpn companyvpn
        option enable 1
        option dev tun
        option proto udp
        option redirect_gateway "def1"
        option ifconfig "10.23.42.22 10.23.42.21"
        list remote "company.server.example.com 25011"
        option comp_lzo 1
        option mute_replay_warnings 1
        option secret "/etc/openvpn_secret.txt"
</pre>

<p>
Derzeit ist die OpenWrt-Firewall nicht in der Lage, das Forwarding und NAT für
Tunnel-Interfaces komfortabel zu konfigurieren (wenn man es probiert, ohne
diese Optionen einzustellen, erhält man die Fehlermeldung „Destination port
unreachable”), stattdessen schreiben wir uns eine eigene Ergänzung für die
iptables-Konfiguration in /etc/firewall.user:
</p>

<pre>
iptables -I FORWARD -o tun+ -j ACCEPT
iptables -t nat -I POSTROUTING -o tun+ -j MASQUERADE
</pre>

<p>
…und binden diese ein:
</p>

<pre>
OpenWrt # uci add firewall include
OpenWrt # uci set firewall.@include[0].path=/etc/firewall.user
OpenWrt # uci commit
OpenWrt # /etc/init.d/firewall restart
</pre>

<p>
Sofern du nur ein OpenVPN konfigurieren wolltest, bist du hiermit fertig. Mit
/etc/init.d/openvpn start (passiert beim Booten automatisch) kannst du das VPN
starten.
</p>

<h2>vpnc</h2>

<p>
Ich gehe davon aus, dass du bereits eine funktionierende vpnc-Installation auf
einem deiner Rechner hast. Falls nicht, kannst du nach <a
href="http://www.rzuser.uni-heidelberg.de/~ic001/vpnc-howto.html">dieser
vpnc-Anleitung</a> vorgehen.
</p>

<p>
Kopiere zuerst deine Konfiguration und das benötigte Zertifikat auf den Router
und stelle die Uhrzeit korrekt ein (ansonsten wird das Verifizieren das
Zertifikats vermutlich fehlschlagen, obwohl das korrekte Zertifikat vorhanden
ist):
</p>

<pre>
x200 $ scp /etc/vpnc/uni.conf root@192.168.2.1:/etc/vpnc/default.conf
x200 $ scp /etc/ssl/certs/rootcert.pem root@192.168.2.1:/etc/ssl/certs/
OpenWrt # rdate time.fu-berlin.de
</pre>

<p>
Wenn du nur vpnc konfigurieren wolltest, bist du hiermit fertig. Mit vpnc
kannst du vpnc starten. Zum automatischen Starten kannst du mein Initscript
etwas weiter unten verwenden und vorher die route-Befehle entfernen.
</p>

<h2>openvpn im vpnc</h2>

<p>
vpnc hat (der Original-Cisco-VPN-Client hat das auch, habe ich gelesen) die
Eigenschaft, dass er eine Default-Route mit Gateway 0.0.0.0 auf das Interface
tun0 legt. Dadurch denkt OpenVPN aber, dass es das Gatway nicht auslesen könne
und weigert sich, die Routen entsprechend zu ändern. Abhilfe schafft folgender
Workaround: Man entfernt die Route von vpnc, erstellt manuell eine Host-Route
zu dem OpenVPN-Server und gibt in der OpenVPN-Config an, dass OpenVPN lediglich
die Default-Route ergänzen soll (statt 123.456.789.000 muss natürlich die IP
deines OpenVPN-Servers verwendet werden):
</p>

<pre>
OpenWrt # route del default gw 0.0.0.0
OpenWrt # route add -host 123.456.789.000 dev tun0           
</pre>

<p>
In /etc/config/openvpn entfernt man dann die redirect_gateway-Direktive und
fügt folgende hinzu:
</p>

<pre>
        option route "0.0.0.0 0.0.0.0"
</pre>

<h2>Initscript</h2>

<p>
Damit die Routen automatisch beim Starten korrekt gesetzt werden, habe ich mir
folgendes Initscript (/etc/init.d/vpnc) geschrieben:
</p>

<pre>
#!/bin/sh /etc/rc.common
# Copyright (C) 2008 Michael Stapelberg
# This is free software, public domain.
# $Id$

START=90

start() {
	rdate time.fu-berlin.de
	/usr/sbin/vpnc | logger || {
		logger "Could not start vpnc!"
		exit 1
	}
	route del default gw 0.0.0.0
	route add -host 123.456.789.000 dev tun0
}

stop() {
	killall vpnc
}

restart() {
	stop; sleep 1; start
}
</pre>

<p>
Installiert wird es via:
</p>

<pre>
OpenWrt # cd /etc/rc.d
OpenWrt # ln -s ../init.d/vpnc S90vpnc
</pre>

<h2>Fehlersuche</h2>

<p>
Sollte etwas schief laufen, schau dir auf jeden fall bei jedem Schritt die
Ausgabe von route -n und ifconfig an. Auch logread hilft.
</p>

<h2>Fazit</h2>

<p>
An einigen Stellen könnte das Setup noch schöner sein. Mich stört insbesondere
die Lizenz-Problematik von vpnc/OpenSSL, was der Grund ist, dass man selbst
kompilieren muss. Aber auch die OpenWrt-Firewall könnte einem einen Tick mehr
unter die Arme greifen. Zu guter letzt bleibt natürlich die Geschichte mit der
Default-Route, das ist natürlich auch unschön.
</p>

<p>
Nichtsdestotrotz läuft die Konfiguration nach der ersten Einrichtung
einwandfrei und stabil. 
</p>
