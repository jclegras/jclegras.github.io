
+++
author = "Jean-Charles Legras"
title =  "Le user namespace dans Docker"
date = "2020-06-01"
tags = ["docker", "namespace", "linux", "devops"]
+++

## Avant l'arrivée du user namespace dans Docker

Lancement d'un conteneur Jenkins

{{< highlight bash >}}
$ docker run --name jenkins --d jenkins/jenkins:lts
{{< /highlight >}}

A quel utilisateur appartient le process à l'intérieur du conteneur ?

{{< highlight bash >}}
$ ps -xao uid,cmd | grep jenkins

UID  CMD
1000 java [...] -jar /usr/share/jenkins/jenkins.war
{{< /highlight >}}

L'UID est fixé à **jclegras**, mon nom de login.

{{< highlight bash >}}
$ grep 1000 /etc/passwd

jclegras:x:1000:1000:jclegras,,,:/home/jclegras:/bin/bash
{{< /highlight >}}

Essayons maintenant de lancer le conteneur avec l'utilisateur root (option *-u*)

{{< highlight bash >}}
$ docker run -u root --name jenkins_root –d jenkins/jenkins:lts
{{< /highlight >}}

Encore une fois, regardons le résultat obtenu :

{{< highlight bash >}}
$ ps -xao uid,cmd | grep jenkins 

UID CMD
0   java [...] -jar /usr/share/jenkins/jenkins.war
{{< /highlight >}}

Cette fois-ci le process appartient bien à l'utilisateur **root**.

Une question s’impose alors d’elle-même: comment le docker engine 
fait-il pour faire la correspondance entre les users de mes conteneurs,
et les users de la machine hôte, ou autrement dit, comment se fait-il 
que le premier process appartienne à jclegras et le deuxième à root ?

### Correspondance des users

