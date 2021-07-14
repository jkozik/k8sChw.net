# k8sChw.net
Setup camptonhillsweather.net in a kubernetes cluster as deployment named chwnet; expose as service chwnet; connect to the internet as camptonhillsweather.net through an ingress called chwnet-ingress. Get realtime weather data from an NFS mount on my local host through the PV/PVC named chwcom-persistent-storage -- note, this is the same PV/PVC used by [k8sChw.com](https://github.com/jkozik/k8sChw.com). 

Source image [InstallCHW.net](https://github.com/jkozik/InstallCHW.net)

# build image, put on docker hub
CamptonHillsWeather.net is running in a docker container on my host, directly -- not in a VM.  To make it run in my kubernetes cluster, I need to push the image to my dockerhub repository.  The kubernetes deployment resource has an "image" field that triggers a pull from dockerhub.
```
docker login
docker tag jkozik/chw.net jkozik/chw.net:v1
docker push jkozik/chw.net:v1
```
Next, the website displays weather data derived from realtime data uploaded to an NFS share on my host.  This share is configured into a persistent volume and persistent volume claim called chwcom-persistent storage.  This was setup as part of the [k8sChw.com](https://github.com/jkozik/k8sChw.com) application.  This application uses the same realtime input, but this repository does not re-create the PV/PVC -- it reuses them.  

Verify that the PV/PVC setup by chwcom exists and is bound.

```
[jkozik@dell2 k8sChw.net]$ kubectl get pv,pvc | grep chwcom
persistentvolume/chwcom-persistent-storage      1Gi        ROX            Retain           Bound    default/chwcom-persistent-storage      nfs                     17h
persistentvolumeclaim/chwcom-persistent-storage      Bound    chwcom-persistent-storage      1Gi        ROX            nfs            17h
```
Get the yaml manifest files and apply them.  Verify that the pod, deployment, service and ingress are successfully running.
```
[jkozik@dell2 ~]$ git clone https://github.com/jkozik/k8sChw.net
Cloning into 'k8sChw.net'...
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 11 (delta 0), reused 5 (delta 0), pack-reused 0
Unpacking objects: 100% (11/11), done.

[jkozik@dell2 ~]$ cd k8sChw.net
[jkozik@dell2 k8sChw.net]$ ls
chwnet-deploy.yml  chwnet-ingress.yml  chwnet-svc.yml  README.md

[jkozik@dell2 k8sChw.net]$ kubectl apply -f .
deployment.apps/chwnet created
ingress.networking.k8s.io/chwnet-ingress created
service/chwnet created

[jkozik@dell2 k8sChw.net]$ kubectl logs pod/chwnet-6cc668bd7f-whb84
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.68.77.148. Set the 'ServerName' directive globally to suppress this message
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.68.77.148. Set the 'ServerName' directive globally to suppress this message
[Wed Jul 14 13:32:04.403931 2021] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.38 (Debian) PHP/7.2.31 configured -- resuming normal operations
[Wed Jul 14 13:32:04.403989 2021] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
10.68.77.136 - - [14/Jul/2021:13:32:10 +0000] "GET /weather34uvsolar.php?_=1626212812259 HTTP/1.1" 200 1761 "http://camptonhillsweather.net/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
10.68.77.136 - - [14/Jul/2021:13:32:10 +0000] "GET /moonphase.php?_=1626212812258 HTTP/1.1" 200 1183 "http://camptonhillsweather.net/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
10.68.77.136 - - [14/Jul/2021:13:32:29 +0000] "GET /barometer.php?_=1626212812260 HTTP/1.1" 200 1004 "http://camptonhillsweather.net/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
10.68.77.136 - - [14/Jul/2021:13:32:52 +0000] "GET / HTTP/1.1" 200 36232 "-" "Zabbix"
10.68.77.136 - - [14/Jul/2021:13:34:11 +0000] "GET /weather34uvsolar.php?_=1626212812261 HTTP/1.1" 200 1761 "http://camptonhillsweather.net/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
10.68.77.136 - - [14/Jul/2021:13:34:11 +0000] "GET /moonphase.php?_=1626212812262 HTTP/1.1" 200 1183 "http://camptonhillsweather.net/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
```
Based on the log file, the application is already responding to http GET requests.  Either way, it is useful to verify that camptonhillsweather.net works from a CLI and a browser.
```
[jkozik@dell2 k8sChw.net]$ curl camptonhillsweather.net | head
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
<!DOCTYPE html>
<html lang="en">
<head>
  <title>Campton Hills, IL USA Smart Home Weather Station</title>
  <meta content="Smart Home weather station providing current weather conditions for Campton Hills, IL USA" name="description">
  <meta content="website" property="og:type">
  <meta content="7 days" name="revisit-after">
  <meta content="web" name="distribution">
  <meta content="Campton Hills, IL USA
100  4142    0  4142    0     0  24775      0 --:--:-- --:--:-- --:--:-- 24951
```
