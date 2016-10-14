# Node.js und MySQL mit Vagrant

Für die Entwicklung von *Node.js*-Applikationen habe ich mir eine Laufzeitumgebung mit *Vagrant* eingerichtet, um nicht ständig mein System beschmutzen zu müssen. Die Vorteile liegen klar auf der Hand. Eine klar abgegrenzte virtuelle Maschine sorgt für die Bereitstellung meiner Systemumgebung, während ich mit meinen lokalen Werkzeugen bequem althergebracht an meinem Code schrauben kann.

Die Einrichtung hat mich einige Mühe und die Hilfe verschiedener Quellen gekostet, letztendlich bin ich jedoch am Ziel angelangt, zumindest eine erste Basis für weitere Entwicklungen zu legen.

## Installation der vorrausgesetzten Tools

Folgende Tools werden benötigt und sollten vor Beginn installiert werden:

- *Vagrant*
- *Node.js*
- *npm*
- Texteditor

## Initialisierung des *Vagrant*-Projekts

Zuerst lege ich ein Projektverzeichnis an und initialisiere dort mein ```Vagrantfile```.

[source,bash]
----
mkdir project
cd project
vagrant init
----

In der ```Vagrantfile``` sind ein paar grundlegende Dinge vorhanden, diese ersetze ich jedoch durch folgende Einträge.

[source,ruby]
----
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
  config.vm.network :forwarded_port, guest: 3000, host: 3000
end
----

Wie man sieht, wähle ich als Grundlage Ubuntu aus, auf dem ich ein Vorbereitunsscript ausführe (dazu mehr im nächsten Schritt). Letztendlich sehe ich künftig Port 3000 für die Kommunikation mit meiner Applikation vor.

## Zusammenstellen der bootstrap.sh

In meiner virtuellen Maschine möchte ich folgende Komponenten nutzen.

- *Node.js*
- *npm*
- *GruntJS*
- Einen *MySQL*-Server

Bei der Intialisierung der virtuellen Maschine führt *Vagrant* die von mir anzulegende ```bootstrap.sh``` aus. Als erstes lege ich die Datei an und schreibe Grundlegendes hinein.

[source,bash]
----
#!/usr/bin/env bash
apt-get update
----

### *Node.js*, *npm* und *GruntJS*

Als nächstes wird *Node.js* installiert. Hierfür wird ```curl``` benötigt. *npm* wird zusammen mit *Node.js* installiert.

[source,bash]
----
apt-get install -y curl
curl -sL https://deb.nodesource.com/setup_6.x | bash -
apt-get install -y nodejs
----

Da ich meine Projekte später mit *GruntJS* bauen möchte, installiere ich auch dieses gleich noch global.

[source,bash]
----
npm install -g grunt-cli
----

### *MySQL*-Server

Als nächstes findet die Installation des *MySQL*-Servers statt. Diese erfordert einen kleinen Trick, da im Zuge der Installation normalerweise die Eingabe des root-Passworts gefordert wird. Ich will jedoch den Vorgang "headless", also ohne weitere Interaktion mit dem Benutzer vonstatten gehen lassen. Darum konfiguriere ich mein System zuerst mit den passenden Eingaben vor. Diesen Kniff habe ich http://www.thisprogrammingthing.com/2013/getting-started-with-vagrant/[hier] gefunden.

[source,bash]
----
debconf-set-selections <<< 'mysql-server-5.5 mysql-server/root_password password rootpass'

debconf-set-selections <<< 'mysql-server-5.5 mysql-server/root_password_again password rootpass'
----

Somit wird im nächsten Schritt der *MySQL*-Server mit dem Benutzer ```root``` und dem Passwort ```rootpass``` installiert.

[source,bash]
----
apt-get install -y mysql-server-5.5
----

### Einrichten der Datenbank

Zuletzt initialisiere ich den Datenbankserver mit einem Benutzer für meine Applikation sowie einer dafür vorgesehenen Datenbank.

[source,bash]
----
if [ ! -f /var/log/databasesetup ]; then
	echo "CREATE USER 'nodeuser'@'localhost' IDENTIFIED BY 'nodepass'" | mysql -uroot -prootpass
	echo "CREATE DATABASE node" | mysql -uroot -prootpass
	echo "GRANT ALL ON node.* TO 'nodeuser'@'localhost'" | mysql -uroot -prootpass
	echo "flush privileges" | mysql -uroot -prootpass

	touch /var/log/databasesetup

	if [ -f /vagrant/data/init.sql ]; then
		mysql -uroot -prootpass node < /vagrant/data/init.sql
	fi
fi
----

Als Bonus habe ich die Möglichkeit, in meinem Projektverzeichnis im Unterordner ```/data``` eine Datei namens ```init.sql``` abzulegen, über die ich meine Datenbank letztendlich mit Tabellen und Datensätzen vorkonfigurieren kann.

## Hochfahren und Testen

Meine virtuelle Maschine starte ich nun mit dem Befehl ```vagrant up```. Wenn ich mich nun per ```vagrant ssh``` auf die Maschine aufschalte, kann ich dort Aktionen auf meiner Datenbank ausführen.

Das Verzeichnis ```/vagrant``` beinhaltet alle Dateien meines Arbeitsverzeichnisses, so dass ich meine Applikation nun normal auf dem Hostsystem entwickeln und in meiner virtuellen Maschine hochfahren und testen kann.

Welche Vorteile habe ich nun durch diese Umgebung erhalten?

- Keine Installation von Node.js in meinem eigentlichen Betriebssystem erforderlich (lediglich npm ist für die Installation von Abhängigkeiten direkt vom Hostsystem aus nötig)
- Keine Installation eines MySQL-Servers erforderlich
- Zurücksetzen der Laufzeitumgebung ist jederzeit mit ```vagrant destroy``` und ```vagrant up``` möglich

Die komplette ```bootstrap.sh``` findest Du https://gist.github.com/cbeulke/3a3611d6e06a457e081332e6ff0c4a56[hier].