+++
author = "Jean-Charles Legras"
title =  "Permissions spéciales"
date = "2020-09-03"
draft = false
tags = ["setuid", "setgid", "sticky bit", "special permissions", "restricted deletion flag"]
+++

## Les permissions spéciales sous Linux

### Résumé

Je vous présente ici les permissions dites spéciales qu'on utilise moins souvent que les permissions standards.

Pour commencer, nous allons parler **setuid** et **setgid** qu'on peut setter sur les fichiers avec un petit programme Golang pour voir les choses.

Ensuite, nous examinerons le **setgid** mais cette fois-ci sur des répertoires.

Finalement, on s'intéressera au **sticky bit** sur les répertoires.

Pour conclure, on jettera un oeil à une fonctionnalité de Kubernetes qui fait usage du **setgid**.

### Permissions spéciales sur un fichier

#### setuid/setgid

__Besoin__ : exécuter un fichier exécutable avec les privilèges du propriétaire/groupe resp. du fichier

**main.go**
{{< highlight golang >}}
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Printf("Real UID\t= %d\n", os.Getuid())
	fmt.Printf("Effective UID\t= %d\n", os.Geteuid())
	fmt.Printf("Real GID\t= %d\n", os.Getgid())
	fmt.Printf("Effective GID\t= %d\n", os.Getegid())
}
{{< /highlight >}}

- Le *Real UID/GID* correspond resp. à l'UID/GID de l'utilisateur courant
- Le *Effective UID/GID* correspond resp. à l'UID/GID adopté
- L'octet de plus haut niveau est setté à 2 pour setgid et 4 pour setuid

{{< highlight bash >}}
$ id
uid=1000(jclegras) gid=1000(jclegras)

$ getent group docker
docker:x:999:jclegras

$ stat -c "%a %A %U:%G" main
775 -rwxrwxr-x root:docker

$ ./main
Real UID        = 1000
Effective UID   = 1000
Real GID        = 1000
Effective GID   = 1000
# Par défaut, les effective UID/GID sont fixées à mes UID/GID utilisateur

$ sudo chmod u+s main && stat -c "%a %A %U:%G" main
4775 -rwsrwxr-x root:docker

$ ./main
Real UID        = 1000
Effective UID   = 0
Real GID        = 1000
Effective GID   = 1000
# Avec setuid, l'UID adopté par le processus vaut celui du propriétaire du fichier exécutable

$ sudo chmod g+s main && stat -c "%a %A %U:%G" main
6775 -rwsrwsr-x root:docker

$ ./main
Real UID        = 1000
Effective UID   = 0
Real GID        = 1000
Effective GID   = 999
# Avec setgid, le GID adopté par le processus vaut celui du groupe du fichier exécutable
# Le setuid étant encore fixé, l'effective UID vaut toujours celui de root

{{< /highlight >}}

Pour commencer, on constate que lorsque les bits setuid sont marqués, l'*effective UID* est fixé à *root*, propriétaire du fichier.

De même, lorsque les bits setgid sont marqués, l'*effective GID* est fixé à *docker*, groupe du fichier.

Par ailleurs, les bits setuid (4) et setgid (2) étant marqués, l'octet de plus haut niveau vaut bien 4 + 2 = 6.

Finalement, le processus représentant l'exécution du fichier *main.go* vaut bien jclegras:jclegras (1000:1000) lorsque les permissions
spéciales ne sont pas fixées du tout.

__Besoin__ : lister les fichiers avec setuid qui conduirait à un effective UID égale à root

Voici un exemple conduit sur le répertoire */bin*:
{{< highlight bash >}}
$ find /bin -user root -perm -4000 -exec ls -ldb {} \;

-rwsr-xr-x 1 root root 26696 mars   5  2020 /bin/umount
-rwsr-xr-x 1 root root 30800 août  11  2016 /bin/fusermount
-rwsr-xr-x 1 root root 64424 juin  28  2019 /bin/ping
-rwsr-xr-x 1 root root 43088 mars   5  2020 /bin/mount
-rwsr-xr-x 1 root root 44664 mars  22  2019 /bin/su
{{< /highlight >}}

