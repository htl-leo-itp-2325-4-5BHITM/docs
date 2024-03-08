## **Auf vm:**

- curl keycloak
- unzip, rm zip, mv to /opt/keylcoak
- add ingress rule all ports to 8000
- iptables -L -n -x -v
- cd etc/iptables → 8000 allown → restart
- start in screen with port → bin/kc.sh start-dev --http-port 8000
- add client backend 
    with Client authentication and Authorization on
    add project-url to root and home url, add web origins: +