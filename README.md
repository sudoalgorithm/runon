<img src="https://github.ibm.com/kubernetes-tools/runon/raw/master/logo.png" width="150">

# Introduction

This repo has two UNIX/Linux shell scripts that use Kubernetes to run
commands on the nodes that _host_ the kubernetes workload.

To install the `runon` and `runevery` scripts as command line tools on your local machine, clone this project, make the files executable, and move them to your local binary directory. Example in OS X or Linux:
```
git clone git@github.ibm.com:kubernetes-tools/runon.git
cd runon
chmod +x runon
mv runon /usr/local/bin/runon
chmod +x runevery
mv runevery /usr/local/bin/runevery
```

## runon

A shell script that is used to run commands, get a shell or start live wireshark capture on a kubernetes node. This can be useful for watching logs, running diagnostics, debugging host network etc.

N.B. This sort of action is an anti pattern, it's a big footgun, but it can be useful occasionally


Usage
-----

OpenShift Prerequisite: 
Because of OpenShift SCC you need to apply the following policy to the ibm-system service account before using runon. 
```
oc adm policy add-scc-to-user privileged system:serviceaccount:ibm-system:default
```

1. collect a list of nodes in your cluster
```
kubectl get nodes
NAME           STATUS    ROLES     AGE       VERSION
10.186.68.77   Ready     <none>    1d        v1.8.8-2+9d6e0610086578
10.186.68.80   Ready     <none>    1d        v1.8.8-2+9d6e0610086578
10.186.68.95   Ready     <none>    1d        v1.8.8-2+9d6e0610086578
```

2. run a command on one of those nodes
```
$ runon 10.186.68.77 command uptime
 19:57:09 up 1 day,  4:02,  0 users,  load average: 0.25, 0.18, 0.16

```

or optionally get a shell on one of the nodes

```
$ runon 10.186.68.80 command bash
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/#
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/# uptime
 19:57:27 up 1 day,  4:02,  0 users,  load average: 0.19, 0.17, 0.16
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/#
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/# w
 19:57:29 up 1 day,  4:02,  0 users,  load average: 0.17, 0.17, 0.16
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/#
root@kube-dal13-cr3d88646a51e643abbdd352a34b9aa307-w3:/# exit
```

3. start wireshark live capture on eth0 interface on one of the nodes. Please note, for this feature wireshark must be installed on your desktop and must be on $PATH.
```
$ runon 10.186.68.80 capture eth0
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
```
## runevery

Whereas `runon` runs a command on one host, `runevery` runs a command
on every host.  Following is an example usage (with reporting cut
short for brevity).

```
mjs11:runon mspreitzus.ibm.com$ ./runevery grep -w VERSION /etc/os-release
set=r065dolsjm3ge4q5rupvtjcihh6f741d srctime=190126-023037 logfile=/tmp/pod/from-mjs11-at-190126-023037 command=grep -w VERSION /etc/os-release
desired=0 available=
desired=208 available=162
desired=208 available=208
At css1comp0: VERSION="18.04.1 LTS (Bionic Beaver)"

At css1comp1: VERSION="18.04.1 LTS (Bionic Beaver)"

At css1comp10: ^Cmjs11:runon mspreitzus.ibm.com$ 
mjs11:runon mspreitzus.ibm.com$ kubectl -n ibm-system delete ds r065dolsjm3ge4q5rupvtjcihh6f741d
daemonset.extensions "r065dolsjm3ge4q5rupvtjcihh6f741d" deleted
```

This created a DaemonSet that ran the given command on every host in
an initContainer and then ran a sleep loop in a regular container.
The shell command waited until every desired pod was "available", and
then iterated through them and fetched the logged output.

The shell script does _not_ delete the DaemonSet.  You want to do that
yourself.  You can do that for one particular DaemonSet as shown
above.  Or you can delete all of them as follows.

```
kubectl -n ibm-system delete ds -l app=runevery
```

### Quoting Hell

The arguments to runevery are joined with a space between them, then
enclosed by dquotes, and then that is interpreted by bash in the wrong
context to produce the command that will be executed.  Following are
some examples.

#### A happy example

```bash
runevery echo "'hi there'"
```
will go through
```bash
bash -c "echo 'hi there'"
```
in the wrong context and execute
```bash
echo 'hi there'
```
in the right context.

#### Beware DQuotes

```bash
runevery 'echo "hi there" >' file
```
will try go through
```bash
bash -c "echo "hi there" > file"
```
--- which will not be happy, because dquotes do not nest.

#### Evaluating too early

```bash
runevery echo '$(ls)'
```
will go through
```bash
bash -c "echo $(ls)"
```
in the wrong context to execute
```bash
echo file1 file2 and others from the wrong context
```


#### Delay evaluation enough

```bash
runevery echo '\$(ls)'
```
will go through
```bash
bash -c "echo \$(ls)"
```
in the wrong context to execute
```bash
echo $(ls)
```
in the right context.