On voit bien par exemple que la commande ping impose des privilèges qu'un utilisateur courant n'est pas censé posséder. Les bits setuid sont donc fixés sur la commande *ping*.

En effet, l'utilitaire *ping* doit pouvoir créer des sockets ICMP qui connaissent des restrictions dans les kernels Linux: [man pages icmp(7)](http://man7.org/linux/man-pages/man7/icmp.7.html)

       ping_group_range (two integers; default: see below; since Linux
       2.6.39)
              Range of the group IDs (minimum and maximum group IDs,
              inclusive) that are allowed to create ICMP Echo sockets.  The
              default is "1 0", which means no group is allowed to create
              ICMP Echo sockets.

### Permissions spéciales sur un répertoire

#### setgid

__Besoin__ : je veux partager un répertoire entre plusieurs personnes pour en faire un espace de travail sans avoir à changer mon groupe (GID) primaire

{{< highlight bash >}}
$ mkdir special-directory && \
chown jclegras:docker special-directory/ && \
stat -c "%a %A %U:%G" special-directory/

775 drwxrwxr-x jclegras:docker 
# On crée un répertoire en 775 d'UID:GID jclegras:docker

$ touch special-directory/my-file && \
  stat -c "%a %A %U:%G" special-directory/my-file

664 -rw-rw-r-- jclegras:jclegras 
# Le fichier créé a pour GID celui de mon utilisateur courant

$ chmod g+s special-directory/ && \
  stat -c "%a %A %U:%G" special-directory

2775 drwxrwsr-x jclegras:docker 
# On remarque l'octet de haut niveau fixé à 2 (setgid)

$ stat -c "%a %A %U:%G" special-directory/my-file

664 -rw-rw-r-- jclegras:jclegras 
# Les permissions du fichier existant ne bougent pas

$ touch special-directory/my-file2 && \
  stat -c "%a %A %U:%G" special-directory/my-file2

664 -rw-rw-r-- jclegras:docker 
# Le nouveau fichier créé obtient le GID du répertoire dans lequel il est créé

$ touch my-file3 && \
  stat -c "%a %A %U:%G" my-file3

664 -rw-rw-r-- jclegras:jclegras

$ mv -v my-file3 special-directory/ && \
  stat -c "%a %A %U:%G" special-directory/my-file3

664 -rw-rw-r-- jclegras:jclegras
# Les permissions d'un fichier existant déplacé dans le répertoire ne changent pas

$ mkdir special-directory/subdirectory && \
  stat -c "%a %A %U:%G" special-directory/subdirectory/

2775 drwxrwsr-x jclegras:docker
# Le répertoire « hérite » de la permission setgid
{{< /highlight >}}

On observe avec cette suite de commandes ce qui suit :
- Par défaut, un fichier créé depuis un répertoire « classique » (sans setgid par exemple) possède un GID égale au GID primaire de l'utilisateur qui l'a créé ;
- Lorsqu'on set les bits setgid sur le répertoire, le changement de propriétaire ne s'applique pas aux fichiers existants dans ce répertoire ;
- Un fichier créé dans un répertoire avec setgid possède un GID égale au GID du répertoire ;
- L'« héritage » du groupe à un fichier ne s'applique qu'à la création du fichier, déplacer un fichier dans un répertoire avec setgid ne change pas les permissions dudit fichier ;
- Tout répertoire créé dans un répertoire avec un setgid fixé possède également le setgid hérité de son répertoire parent.

__Note__: *setuid* est ignoré sur un répertoire sous Linux.

#### Sticky bit (plus précisément ici : restricted deletion flag)

__Rappel__ : dès lors qu'on possède les droits en écriture sur un répertoire, on peut supprimer les fichiers qu'il contient,
qu'on possède les droits en écriture sur le fichier ou non.

Si on ne possède pas les droits sur le fichier on aura seulement un prompt d'avertissement :

{{< highlight bash >}}
$ mkdir directory && \
  chmod 777 directory && \
  touch directory/root_file && \
  sudo chown -R root:root directory/root_file && \
  stat -c "%a %A %U:%G %n" my-directory

