# consul-basics-playground

## Bootstrap Datacenter
- environment overview:
```
georgiman@MacBook-Machine consul-basics-playground (vagrant_add) $ vagrant status
Current machine states:

server1                   running (virtualbox)
server2                   running (virtualbox)
server3                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
```
- Starting consul agent in server mode on server1
```
georgiman@MacBook-Machine consul-basics-playground (vagrant_add) $ vagrant ssh server1
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-151-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Fri Nov  8 08:53:24 2019 from 10.0.2.2
vagrant@server1:~$ consul agent -server -bootstrap-expect=3 -bind 192.168.10.11 -data-dir=/tmp/consul
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
           Version: 'v1.6.1'

```
- Starting consul agent in server mode on server2
```
vagrant@server2:~$ consul agent -server -bootstrap-expect=3 -bind=192.168.10.21 -data-dir=/tmp/consul
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
           Version: 'v1.6.1'
```
- Starting consul agent in server mode on server3
```
vagrant@server3:~$ consul agent -server -bootstrap-expect=1 -bind=192.168.10.31 -data-dir=/tmp/consul
BootstrapExpect is set to 1; this is the same as Bootstrap mode.
bootstrap = true: do not enable unless necessary
==> Starting Consul agent...
           Version: 'v1.6.1'
```
- Start bootstarping procedure on server1
```
vagrant@server1:~$ consul join 192.168.10.21 192.168.10.31
Successfully joined cluster by contacting 2 nodes.
vagrant@server1:~$ 
vagrant@server1:~$ consul members
Node     Address             Status  Type    Build  Protocol  DC   Segment
server1  192.168.10.11:8301  alive   server  1.6.1  2         dc1  <all>
server2  192.168.10.21:8301  alive   server  1.6.1  2         dc1  <all>
server3  192.168.10.31:8301  alive   server  1.6.1  2         dc1  <all>
vagrant@server1:~$ 
```

## Consul -retry-join configuration
- I have added `basic_config.json` to the /etc/consul.d
```
{
    "bootstrap": false,
    "bootstrap_expect": 3,
    "server": true,
    "retry_join": ["192.168.10.11", "192.168.10.21", "192.168.10.31"]
  }
```
- server1 has been started with following command:
```
vagrant@server1:~$ consul agent -bind=192.168.10.11 -data-dir=/tmp/consul -config-dir=/etc/consul.d
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
           Version: 'v1.6.1'
```
- server2 and server3 have been started with following command:
```
consul agent -bind=192.168.10.21 -data-dir=/tmp/consul -config-dir=/etc/consul.d
consul agent -bind=192.168.10.31 -data-dir=/tmp/consul -config-dir=/etc/consul.d
```

- as result, server2 and server3 were joined
```
2019/11/08 13:18:58 [INFO] serf: Re-joined to previously known node: server3: 192.168.10.31:8301
2019/11/08 13:18:58 [INFO] consul: Adding LAN server server2 (Addr: tcp/192.168.10.21:8300) (DC: dc1)
2019/11/08 13:18:58 [INFO] consul: Adding LAN server server3 (Addr: tcp/192.168.10.31:8300) (DC: dc1)
2019/11/08 13:18:58 [INFO] consul: Handled member-join event for server "server3.dc1" in area "wan"
2019/11/08 13:18:58 [INFO] consul: Handled member-join event for server "server2.dc1" in area "wan"
2019/11/08 13:18:58 [INFO] serf: Re-joined to previously known node: server2.dc1: 192.168.10.21:8302
2019/11/08 13:18:58 [INFO] agent: (LAN) joined: 3
2019/11/08 13:18:58 [INFO] agent: Join LAN completed. Synced with 3 initial agents
```
## KV store CLI commands:
```
vagrant@server1:~$ consul kv put redis/config/connections 5
Success! Data written to: redis/config/connections
vagrant@server1:~$ 
vagrant@server1:~$ consul kv get redis/config/connections
5
vagrant@server1:~$ consul kv get -detailed redis/config/connections
CreateIndex      559
Flags            0
Key              redis/config/connections
LockIndex        0
ModifyIndex      559
Session          -
Value            5
vagrant@server1:~$ consul kv delete redis/config/connections
Success! Deleted key: redis/config/connections
vagrant@server1:~$ 
```

## Playing with envconsul
- starting consul agent in dev mode
```
vagrant@server1:~$ consul agent -dev
==> Starting Consul agent...
           Version: 'v1.6.1'
```
- write some data:
```
vagrant@server1:~$ consul kv put my-app/address 1.2.3.4
Success! Data written to: my-app/address
vagrant@server1:~$ 
vagrant@server1:~$ consul kv put my-app/port 80
Success! Data written to: my-app/port
vagrant@server1:~$ 
vagrant@server1:~$ consul kv put my-app/max_cons 5
Success! Data written to: my-app/max_cons
vagrant@server1:~$ 
```
- Execute envconsul with a subprocess (I will use /bin/bash)
```
vagrant@server1:~$ envconsul -prefix my-app /bin/bash
```
- Envconsul will connect to Consul, read the data from the key-value store, and populate environment variables corresponding to those values
```
vagrant@server1:~$ env | egrep 'address|port|max_cons'
address=1.2.3.4
port=80
max_cons=5
vagrant@server1:~$
```

## Consul-template usage
There are two use cases: Consul KV and Discover All Services

### Consul-template to query Consul's KV store
- start consul server in dev mode.
```
consul agent -dev
```
- execute consul-template command, which will start query the consul's KV store 
```
consul-template -template "find_address.tpl:hashicorp_address.txt"
```
- we are going to write data to the consul key:
```
consul kv put hashicorp/street_address "101 2nd St"
```
- into /vagrant directory (directory from which we have started consul-template), we need to have `hashicorp_address.txt` file. Into this file should be populated the value of the key `hashicorp/street_address`
```
vagrant@server1:/vagrant$ cat hashicorp_address.txt 
101 2nd St
vagrant@server1:/vagrant$
```
- we will update the `hashicorp/street_address` key and will check whether content of `hashicorp_address.txt` will be changed:
```
vagrant@server1:/vagrant$ consul kv put hashicorp/street_address "22b Baker ST"
Success! Data written to: hashicorp/street_address
vagrant@server1:/vagrant$ 
vagrant@server1:/vagrant$ cat hashicorp_address.txt 
22b Baker ST
vagrant@server1:/vagrant$
```
#### Consul-template to Discover All Services
- start consul server in dev mode.
```
consul agent -dev
```
- will create template file which will query all services
```
{{range services}}# {{.Name}}{{range service .Name}}
{{.Address}}{{end}}

{{end}}
```
- run consul template specifying `-once` flag. (will run the process once and then quit)
```
vagrant@server1:/vagrant$ consul-template -template="all-services.tpl:all-services.txt" -once
vagrant@server1:/vagrant$ cat all-services.txt
# consul
127.0.0.1
vagrant@server1:/vagrant$
```