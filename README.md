﻿# Deploy Rocket.Chat with K8s, a simple guide.

This guide will help you deploy a Rocket.Chat server with 2 Rocket.Chat replicas and 3 mongodb Replicas.

## Installation

Create the mongodb service:

```bash
$ kubectl apply -f mongo.service.yaml
```

Create the mongodb deployment:

```bash
$ kubectl apply -f mongo.deployment.yaml
```

With this config, we have 3 replicas which one of them will be the primary database and 2 of them are the mirrors of the primary database.

But still, there are not configured in that way and they are just 3 mongo databases with no relation between them. To start connecting, we should execute one of them to make it the primary database:

```bash
$ kubectl exec -it rocketmongo-0 mongo
```

You will bash into mongo server deployment to initialize the replicas

```bash
> rs.initiate()
```

The response should be like this:

```bash
{
    "info2" : "no configuration specified. Using a default configuration for the set",
    "me" : "rocketmongo-0:27017",
    "ok" : 1,
    "operationTime" : Timestamp(1609450853, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1609450853, 1),
        "signature" : {
           "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
           "keyId" : NumberLong(0)
        }
    }
}
```

Now we should set this mongo to be the primary one. So, run this code:

```bash
> var config = rs.conf()
> config.members[0].host="rocketmongo-0.rocketmongo:27017"
> rs.reconfig(config)
```

Now, we add the other databases as replicas using this command:

```bash
> rs.add("rocketmongo-1.rocketmongo:27017")
> rs.add("rocketmongo-2.rocketmongo:27017")
```

The configuration of the replica set is finished and we can check the status using this function:

```bash
> rs.status()
```

Create the Rocket.Chat service:

```bash
$ kubectl apply -f rocket.service.yaml
```

Create the Rocket.Chat deployment:

```bash
$ kubectl apply -f rocket.deployment.yaml
```

Expose it to external access with your k8s tool

In this case, the service is using the nodePort: 30007

So, usign nginx, this port is being forwared to a FQDN:

```bash
server {
    listen 443 ssl;
    server_name chat.YOUR-DOMAIN.ga;

    ssl_certificate /etc/letsencrypt/live/api.lgrocket.ga/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.lgrocket.ga/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://chat.YOUR-DOMAIN.ga:30007;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Nginx-Proxy true;

        proxy_redirect off;
    }
}
```

To upgrade the Rocket.Chat version (IE.: MicroK8s):

```bash
microk8s.kubectl set image deployment/rocketchat-server rocketchat-server=rocketchat/rocket.chat:4.4.0
```







