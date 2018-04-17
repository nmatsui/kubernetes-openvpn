# kubernetes-openvpn
 A kubernetes configuration to create VPN over 443/tcp using pieterlange/kube-openvpn

## Usage
### Construct OpenVPN on Kubernetes
1. generate PKI key

    ```bash
    $ docker run --user=$(id -u) -e OVPN_SERVER_URL=tcp://<FQDN of OpenVPN Service on Kubernetes>:443 -v $PWD:/etc/openvpn:z -ti ptlange/openvpn ovpn_initpki
    $ docker run --user=$(id -u) -e EASYRSA_CRL_DAYS=180 -v $PWD:/etc/openvpn:z -ti ptlange/openvpn easyrsa gen-crl
    ```
1. get CIDR of Kubernetes's Service and Pod
    * When you cannot get service cider, you can use `10.0.0.0/8` as service CIDR if it does not conflict the client network.

    ```bash
    $ kubectl cluster-info dump | grep "service-cluster-ip-range"
    $ kubectl cluster-info dump | grep "cluster-cidr"
    ```
1. start OpenVPN on Kubernetes

    ```bash
    $ ./kube-openvpn/deploy.sh <namespace> <OpenVPN URL> <service cidr> <pod cidr>
    ```
    * like below:

    ```bash
    $ ./kube-openvpn/deploy.sh default tcp://vpn.example.com:443 10.0.0.0/8 10.244.0.0/16
    ```
1. create client configuration file as `client.ovpn`

    ```bash
    $ docker run --user=$(id -u) -v $PWD:/etc/openvpn:z -ti ptlange/openvpn easyrsa build-client-full client nopass
    $ docker run --user=$(id -u) -e OVPN_SERVER_URL=tcp://vpn.example.com:443 -v $PWD:/etc/openvpn:z --rm ptlange/openvpn ovpn_getclient client > client.ovpn
    ```

### Confirm VPN connection and client IPv4 Address
1. scp `client.ovpn` to `ubuntu-1`
1. install openvpn package to `ubuntu-1`

    ```bash
    ubuntu-1:~$ sudo apt update && sudo apt install openvpn -y
    ```
1. connect to OpenVPN
    * If you can see the `Initialization Sequence Completed` message, your vpn connection is established.

    ```bash
    ubuntu-1:~$ sudo openvpn ./client.ovpn
    ...
    Tue Apr 17 04:01:24 2018 Initialization Sequence Completed
    ```
1. check the IPv4 Address of `eth0` and `tun0`
    * assume that the IPv4 address of `eth0` is `192.168.0.4/24` and the IPv4 Address of `tun0` is `10.140.0.2/24`

    ```bash
    ubuntu-1:~$ ip addr show dev eth0
    ubuntu-1:~$ ip addr show dev tun0
    ```
1. close vpn connection

### Access a service on Kubernetes from vpn client machine(ubuntu-1)
1. start testservice on Kubernetes
    * assume that the IPv4 address of `testservice` is `10.0.19.96`

    ```bash
    $ kubectl apply -f testservice.yaml
    $ kubectl get services -l app=testservice
    ```
1. start vpn connection
1. access from vpn client machine(`ubuntu-1`) to `testservice`

    ```bash
    ubuntu-1:~$ curl -i http://10.0.19.96:3000/
    ```
1. close vpn connection

### Access a service on Kubernetes from another machine(ubuntu-2) which is placed on the same network of vpn client
1. allow packet forwarding of `ubuntu-1`

    ```bash
    ubuntu-1:~$ sudo sysctl -w net.ipv4.ip_forward=1
    ```
1. add nat rule to forward packet (port 3000) from local network to 'testservice'

    ```bash
    ubuntu-1:~$ sudo iptables -t nat -A PREROUTING -m tcp -p tcp --dst 192.168.0.4 --dport 3000 -j DNAT --to-destination 10.0.19.96:3000
    ubuntu-1:~$ sudo iptables -t nat -A POSTROUTING -m tcp -p tcp --dst 10.0.19.96 --dport 3000 -j SNAT --to-source 10.140.0.2
    ```
1. add forward filter to allow forwarding packet

    ```bash
    ubuntu-1:~$ sudo iptables -A FORWARD -m tcp -p tcp --dst 10.0.19.96 --dport 3000 -j ACCEPT
    ubuntu-1:~$ sudo iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
    ```
1. start testservice on Kubernetes
    * assume that the IPv4 address of `testservice` is `10.0.19.96`

    ```bash
    $ kubectl apply -f testservice.yaml
    $ kubectl get services -l app=testservice
    ```
1. start vpn connection
1. access from `ubuntu-2` to `testservice` through `ubuntu-1`

    ```bash
    ubuntu-2:~$ curl -i http://192.168.0.4:3000
    ```