Pour le premier exemple, il faut regarder d’un peu plus près le [Dockerfile de Jenkins](https://github.com/jenkinsci/docker/blob/master/Dockerfile).

Les lignes qui nous intéressent sont :

{{< highlight docker >}}
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
[...]
# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN mkdir -p $JENKINS_HOME \
  && chown ${uid}:${gid} $JENKINS_HOME \
  && groupadd -g ${gid} ${group} \
  && useradd -d "$JENKINS_HOME" -u ${uid} -g ${gid} -m -s /bin/bash ${user}
[...]
USER ${user}  
{{< /highlight >}}

Et voici ce que ça raconte : on crée un user et un group jenkins avec un uid
et un gid égales à 1000 puis on dit que, par défaut, en l’absence de user 
explicite (l’option -u que nous avons fourni plus haut), 
le process à l'intérieur du conteneur s’exécutera avec les droits de 
l’utilisateur **jenkins**.

Mais alors, comment se fait-il qu’un ps m’affirme que le conteneur 
appartient à *jclegras* ?

A dire vrai, avant la [1.10](https://www.docker.com/blog/docker-engine-1-10-security/), Docker mappait exclusivement les users
(ou plus exactement les uid/gid) selon l’hôte:

{{< highlight bash >}}
$ id

jclegras uid=1000(jclegras) gid=1000(jclegras)
{{< /highlight >}}

C’est ainsi qu’on aperçoit le côté un peu tricky de la chose : 
à l’intérieur du conteneur, par défaut, je suis le user jenkins mais
à l’extérieur, je suis mappé sur un user qui **correspond à** jenkins,
c’est-à-dire à quelqu’un possédant ses identifiants :

````
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
````

Autrement dit, l’utilisateur jenkins est en réalité l’utilisateur 
jclegras sur la machine hôte !

Par extension, lorsque j’ai lancé mon deuxième conteneur en tant que 
root, l’utilisateur root du conteneur était en réalité l’utilisateur 
correspondant aux identifiants (uid/gid) de root :

```
uid=0(root) gid=0(root) groups=0(root)
```

Et je ne vous apprends rien en disant que ce sont exactement les même 
identifiants que le user root de la machine hôte.

Le root du conteneur est donc mappé sur le root de l’hôte !

**C’est pourquoi, dans les bonnes pratiques de Docker, il est conseillé 
de lancer vos conteneurs sous un autre utilisateur que root** 
(ce qu’ont bien compris les concepteurs de l’image Jenkins). 

### Le danger de la correspondance des users par défaut

Imaginons que je suive donc les best practices et que je lance le
conteneur Jenkins avec l'utilisateur jenkins comme dans le premier
exemple.

{{< highlight bash >}}
docker run -name jenkins -d jenkins
{{< /highlight >}}

Rien ne m’empêche par la suite de rentrer à l’intérieur avec 
l’utilisateur root (il y a toujours un utilisateur root disponible dans les conteneurs) :

{{< highlight bash >}}
docker exec -u root -it jenkins bash
{{< /highlight >}}

Vous me direz que ce n’est rien, après tout, tout ce que je peux faire
dans le conteneur reste dans le conteneur, et c’est le but d’un tel outil. 

**Que se passe-t-il lorsque nous commençons à monter des volumes ?**

{{< highlight bash >}}
docker run -v $(pwd)/jenkins_home:/var/jenkins_home -d jenkins
{{< /highlight >}}

Ou plus généralement :

{{< highlight bash >}}
docker run -v /répertoire_critique:/opt/dest -d jenkins
{{< /highlight >}}

Vous imaginez dès lors les conséquences d’une prise de contrôle de la 
part d’un utilisateur malveillant sur, par exemple, une application web
(ou pire une base de données) ayant au moins un volume partagé : 
on peut tout supprimer. Et si mon “répertoire_critique” contient 
d’autres fichiers que ceux exigés par l’application web, je peux aussi
les modifier et mettre en péril, dans le pire des cas, **l’ensemble
de mon système hôte** (pas génial si vous êtes en prod…).

L’idéal serait d’avoir un système permettant de cloisonner
ces utilisateurs dans le conteneur afin que, même si je suis root à 
l’intérieur du conteneur (et que je puisse donc faire tout ce que bon me semble),
je ne puisse pas mettre en danger le système hôte.

De la même façon, même si j’utilise un utilisateur custom dans mon conteneur,
je voudrais ne pas correspondre à un user random sur le système hôte 
(même si dans les faits, pour éviter que jenkins corresponde à jclegras, 
on fait en sorte d’attribuer des uid et des gid plus élevés afin de faire 
correspondre jenkins au “vrai” user jenkins sur l’hôte créé pour l’occasion).

Cependant, et vous vous en doutez, il faut aller bidouiller les différents 
Dockerfiles de toutes ces images officielles et procéder nous-même aux modifications qui vont bien,
 tout en répercutant sur le système hôte, et en espérant que mon user 
jenkins ne se voit pas, un beau jour, attribuer des privilèges qu’il ne mérite pas
(tiens, pourquoi jenkins est dans les sudoers d’un seul coup ?)

## Le user namespace dans Docker

Bref, c’est ici que le user namespace finalement intégré au Docker engine
(bien que présent depuis quelque temps dans le noyau linux) arrive à la rescousse.

Succinctement, cette feature permet de mapper (dynamiquement) mes users comme suit :

```
root -> un user_non_privilégié
jenkins -> un autre_user_non_privilégié ...
```

En pratique, voici comment le mettre en application : https://docs.docker.com/engine/security/userns-remap/

Je ne sais pas si c’est très clair, mais essayons de “singer” ce tuto en partant d’un besoin réel :

- Je souhaite que tous mes conteneurs s’exécutant sous root soient mappés sur le user et le groupe *dockremap* ;
- Je souhaite également que les conteneurs utilisant un utilisateur personnalisé d’UID 1000
et appartenant à un groupe de GID 1000 soient mappés sur l’utilisateur et le groupe *dockremap-user* ;
- Finalement, je souhaite que les users/groupes dockremap et dockremap-user possèdent le moins de droits possible.

Commençons par l’étape de création des utilisateurs et des groupes :

{{< highlight bash >}}
groupadd -g 500000 dockremap && \
 groupadd -g 501000 dockremap-user && \
 useradd -u 500000 -g dockremap -s /bin/false dockremap && \
 useradd -u 501000 -g dockremap-user -s /bin/false dockremap-user
{{< /highlight >}}

Ensuite, ajoutons une liste d’utilisateurs “subalternes” et une liste 
de groupes “subalternes” en utilisant comme point de référence, resp.,
l’utilisateur et le groupe dockremap :

{{< highlight bash >}}
echo "dockremap:500000:65536" >> /etc/subuid && \
 echo "dockremap:500000:65536" >> /etc/subgid
{{< /highlight >}}

Finalement, ajoutons l’option magique **userns-remap** pour activer le user namespace dans les conteneurs
(passons par le fichier de conf du démon docker pour l’occasion) :

{{< highlight json >}}
{ "userns-remap": "default" }
{{< /highlight >}}

On remarquera que default est un alias de *dockremap* :
en fait, quand on met *default*, le démon docker crée (si ce n’est pas déjà fait),
un utilisateur et un groupe dockremap (avec un “numerical subordinate user/group ID” 
positionné à un offset correspondant à ce qui était déjà présent dans les fichiers 
*/etc/subuid* et */etc/subgid* lors de la création).

On aurait donc pu se passer des étapes précédentes, 
mais on y gagne au moins deux choses : nous avons choisi nos propres ID (500000 dans les deux cas) et on sait le faire manuellement.

Pour finir, redémarrons le démon docker :

{{< highlight bash >}}
systemctl daemon-reload && systemctl restart docker
{{< /highlight >}}

Mission accomplie, nos users sont bien mappés comme nous le voulions !

En fait, en s’appuyant sur les utilisateur et groupe de référence dockremap
fixés tout deux à 500000 et sur la configuration des “subordinate files“ (les */etc/sub\**),
le mapping sera comme suit:

| ID machine hôte \|| ID conteneur | 
|---------------- |:------------------ | 
| 500000 | 0 | (root) | ... | ... | 
| 501000 | 1000 | (jenkins, ...) 
| ... | ... | 
| 565536 | 65536 |


On peut donc dès lors se contenter d’utiliser root dans tous nos conteneurs
puisque ce dernier sera automatiquement converti en dockremap sur l’hôte,
tout en gardant les avantages d’être root, et donc tout puissant, à l’intérieur de nos conteneurs.

## Bonus

Cette section est créée en guise d’avertissement.

Avant, lorsque nous étions root dans nos conteneurs, on pouvait monter très simplement nos volumes partagés
(j’entends par là : entre le système hôte et le conteneur cible) :

{{< highlight bash >}}
docker run -v /mon_repertoire:/mon_repertoire -d [mon_image]
{{< /highlight >}}

Et je pouvais faire ce que je voulais sur les fichiers contenus dans 
**mon_repertoire** (les modifier, les supprimer…). 

En effet, rappelez-vous, root du conteneur était mappé sur root de l’hôte
et par conséquent, il avait tous les droits. 

Comment faire dès lors pour pouvoir modifier les fichiers contenus dans
le répertoire lorsque j’active le user namespace et que mon user root est mappé sur dockremap ?

C'est très simple, il vous faudra attribuer les droits qui vont bien aux fichiers en question :

{{< highlight bash >}}
chown -R dockremap:dockremap /mon_repertoire/
{{< /highlight >}}

Voici une démo qui montre que ça fonctionne pas trop mal :

{{< highlight bash >}}
$ ls -l test_docker/ 

-rw-r--r-- 1 dockremap dockremap 0 sept. 4 01:01 dummy_dockremap 
-rw-rw-r-- 1 dockremap-user dockremap-user 0 sept. 4 01:00 dummy_dockremap-user 
-rw-rw-r-- 1 jclegras jclegras 12 sept. 3 16:29 dummy_jclegras 
-rw-rw-r-- 1 jenkins jenkins 12 sept. 2 00:59 dummy_jenkins 
-rw-rw-r-- 1 root root 0 sept. 2 00:28 dummy_root 
{{< /highlight >}}

{{< highlight bash >}}
$ docker run -v $(pwd)/test_docker:/var/jenkins_home/test_docker -d jenkins 
070003aa39b81a53ffe6b01f44986922058f1585d8da285f981cf3b673b9d734 

$ docker exec -it 070 bash 

jenkins@070003aa39b8:/$ ls -l /var/jenkins_home/test_docker/ 
-rw-r--r-- 1 root root 0 Sep 3 23:01 dummy_dockremap 
-rw-rw-r-- 1 jenkins jenkins 0 Sep 3 23:00 dummy_dockremap-user 
-rw-rw-r-- 1 nobody nogroup 12 Sep 3 14:29 dummy_jclegras 
-rw-rw-r-- 1 nobody nogroup 12 Sep 1 22:59 dummy_jenkins 
-rw-rw-r-- 1 nobody nogroup 0 Sep 1 22:28 dummy_root 

jenkins@070003aa39b8:/$ whoami 
jenkins 

jenkins@070003aa39b8:/$ id 
jenkins uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins) 

jenkins@070003aa39b8:/$ echo "hello world">> /var/jenkins_home/test_docker/dummy_dockremap 
bash: /var/jenkins_home/test_docker/dummy_dockremap: Permission denied 

jenkins@070003aa39b8:/$ echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap-user 

jenkins@070003aa39b8:/$ echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap-jclegras 
bash: /var/jenkins_home/test_docker/dummy_dockremap-jclegras: Permission denied 

jenkins@070003aa39b8:/$ echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap-jenkins 
bash: /var/jenkins_home/test_docker/dummy_dockremap-jenkins: Permission denied 

jenkins@070003aa39b8:/$ echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap-root 
bash: /var/jenkins_home/test_docker/dummy_dockremap-root: Permission denied
{{< /highlight >}}

En utilisant root dans mon conteneur :

{{< highlight bash >}}
$ docker exec -u root -it 070 bash 

root@070003aa39b8:/# ls -l 
/var/jenkins_home/test_docker/ 
-rw-r--r-- 1 root root 0 Sep 3 23:01 dummy_dockremap 
-rw-rw-r-- 1 jenkins jenkins 12 Sep 3 23:07 dummy_dockremap-user 
-rw-rw-r-- 1 nobody nogroup 12 Sep 3 14:29 dummy_jclegras 
-rw-rw-r-- 1 nobody nogroup 12 Sep 1 22:59 dummy_jenkins 
-rw-rw-r-- 1 nobody nogroup 0 Sep 1 22:28 dummy_root 

root@070003aa39b8:/# id `whoami` 
uid=0(root) gid=0(root) groups=0(root) 

root@070003aa39b8:/# echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap 

root@070003aa39b8:/# echo "hello world" >> /var/jenkins_home/test_docker/dummy_dockremap-user 

root@070003aa39b8:/# echo "hello world" >> /var/jenkins_home/test_docker/dummy_jclegras 
bash: /var/jenkins_home/test_docker/dummy_jclegras: Permission denied 

root@070003aa39b8:/# echo "hello world" >> /var/jenkins_home/test_docker/dummy_jenkins 
bash: /var/jenkins_home/test_docker/dummy_root: Permission denied
{{< /highlight >}}

Et enfin, en regardant la table des processus (avec le conteneur lancé avec le user par défaut, jenkins) :

{{< highlight bash >}}
$ ps aux | grep dock 
dockrem+ 16807 0.0 0.0 1112 4 ? Ss 01:03 0:00 /bin/tini -- /usr/local/bin/jenkins.sh 
dockrem+ 16825 2.3 5.2 3073244 213492 ? Sl 01:03 0:17 java -jar /usr/share/jenkins/jenkins.war 

$ ps -U dockremap-user 
PID TTY TIME     CMD 
16807 ? 00:00:00 tini
16825 ? 00:00:17 java
{{< /highlight >}}

## Ressources

Le user namespace en long, en large et en travers :

- [Configuration du user namespace sous Docker](https://docs.docker.com/engine/security/userns-remap)
- [Article LWN.net sur le user namespace](https://lwn.net/Articles/532593/)
- [Restrictions inhérentes à l’activation du user namespace](https://docs.docker.com/engine/security/userns-remap/#user-namespace-known-limitations)

Les namespaces du kernel linux, fondations des conteneurs (mais pas que) :

- [Les namespaces en général sous Linux](https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html)
- [Un autre article sur les namespaces du kernel linux](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)

Bonnes pratiques Docker :

- [10 bonnes pratiques](https://w3blog.fr/2016/02/23/docker-securite-10-bonnes-pratiques/)

J'espère que ce billet vous a plu et qu'il vous donne envie de creuser 
le sujet plus en profondeur (Docker, les namespaces linux, les cgroups, les fondations des conteneurs...).

Merci pour votre lecture.