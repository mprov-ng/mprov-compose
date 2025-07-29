# mprov-compose
A docker compose file to start up an entire mprov stack on a single node.

We recommend using docker-ce for your docker environment.


Pacakge Dependencies: 

- text editor (vim)
- tmux or screen
- epel-release
- git
- docker-ce

General installation instructions:
```
- install docker-ce 
  - https://docs.docker.com/engine/install/rhel/
- git clone https://github.com/mprov-ng/mprov-compose.git
- cd mprov-compose
- edit stack.env
- docker compose -f mprov-compose.yml -d --env-file stack.env up
< go have a sandwich >
- http://ipaddress/admin/ to login 
```
