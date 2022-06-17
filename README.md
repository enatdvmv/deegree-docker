#

deegree in Docker-Containern
============================

## Inhalt
* [Einleitung](#einleitung)
* [deegree Docker Einführung](#deegree-docker-einführung)
* [deegree Docker Dev-Umgebung](#deegree-docker-dev-umgebung)
* [deegree Docker Prod-Umgebung](#deegree-docker-prod-umgebung)
* [Ausblick](#ausblick)


## Einleitung
Mit diesem Repository wollen wir erste Erfahrungen mit deegree WebServices in Docker Container sammeln. Es enthält Docker spezifische Dateien *(Dockerfile, docker-compose.yml)* für das Dev-System sowie zur Simulation eines Prod-Systems. Der deegree Workspace enthält einen View- und einen Download-Service zum INSPIRE Thema GovernmentalService.

Bevor wir loslegen zunächst eine kurze Einführung in die Thematik.


## deegree Docker Einführung
Zur Installation von Docker sei auf [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/) sowie [Install Docker Compose](https://docs.docker.com/compose/install/) verwiesen.

Soweit noch nicht erfolgt, sind die Rechte für die Group Docker zu setzen *(anschließend Session neu starten)*.
```
sudo -i
usermod -aG docker myUserName
```

Kommen wir jetzt zu deegree. Im einfachsten Fall laden wir uns das [Docker Image](https://hub.docker.com/r/deegree/deegree3-docker/) der letzten deegree Version vom Docker-Hub und starten einen Docker Container. Mit *-p* leiten wir den public Port 8081 zum private Port 8080 innerhalb des Containers um.
```
docker pull deegree/deegree3-docker:latest
docker run --name deegree -d -p 8081:8080 deegree/deegree3-docker
```
Auf die deegree Webservices Console können wir nun über die URL http://localhost:8081/deegree-webservices/ zugreifen. Sämtliche Konfigurations-Dateien befinden sich i.d.F. innerhalb des Containers und gehen mit dem Löschen des Containers verloren. Ein Zugriff auf diese Dateien ist möglich, indem eine Shell innerhalb des Containers gestartet wird. Zuvor werfen wir noch einen Blick auf die Container und Images auf dem Server.
```
docker ps -a
docker image ls

docker exec containerId ls -l  /root/.deegree

docker exec -it containerId  '/bin/bash'
cd /root/.deegree
ls -l
exit
```
Und ein Blick auf die Log-Files.
```
docker logs containerId --tail 200
docker logs containerId --follow
```
Die Daten, hier die deegree Konfigurations-Dateien, wollen wir natürlich nicht mit jedem Container neu anlegen. Deshalb legen wir uns einen Ordner an und führen darauf ein volume mount durch.
```
docker stop deegree
docker rm deegree

cd
mkdir deegree/workspaces
docker run -d --name deegree -v ~/deegree/workspaces/:/root/.deegree -p 8081:8080 deegree/deegree3-docker
```
Bisher haben wir aber noch keinen Workspace und damit keine WebServices angelegt. Dazu kommen wie jetzt im nächsten Kapitel.


## deegree Docker Dev-Umgebung
Auf dem Dev-System klonen wir uns das Repository und mounten es **(bind mount)** an einen Container. Damit können wir die deegree Dienste ganz normal in unserem Host-Dateisystem bearbeiten.

Alles was wir benötigen finden wir in diesem Git-Repository. Klonen wir das Git-Repository in unser Homeverzeichnis.
```
cd
git clone https://github.com/enatdvmv/deegree-docker.git
cd deegree-docker
```
Und überschreiben das [Connection-File](workspaces/deegree_workspace_inspire/jdbc/db.xml) mit den Dev-Einstellungen *(Host, User, Password )*. Diese Änderungen dürfen natürlich nicht zurück in das Remote Git-Repository.

Das [Dockerfile](docker_dev/Dockerfile) unter *docker_dev* ist komplett. Im [docker-compose.yml](docker_dev/docker-compose.yml) passen wird unter *volumes* noch den Pfad zu unserem lokalen Workspace an.

Über *docker-compose* erstellen wir nun das Image und starten einen Container.
```
cd
cd deegree-docker
docker-compose -f docker_dev/docker-compose.yml up --build -d deegree
```
Alternativ kann dies auch separat über Docker-Kommandos erfolgen oder eine angepasste *docker-compose.yml*.
```
cd docker_dev
docker image build -t deegree-inspire:dev ./
docker run -d --name deegree-inspire -v ~/deegree-docker/workspaces/:/root/.deegree -p 8084:8080 deegree-inspire:dev
```

Die deegree Webservices Console könnten wir nun über http://host:8084/deegree-webservices/ erreichen.

Änderungen an den Konfigurations-Dateien pushen wir zurück in das Git-Repository *(git status, git add, git commit, git push)*. Natürlich nicht die angepassten Connection-Files.


## deegree Docker Prod-Umgebung
In diesem Kapitel wollen wir den Ablauf auf einem produktiven System simulieren. Auch hier klonen wir zunächst das Git-Repository auf unseren Host und passen die Connection-Files an. Wir werden i.d.F. aber kein *bind mount* vornehmen, sondern zwei andere Varianten ausprobieren. Im ersten Fall wird der deegree Workspace in den Container mit verpackt und im zweiten Fall verwenden wir ein Docker *volume mount*.

In der Produktion müssen die 3 Dateien unter [prod](prod) an der richtigen Stelle platziert werden. Das passiert alles im Dockerfile. Die Datei *main.xml* wird dabei in den deegree Workspace kopiert, während die *rewrite.config* und die *context.xml* im Tomcat zu platzieren sind. Im Dockerfile wird das war-File deshalb zunächst unter */tmp* gespeichert, entpackt und um die beiden Dateien ergänzt. Dann wird der Ordner *deegree-webservices* in den Tomcat verschoben. Einziger Nachteil an diesem Umweg ist der, dass das Image ca. 200 MB größer wird. Der deegree Workspace wird beim Image-Bau aus dem *Docker Context* kopiert, könnte alternativ im Dockerfile auch vom Remote Git-Repository gezogen werden.

### Fall 1: Workspace im Container
[Dockerfile](docker_prod_inside/Dockerfile) und [docker-compose.yml](docker_prod_inside/docker-compose.yml) unter *docker_prod_inside* sind komplett.

Über *docker-compose* erstellen wir nun das Image und starten einen Container.
```
cd
cd degree-docker
docker-compose -f docker_prod_inside/docker-compose.yml up --build -d deegree
```

Die deegree Webservices Console könnten wir nun über http://host:8084/deegree-webservices/ erreichen.

Nachteilig an dieser Vorgehensweise ist, dass bei jeder Änderung an den deegree Diensten ein neues Image gebaut werden muss.


### Fall 2: Volume mount
[Dockerfile](docker_prod_volume/Dockerfile) und [docker-compose.yml](docker_prod_volume/docker-compose.yml) unter *docker_prod_volume* sind komplett.

Bei diesem Ansatz erstellen wir zunächst ein Docker Volume. Dieses Volume wird von Docker gemanagt. Wir sollten also nicht direkt Daten in den physischen Speicherort schreiben sondern dies nur über einen Container tun.
```
docker volume create vol_inspire
```
Image erstellen und Container starten.
```
cd
cd degree-docker
docker-compose -f docker_prod_volume/docker-compose.yml up --build -d deegree
```
Alternativ, wenn Image schon vorhanden, nur den Container starten.
```
docker run -d --name deegree-inspire -v vol_inspire:/root/.deegree -p 8084:8080 deegree-inspire:prod
```
Jetzt kopieren wir den deegree Workspace über den Container in das gemountete Docker Volume.
```
docker cp /home/user/degree-docker/workspaces/deegree_workspace_inspire/ deegree-inspire:/root/.deegree/
docker cp /home/user/degree-docker/workspaces/webapps.properties deegree-inspire:/root/.deegree/
docker cp /home/user/degree-docker/prod/main.xml deegree-inspire:/root/.deegree/deegree_workspace_inspire/services/main.xml
```
Und starten den Container neu.
```
docker restart deegree-inspire
```

Die deegree Webservices Console könnten wir nun über http://host:8084/deegree-webservices/ erreichen.

Der Vorteil gegenüber Variant 1 ist der, dass nicht bei jeder Änderung an den deegree Diensten ein neues Image gebaut werden muss. Dafür muss man sich Gedanken über eine Aktualisierungsstrategie des Docker Volume machen.


## Ausblick
In diesem Repository haben wir erste Erfahrungen gesammelt wie wir deegree Webservices in Docker Containern betreiben können. Produktiv könnte das Docker-Image über eine Jenkins-Pipeline gebaut und in einem Artifactory gespeichert werden. Und die Container in einem Kubernetes-Cluster ausgeführt werden, wie im Repository [deegree-kubernates](https://github.com/enatdvmv/deegree-docker) beschrieben.