1. close vpn connection

### Routing back a service on vpn client machine(ubuntu-1) from Kubernetes
1. set tun0 IPv4 address and netmask of `ubuntu-1`

    ```yaml
    $ kubectl edit configmap openvpn-ccd
    ```
    ```diff:diff
    --- /tmp/openvpn-ccd.org	2018-04-17 13:23:32.000000000 +0900
    +++ /tmp/openvpn-ccd	2018-04-17 13:24:14.000000000 +0900
    @@ -5,6 +5,7 @@
     apiVersion: v1
     data:
       example: ifconfig-push 10.140.0.5 255.255.255.0
    +  ubuntu-1: ifconfig-push 10.140.0.2 255.255.255.0
     kind: ConfigMap
     metadata:
       annotations:
    ```
1. set port mapping (8081 forward to `ubuntu-1:8081`)

    ```bash
    $ kubectl edit configmap openvpn-portmapping
    ```
    ```diff:diff
    --- /tmp/openvpn-portmapping.org	2018-04-17 13:28:40.000000000 +0900
    +++ /tmp/openvpn-portmapping	2018-04-17 13:29:10.000000000 +0900
    @@ -5,6 +5,8 @@
     apiVersion: v1
     data:
       "20080": example:80
    +  "8081": ubuntu-1:8081
     kind: ConfigMap
     metadata:
       annotations:
    ```
1. start proxy service of OpenVPN like below:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: testproxy-1
      labels:
        app: testproxy-1
    spec:
      type: ClusterIP
      selector:
        openvpn: <FQDN of OpenVPN Service on Kubernetes>
      ports:
      - port: 8081
        targetPort: 8081
    ```
1. start vpn connection
1. start httpd (listening 8081) on `ubuntu-1`

    ```bash
    ubuntu-1:~$ python -m SimpleHTTPServer 8081
    ```
1. start pod on Kubernetes and connect to `ubuntu-1`

    ```bash
    $ kubectl run testpod --rm -it --image=yauritux/busybox-curl
    testpod:~$ curl -i http://testproxy-1:8081
    ```
1. close vpn connection

### Routing back a service on another machine(ubuntu-2) from Kubernetes
1. set port mapping (8082 forward to `ubuntu-1:8082`)

    ```bash
    $ kubectl edit configmap openvpn-portmapping
    ```
    ```diff:diff
    --- /tmp/openvpn-portmapping.org	2018-04-17 13:28:40.000000000 +0900
    +++ /tmp/openvpn-portmapping	2018-04-17 13:29:10.000000000 +0900
    @@ -5,6 +5,8 @@
     apiVersion: v1
     data:
       "20080": example:80
       "8081": ubuntu-1:8081
    +  "8082": ubuntu-1:8082
     kind: ConfigMap
     metadata:
       annotations:
    ```
1. start proxy service of OpenVPN like below:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: testproxy-2
      labels:
        app: testproxy-2
    spec:
      type: ClusterIP
      selector:
        openvpn: <FQDN of OpenVPN Service on Kubernetes>
      ports:
      - port: 8082
        targetPort: 8082
    ```
1. allow packet forwarding of `ubuntu-1`

    ```bash
    ubuntu-1:~$ sudo sysctl -w net.ipv4.ip_forward=1
    ```
1. add nat rule to forward packet (port 8082) from Kubernetes to 'ubuntu-2'

    ```bash
    ubuntu-1:~$ sudo iptables -t nat -A PREROUTING -m tcp -p tcp --dst 10.140.0.2 --dport 8082 -j DNAT --to-destination 192.168.0.5:8082
    ubuntu-1:~$ sudo iptables -t nat -A POSTROUTING -m tcp -p tcp --dst 192.168.0.5 --dport 8082 -j SNAT --to-source 192.168.0.4
    ```
1. add forward filter to allow forwarding packet

    ```bash
    ubuntu-1:~$ sudo iptables -A FORWARD -m tcp -p tcp --dst 192.168.0.5 --dport 8082 -j ACCEPT
    ubuntu-1:~$ sudo iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
    ```
1. start vpn connection
1. start httpd (listening 8082) on `ubuntu-2`

    ```bash
    ubuntu-2:~$ python -m SimpleHTTPServer 8082
    ```
1. start pod on Kubernetes and connect to `ubuntu-2`

    ```bash
    $ kubectl run testpod --rm -it --image=yauritux/busybox-curl
    testpod:~$ curl -i http://testproxy-2:8082
    ```
1. close vpn connection


## See also

* [kube-openvpn](https://github.com/pieterlange/kube-openvpn)

## License

[MIT License](/LICENSE)

## Copyright
Copyright (c) 2018 Nobuyuki Matsui <nobuyuki.matsui@gmail.com>
