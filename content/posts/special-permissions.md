+++
author = "Jean-Charles Legras"
title = "Special Permissions"
date = "2020-09-03"
draft = false
tags = ["setuid", "setgid", "sticky bit", "special permissions", "restricted deletion flag"]
toc = true
+++

## Summary

In this section, I will introduce the special permissions that are less commonly used than standard permissions.

First, we will discuss **setuid** and **setgid** which can be set on files, using a small Golang program to observe their effects.  
Next, we will examine **setgid** but this time on directories.  
Finally, we will look into the **sticky bit** on directories.  
To conclude, we will take a look at a Kubernetes feature that utilizes **setgid**.

## Special Permissions on a File

### setuid/setgid

__Requirement__: Execute a file with the privileges of the file's owner/group respectively.

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

- The *Real UID/GID* corresponds respectively to the UID/GID of the current user
- The *Effective UID/GID* corresponds respectively to the adopted UID/GID
- The highest-order octet is set to 2 for setgid and 4 for setuid

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
# By default, the effective UID/GID are set to my user UID/GID

$ sudo chmod u+s main && stat -c "%a %A %U:%G" main
4775 -rwsrwxr-x root:docker

$ ./main
Real UID        = 1000
Effective UID   = 0
Real GID        = 1000
Effective GID   = 1000
# With setuid, the UID adopted by the process is that of the owner of the executable file

$ sudo chmod g+s main && stat -c "%a %A %U:%G" main
6775 -rwsrwsr-x root:docker

$ ./main
Real UID        = 1000
Effective UID   = 0
Real GID        = 1000
Effective GID   = 999
# With setgid, the GID adopted by the process is that of the group of the executable file
# Since setuid is still set, the effective UID remains that of root

{{< /highlight >}}

To begin with, we observe that when the setuid bits are set, the *effective UID* is set to *root*, the owner of the file.

Similarly, when the setgid bits are set, the *effective GID* is set to *docker*, the group of the file.

Furthermore, since both setuid (4) and setgid (2) bits are set, the highest-order octet is indeed 4 + 2 = 6.

Finally, the process representing the execution of the *main.go* file is indeed jclegras:jclegras (1000:1000) when no special permissions are set at all.

__Requirement__: list files with setuid that would result in an effective UID equal to root

Here is an example conducted on the */bin* directory:
{{< highlight bash >}}
$ find /bin -user root -perm -4000 -exec ls -ldb {} \;

-rwsr-xr-x 1 root root 26696 mars   5  2020 /bin/umount
-rwsr-xr-x 1 root root 30800 août  11  2016 /bin/fusermount
-rwsr-xr-x 1 root root 64424 juin  28  2019 /bin/ping
-rwsr-xr-x 1 root root 43088 mars   5  2020 /bin/mount
-rwsr-xr-x 1 root root 44664 mars  22  2019 /bin/su
{{< /highlight >}}

We can see, for example, that the ping command imposes privileges that a regular user is not supposed to have. Therefore, the setuid bits are set on the *ping* command.