777 drwxrwxrwx jclegras:jclegras directory/

$ stat -c "%a %A %U:%G %n" directory/root_file

664 -rw-rw-r-- root:root directory/root_file

$ rm -v directory/root_file

rm: remove write-protected regular empty file 'directory/root_file'? y
removed 'directory/root_file'
{{< /highlight >}}

__Besoin__ : protéger les fichiers présents dans un répertoire à son seul propriétaire uniquement (suppression/renommage)


{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" mydir

777 drwxrwxrwx jclegras:jclegras mydir
# Perm. en 777, tout le monde peut supprimer les fichiers à l'intérieur

$ chmod o+t mydir && \
  stat -c "%a %A %U:%G %n" mydir

1777 drwxrwxrwt jclegras:jclegras mydir
# Le sticky bit vaut 1

$ touch mydir/myfile && \
  stat -c "%a %A %U:%G %n" mydir/myfile

664 -rw-rw-r-- jclegras:jclegras mydir/myfile 
# Pour la suppression, les droits du fichier ne rentrent pas en compte (@ rappel)

$ sudo su jcbis

$ rm -v mydir/myfile

rm: remove write-protected regular empty file 'mydir/myfile'? y
rm: cannot remove 'mydir/myfile': Operation not permitted
# Suppression impossible alors que les droits du répertoire le permettent
#   On a bien par ex o+rwx, donc les 'Autres' peuvent supprimer les fichiers
#   C'est bien le sticky bit qui impose une restriction plus forte ici (!)

$ mv -v mydir/myfile .
mv: cannot move 'mydir/myfile' to './myfile': Operation not permitted
# Idem pour le renommage de fichier
{{< /highlight >}}

On peut même pousser le vice encore plus loin en settant le sticky bit ET le setgid sur le répertoire :

{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" mydir

3777 drwxrwsrwt jclegras:jcbis mydir
# Chaque fichier créé aura pour groupe 'jcbis'
#  2 pour setgid et 1 pour sticky bit

$ touch mydir/myfile

$ sudo su jcbis

$ stat -c "%a %A %U:%G %n" mydir/myfile

664 -rw-rw-r-- jclegras:jcbis mydir/myfile
# jcbis appartient au groupe 'jcbis', il pourrait supprimer ce fichier

$ rm -v mydir/myfile

rm: cannot remove 'mydir/myfile': Operation not permitted
# Opération non autorisée par le stick bit (!)
{{< /highlight >}}

Lorsque les setgid et sticky bits sont fixés, le sticky bit s'applique toujours.
Le sticky bit empêche là encore suppression et renommage du fichier.
Ce flag combiné au setgid, sur un répertoire, donne un fichier en append-only.
En effet, rien n'empêche *jcbis* de lire et d'écrire dans le fichier.

On peut trouver l'application du sticky bit sur le répertoire */tmp* par exemple :

{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" /tmp
1777 drwxrwxrwt root:root /tmp
{{< /highlight >}}

On peut rechercher des exemples de répertoire avec sticky bit avec l'outil *find* :
{{< highlight bash >}}
$ sudo find / -user root -perm -1000 -type d -exec ls -ldb {} \;
[...]
drwxrwxrwt 2 root root 40 Sep  5 14:23 /var/lib/docker/containers/7e30ae038ba1d1f0e1196ded588a98a309aa380609ccf315e4fbc3adb3be161b/mounts/shm
drwxrwxrwt 2 root root 40 Sep  5 14:23 /var/lib/docker/containers/8839c653425759dd3e81377de3bf799826872c4ba7751dccac44bc1542378831/mounts/shm
drwxrwxrwt 2 root root 40 Sep  5 14:23 /var/lib/docker/containers/b41eae33e85f9d21d597b1d8af79f6b11bcca326d6d5ff60f5e39e6a5f86672b/mounts/shm
drwxrwxrwt 2 root root 40 Sep  5 14:23 /var/lib/docker/containers/1a5313717c13202c1577546b43490775bed7abf411efc5eab8e1a36c6da79f71/mounts/shm
drwxrwxrwt 2 root root 4096 Nov 21  2019 /var/lib/docker/overlay2/e44048381a2ad8d9fd682002fc364e7bb5add8f1d1d310a72cf3cc3992f5e398/diff/tmp
drwxrwxrwt 2 root root 4096 Feb 26  2020 /var/lib/docker/overlay2/6daa7569532907310395eeba2f7a0e03d220865a94d973e6da91a8020110c6c5/diff/tmp
[...]
{{< /highlight >}}

Selon les [man pages Linux](https://man7.org/linux/man-pages/man1/chmod.1.html#RESTRICTED_DELETION_FLAG_OR_STICKY_BIT), on apprend finalement que le terme *sticky bit* est réservé au bit lorsqu'il est positionné sur un fichier.
Sur les nouveaux systèmes, le sticky bit est ignoré lorsqu'il concerne les fichiers.
Lorsque ce bit est posé sur un répertoire, il se nomme *restricted deletion flag*. Ce qui est décrit plus haut montre son fonctionnement.

### Kubernetes

Une feature située au niveau du [PodSecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) utilise le setgid pour le partage de (certains types) de volumes entre plusieurs conteneurs : *spec.fsGroup*.

Voici un YAML d'exemple pour montrer cette feature :

{{< highlight yaml >}}
apiVersion: apps/v1
kind: Deployment
metadata:
   name: hello-world
   labels:
     app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      securityContext:
        fsGroup: 555
      containers:
      - name: first
        image: alpine
        command: ["/bin/sleep", "999999"]
        securityContext:
          runAsUser: 1111
        volumeMounts:
        - name: shared-volume
          mountPath: /volume
      - name: second
        image: alpine
        command: ["/bin/sleep", "999999"]
        securityContext:
          runAsUser: 2222
        volumeMounts:
        - name: shared-volume
          mountPath: /volume
      volumes:
      - name: shared-volume
        emptyDir:
{{< /highlight >}}

Montrons le *setgid* à l'oeuvre avec cette suite de commandes Bash :

{{< highlight bash >}}
$ kubectl exec -it hello-world-bd78cdd6-pm4tq -c first -- sh
# Ou tout autre nom qu'aura pris votre Pod

/ $ id

uid=1111 gid=0(root) groups=555
# Le user d'UID 1111 a un groupe supplémentaire : 555

/ $ stat -c "%a %A %u:%g" /volume

2777 drwxrwsrwx 0:555
# On reconnait l'octet de haut niveau qui vaut 2 (setgid)

/ $ touch /volume/first_container && \
  stat -c "%a %A %u:%g" /volume/first_container

644 -rw-r--r-- 1111:555
# Le GID du fichier créé est égale au GID du répertoire parent
#  Sans le GID, le fichier aurait eu les perm 1111:0

/ $ ^D

$ kubectl exec -it hello-world-bd78cdd6-pm4tq -c second -- sh

/ $ id

uid=2222 gid=0(root) groups=555
# Le user d'UID 2222 a un groupe supplémentaire : 555

/ $ touch /volume/second_container && \
  stat -c "%a %A %u:%g" /volume/second_container

644 -rw-r--r-- 2222:555
# Le GID du fichier créé est égale au GID du répertoire parent
#  Sans le GID, le fichier aurait eu les perm 2222:0


/ $ stat -c "%a %A %u:%g" /volume/\*

644 -rw-r--r-- 1111:555 /volume/first_container
644 -rw-r--r-- 2222:555 /volume/second_container
# Le conteneur 'first' peut lire le contenu du fichier 
#   appartenant au conteneur 'second' et vice versa
{{< /highlight >}}



## Ressources

* [Setuid - wikipédia](https://en.wikipedia.org/wiki/Setuid)
* [Sticky bit - wikipédia](https://en.wikipedia.org/wiki/Sticky_bit)
* [Permissions spéciales - doc Oracle](https://docs.oracle.com/cd/E19683-01/816-4883/secfile-69/index.html)
* [How to use special permissions - linuxconfig.org](https://linuxconfig.org/how-to-use-special-permissions-the-setuid-setgid-and-sticky-bits)