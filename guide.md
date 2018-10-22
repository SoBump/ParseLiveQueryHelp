# BUMP's Parse Live Query Guide – How to Set Up A Parse Live Query Server

_This was written on 10.22.2018_, using Parse-Server 2.5 or later. 

## Table of Contents:
* [One Server Setup]()
* [Separate Live Query Server Setup]()
* [Resources]()
	* [A note about AWS]()
	* [AWS Nginx Config]()
* [Troubleshooting]()

## One Server Setup
It’s possible to set up a Parse Live Query Server on a single server or single instance of your application. **Please note that if your application runs over multiple instances, users will likely be connected to different sockets which could prevent them from connecting in real time to one another. It is recommended that you use the** [Separate Live Query Server Setup]()

In your `ParseServer` setup,  first include the `liveQuery` key with the following object, specifying the class names of the Mongo objects that you will live query with.

```
  liveQuery: {
    classNames: ['messages', '_User'], // for example
    redisURL: "redis://:{password}@redis-12345.c10.us-east-1-1.ec2.cloud.redislabs.com:18091" || redis://localhost:6379
  },
```

_Note: I signed up for a Redis Cloud Free Tier Account (see Resources) to get this URL and password_

For an example working index.js file, see the following Gist – [Single Server Live Query Setup · GitHub](https://gist.github.com/zackshapiro/279d4acf83e81da0ea03f3553619ef87)

You’ll connect via a client by doing `self.client = ParseLiveQuery.client(server: "wss://myApp.com")` (this is Swift 4, this will change depending on your client language)

**Diagram of the Single Server Setup**

![](&&&SFLOCALFILEPATH&&&lq_local.png)


## Separate Live Query Server Setup
The benefit of running a separate Live Query Server from the rest of your production app is that you can provide a separate URL to hit, regardless of the instance the user’s account is on when they interact with your app.

For example, my app is deployed on 30+ AWS instances (read: servers) but all of them connect to `wss://socket.myApp.com` to use my Parse Live Query server, so it doesn’t matter which instance a user is doing their session’s requests on.

For an example of two working index.js files, see the following Gist – [Parse Separate Live Query Server Setup  · GitHub](https://gist.github.com/zackshapiro/8c8b1967da5788ffed801ff4e9184dde)

Some keys to note here:
* Your main app’s `index.js` file includes the key `liveQuery` with the associated object. The `redisURL` key is optional, only include if you’re using Redis.
* Your separate Live Query Server is instantiated with a simple server listening on a port and the `/`  route, for good measure, then you give it an object with the same `appId` and  `masterKey` as your main app, so they know they can talk to each other.
	* redisURL key here is again optional but if you’re using it on your main app, you also have to use it here to ensure the pub/sub connection is connected.
* In your client code, you’ll connect by specifying the exact URL of the socket. For example, in my Swift 4 code, I do `self.client = ParseLiveQuery.client(server: "wss://socket.myApp.com")`

**Diagram of a the Separate Live Query Server setup (via Parse docs):**

![](&&&SFLOCALFILEPATH&&&lq_multiple.png)

## Resources:
* [ParsePlatform/Chat - Gitter](https://gitter.im/ParsePlatform/Chat) – A great community of Parse users. Feel free to shoot me a message on there if you have questions.
* [Parse Server Guide | Parse](https://docs.parseplatform.org/parse-server/guide/#live-queries)
* Test your web socket URL [websocket.org Echo Test - Powered by Kaazing](https://www.websocket.org/echo.html)

### A Note About AWS

In order for Parse Live Query Servers to work, you need to make sure that in Elastic Beanstalk -> Your Server Name -> Configuration your Load Balancer type is `application`. If you’re using a Classic Load Balancer, Parse Live Query will not work.

![](&&&SFLOCALFILEPATH&&&Screen%20Shot%202018-10-22%20at%203.48.56%20PM.png)

### AWS Nginx Config

This is taken straight from Amazon’s docs, no custom setup applied.

```
###################################################################################################
#### Copyright 2016 Amazon.com, Inc. or its affiliates. All Rights Reserved.
####
#### Licensed under the Apache License, Version 2.0 (the "License"). You may not use this file
#### except in compliance with the License. A copy of the License is located at
####
####     http://aws.amazon.com/apache2.0/
####
#### or in the "license" file accompanying this file. This file is distributed on an "AS IS"
#### BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
#### License for the specific language governing permissions and limitations under the License.
###################################################################################################

###################################################################################################
#### To enable WebSockets on the Node.js platform, this configuration file
#### configures the Nginx proxy server.
#### It replaces the Nginx configuration file with a version that includes the
#### "Upgrade" and "Connection" headers: https://www.nginx.com/blog/websocket-nginx/
####
#### This Elastic Beanstalk configuration file replaces the entire Nginx configuration file.
#### As a result, you won't get our managed updates to the Nginx configuration file, if we make
#### updates in future platform updates. Be sure to test this configuration with new platform versions.
####
#### This configuration file works with single-instance and load-balanced environments.
####
#### If you're using a Classic Load Balancer, you need to enable TCP listeners. See:
#### http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.managing.elb.html

#### This file from: https://github.com/awsdocs/elastic-beanstalk-samples/blob/master/configuration-files/aws-provided/instance-configuration/websockets/nodejs/websockets-nodejs.config
###################################################################################################

files:
   /etc/nginx/conf.d/proxy.conf:
     owner: root
     group: root
     mode: "000644"
     content: |
       # Elastic Beanstalk Managed

       # Elastic Beanstalk managed configuration file
       # Some configuration of nginx can be by placing files in /etc/nginx/conf.d
       # using Configuration Files.
       # http://docs.amazonwebservices.com/elasticbeanstalk/latest/dg/customize-containers.html

       client_max_body_size 5M;

       upstream nodejs {
           server 127.0.0.1:8081;
           keepalive 256;
       }

       server {
           listen 8080;

           if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
               set $year $1;
               set $month $2;
               set $day $3;
               set $hour $4;
           }
           access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd;
           access_log  /var/log/nginx/access.log  main;

           location / {
               proxy_pass  http://nodejs;
               proxy_http_version 1.1;
               proxy_set_header        Upgrade $http_upgrade;
               proxy_set_header        Connection "upgrade";
               proxy_set_header        Host            $host;
               proxy_set_header        X-Real-IP       $remote_addr;
               proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
           }

       gzip on;
       gzip_comp_level 4;
       gzip_types text/html text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

       }

   /opt/elasticbeanstalk/hooks/configdeploy/post/99_kill_default_nginx.sh:
     owner: root
     group: root
     mode: "000755"
     content: |
       #!/bin/bash -xe
       rm -f /etc/nginx/conf.d/00_elastic_beanstalk_proxy.conf
       service nginx stop
       service nginx start

container_commands:
  removeconfig:
    command: "rm -f /tmp/deployment/config/#etc#nginx#conf.d#00_elastic_beanstalk_proxy.conf /etc/nginx/conf.d/00_elastic_beanstalk_proxy.conf"
```

## Troubleshooting

#### Redis
* You may need to include a Redis password in the format I’ve specified above to securely connect to your Redis connection
* For security reasons, use an environment variable for that Redis password

#### Ports 
* If you’re connecting to a separate Live Query Server, you’ll have to specify that URL when you create your client.
* I my main app and separate servers on separate ports when I test locally, for example 1337 (main) and 1338 (live)