Indeed, the *ping* utility must be able to create ICMP sockets, which have restrictions in Linux kernels: [man pages icmp(7)](http://man7.org/linux/man-pages/man7/icmp.7.html)

       ping_group_range (two integers; default: see below; since Linux
       2.6.39)
              Range of the group IDs (minimum and maximum group IDs,
              inclusive) that are allowed to create ICMP Echo sockets.  The
              default is "1 0", which means no group is allowed to create
              ICMP Echo sockets.

### Special Permissions on a Directory

#### setgid

__Requirement__: I want to share a directory among multiple people to create a workspace without having to change my primary group (GID).

{{< highlight bash >}}
$ mkdir special-directory && \
chown jclegras:docker special-directory/ && \
stat -c "%a %A %U:%G" special-directory/

775 drwxrwxr-x jclegras:docker 
# Create a directory with 775 permissions and UID:GID jclegras:docker

$ touch special-directory/my-file && \
  stat -c "%a %A %U:%G" special-directory/my-file

664 -rw-rw-r-- jclegras:jclegras 
# The file created has the GID of my current user

$ chmod g+s special-directory/ && \
  stat -c "%a %A %U:%G" special-directory

2775 drwxrwsr-x jclegras:docker 
# We notice the highest-order octet set to 2 (setgid)

$ stat -c "%a %A %U:%G" special-directory/my-file

664 -rw-rw-r-- jclegras:jclegras 
# The permissions of the existing file do not change

$ touch special-directory/my-file2 && \
  stat -c "%a %A %U:%G" special-directory/my-file2

664 -rw-rw-r-- jclegras:docker 
# The new file created inherits the GID of the directory in which it is created

$ touch my-file3 && \
  stat -c "%a %A %U:%G" my-file3

664 -rw-rw-r-- jclegras:jclegras

$ mv -v my-file3 special-directory/ && \
  stat -c "%a %A %U:%G" special-directory/my-file3

664 -rw-rw-r-- jclegras:jclegras
# The permissions of an existing file moved into the directory do not change

$ mkdir special-directory/subdirectory && \
  stat -c "%a %A %U:%G" special-directory/subdirectory/

2775 drwxrwsr-x jclegras:docker
# The directory "inherits" the setgid permission
{{< /highlight >}}

We observe the following with this series of commands:
- By default, a file created from a "classic" directory (without setgid, for example) has a GID equal to the primary GID of the user who created it;
- When the setgid bits are set on the directory, the change of ownership does not apply to existing files in this directory;
- A file created in a directory with setgid has a GID equal to the GID of the directory;
- The "inheritance" of the group to a file only applies at the creation of the file, moving a file into a directory with setgid does not change the permissions of said file;
- Any directory created within a directory with setgid set also inherits the setgid from its parent directory.

__Note__: *setuid* is ignored on a directory under Linux.

#### Sticky bit (more precisely here: restricted deletion flag)

__Reminder__: as soon as you have write permissions on a directory, you can delete the files it contains,
whether you have write permissions on the file or not.

If you do not have permissions on the file, you will only get a warning prompt:

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

__Requirement__: Protect files in a directory so that only the owner can delete/rename them


{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" mydir

777 drwxrwxrwx jclegras:jclegras mydir
# With permissions set to 777, anyone can delete the files inside

$ chmod o+t mydir && \
  stat -c "%a %A %U:%G %n" mydir

1777 drwxrwxrwt jclegras:jclegras mydir
# The sticky bit is set to 1

$ touch mydir/myfile && \
  stat -c "%a %A %U:%G %n" mydir/myfile

664 -rw-rw-r-- jclegras:jclegras mydir/myfile 
# For deletion, the file permissions do not matter (reminder)

$ sudo su jcbis

$ rm -v mydir/myfile

rm: remove write-protected regular empty file 'mydir/myfile'? y
rm: cannot remove 'mydir/myfile': Operation not permitted
# Deletion impossible even though the directory permissions allow it
#   For example, we have o+rwx, so 'Others' can delete files
#   It is indeed the sticky bit that imposes a stronger restriction here (!)

$ mv -v mydir/myfile .
mv: cannot move 'mydir/myfile' to './myfile': Operation not permitted
# Same for file renaming
{{< /highlight >}}

We can even take it a step further by setting both the sticky bit AND the setgid on the directory:

{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" mydir

3777 drwxrwsrwt jclegras:jcbis mydir
# Each file created will have the group 'jcbis'
#  2 pour setgid et 1 pour sticky bit

$ touch mydir/myfile

$ sudo su jcbis

$ stat -c "%a %A %U:%G %n" mydir/myfile

664 -rw-rw-r-- jclegras:jcbis mydir/myfile
# jcbis belongs to the 'jcbis' group, they could delete this file

$ rm -v mydir/myfile

rm: cannot remove 'mydir/myfile': Operation not permitted
# Operation not permitted by the sticky bit (!)
{{< /highlight >}}

When both setgid and sticky bits are set, the sticky bit always applies.
The sticky bit prevents deletion and renaming of the file.
This flag combined with setgid on a directory results in an append-only file.
Indeed, nothing prevents *jcbis* from reading and writing to the file.

An example of the sticky bit application can be found on the */tmp* directory:

{{< highlight bash >}}
$ stat -c "%a %A %U:%G %n" /tmp
1777 drwxrwxrwt root:root /tmp
{{< /highlight >}}

We can search for examples of directories with the sticky bit using the *find* tool:
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
According to the [Linux man pages](https://man7.org/linux/man-pages/man1/chmod.1.html#RESTRICTED_DELETION_FLAG_OR_STICKY_BIT), we learn that the term *sticky bit* is reserved for the bit when it is set on a file.
On modern systems, the sticky bit is ignored when it concerns files.
When this bit is set on a directory, it is called the *restricted deletion flag*. The description above shows its functionality.

### Kubernetes

A feature located at the [PodSecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) level uses setgid for sharing (certain types of) volumes between multiple containers: *spec.fsGroup*.

Here is a sample YAML to demonstrate this feature:

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

Let's demonstrate *setgid* in action with this series of Bash commands:

{{< highlight bash >}}
$ kubectl exec -it hello-world-bd78cdd6-pm4tq -c first -- sh
# Or any other name your Pod might have

/ $ id

uid=1111 gid=0(root) groups=555
# The user with UID 1111 has an additional group: 555

/ $ stat -c "%a %A %u:%g" /volume

2777 drwxrwsrwx 0:555
# We recognize the highest-order octet set to 2 (setgid)

/ $ touch /volume/first_container && \
  stat -c "%a %A %u:%g" /volume/first_container

644 -rw-r--r-- 1111:555
# The GID of the created file is equal to the GID of the parent directory
# Without the GID, the file would have the permissions 1111:0

/ $ ^D

$ kubectl exec -it hello-world-bd78cdd6-pm4tq -c second -- sh

/ $ id

uid=2222 gid=0(root) groups=555
# The user with UID 2222 has an additional group: 555

/ $ touch /volume/second_container && \
  stat -c "%a %A %u:%g" /volume/second_container

644 -rw-r--r-- 2222:555
# The GID of the created file is equal to the GID of the parent directory
# Without the GID, the file would have the permissions 2222:0


/ $ stat -c "%a %A %u:%g" /volume/\*

644 -rw-r--r-- 1111:555 /volume/first_container
644 -rw-r--r-- 2222:555 /volume/second_container
The 'first' container can read the contents of the file 
  belonging to the 'second' container and vice versa
{{< /highlight >}}

## Resources

* [Setuid - Wikipedia](https://en.wikipedia.org/wiki/Setuid)
* [Sticky bit - Wikipedia](https://en.wikipedia.org/wiki/Sticky_bit)
* [Special Permissions - Oracle Documentation](https://docs.oracle.com/cd/E19683-01/816-4883/secfile-69/index.html)
* [How to use special permissions - linuxconfig.org](https://linuxconfig.org/how-to-use-special-permissions-the-setuid-setgid-and-sticky-bits)