# Node Interactive 2016 - Amsterdam

__Deploying Node.js to Production using Plain Old Linux - Luke Bond__

- __t:__ @lukeb0nd
- __e:__ luke.n.bond@gmail.com

## Materials

Available from Github in the form of markdown and can be served locally:

```
$ git clone https://github.com/lukebond/node-interactive-systemd-workshop.git
$ npm install
$ npm start
```

This will launch a browser window/tab with the docs, otherwise they can be
found at [http://localhost:8000/workshop.md](http://localhost:8000/workshop.md).

Feedback on this workshop will be gratefully received!! Contact details are above.

## How Will This Workshop Work?

- I'll give you a VM to use
  - Email me at luke.n.bond@gmail.com and I'll give you access details for one
  - Please don't do anything illegal with it!
  - I'll tear it down at the end of the workshop (sorry!!)
- Run the materials locally on your laptop
- SSH into the VM and work through the exercises yourself
- Ask me for help when you get stuck
  - This is where the value is in the workshop so don't be shy!!
  - Also feel free to pick my brain about anything you think I may be able to help with

## Workshop

Deployment is a large subject. "Production" environments vary wildly.
This workshop will focus on Linux, process monitoring and on systemd.
It could perhaps be subtitled "how to do what PM2 does with systemd".

The main goal of this workshop is to show you how to implement the basic
process monitoring, logging and port-sharing features of PM2 using "plain old
Linux" (systemd and a few simple tools).

We'll set up a basic Node.js sample app that talks to Redis using systemd.

This means the following:

- The Node service will be restarted when it crashes
- The Node service will be started on reboot
- Starting the Node service will trigger Redis to be started
- We'll do some simple load balancing in order to run multiple instances
- We'll explore systemd's powerful tools for logging and other admin tasks

Why learn these things instead of just using PM2?

- For the record: I don't have anything against PM2!
- I'm a firm believer in using standard, proven and universally understood tools
- Any sysadmin will understand this (PM2 is little known outside the Node world)
- Sysadmins have been doing this with init systems for decades
- systemd is the standard init system in most new distro releases
- Moving away from a process monitor will make it easier to move to a
  containers, a PaaS or an orchestration platform
- Learn more about the operating system on which your apps are running
- It's easier than you think; if you use PM2 because it's easy, you'll be
  surprised how easy this is!

### 1. Access your VM and Sanity Check

SSH into the VM.
Ensure Node.js, Redis and Balance are present, and that you have `sudo` access.
Also check that the demo app is there.

```
$ which node redis-server balance > /dev/null 2>1 && echo 'Okay' || echo 'Not okay'
Okay
$ sudo -v
$ cd /home/ubuntu/demo-api-redis
$ node index
^C
```

If you get "Not okay", let me know!!

(_Note:_ The app will fail because Redis isn't running- that's fine!)

Some useful links to have open during the workshop:

- [Article about systemctl](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)
- [Article about journalctl](https://www.digitalocean.com/community/tutorials/how-to-use-journalctl-to-view-and-manipulate-systemd-logs)
- My [blog post](https://blog.codeship.com/running-node-js-linux-systemd/) on the same subject

### 2. Write a Basic Unit File and Run the Service

Linux init systems have been used for decades to start, stop and restart the
services we run on our Linux servers (e.g. apache, mysql and the like). Before
systemd, we would write bash scripts that the init system would use in order
to start, stop and restart our services. The concept is the same in systemd
but the implementation is different.

We describe _services_ to systemd by writing _unit files_. A unit file is a
text file, like an INI file, that descibes how to start, stop and restart a
service, as well as some configuration information.

In this workshop we will write unit files to descibe our service, evolving
the unit file incrementally as we learn more about systemd.

Unit files live in `/etc/systemd/system`, and systemd can be triggered to
re-read that directory by sending it a signal (which can be sent via the shell
command `systemctl daemon-reload`).

Let's create our first, basic unit file. Create the following text file and
copy it to `/etc/systemd/system/demo-api-redis@.service`:

```
[Unit]
Description=HTTP Hello World
After=network.target

[Service]
User=ubuntu
Environment=REDIS_HOST=localhost
WorkingDirectory=/home/ubuntu/demo-api-redis
ExecStart=/usr/bin/node index.js

[Install]
WantedBy=multi-user.target
```

Run the following commands to start the service:

```
$ systemctl daemon-reload
$ systemctl enable demo-api-redis@1
$ systemctl start demo-api-redis@1
```

Check the status of the service to see if it worked:

```
$ systemctl status demo-api-redis@1
● demo-api-redis@1.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: activating (auto-restart) (Result: exit-code) since Thu 2016-06-30 17:20:09 BST; 62ms ago
  Process: 29787 ExecStart=/usr/bin/node index.js (code=exited, status=1/FAILURE)
 Main PID: 29787 (code=exited, status=1/FAILURE)

Jun 30 17:20:09 luke-arch systemd[1]: demo-api-redis@1.service: Main process exited, code=exited, status=1/FAILURE
Jun 30 17:20:09 luke-arch systemd[1]: demo-api-redis@1.service: Unit entered failed state.
Jun 30 17:20:09 luke-arch systemd[1]: demo-api-redis@1.service: Failed with result 'exit-code'.
```

This is failing because Redis isn’t running. Let’s explore dependencies in systemd!

We can add the `Wants=` directive to the `[Unit]` section of a unit file to
declare dependencies between services. There are other directives with
different semantics (e.g., `Requires=`) but Wants= will cause the depended-upon
service (in this case, Redis) to be started when our Node.js service is
started.

Your unit file should now look like this:

```
[Unit]
Description=HTTP Hello World
After=network.target
Wants=redis.service

[Service]
User=ubuntu
Environment=REDIS_HOST=localhost
WorkingDirectory=/home/ubuntu/demo-api-redis
ExecStart=/usr/bin/node index.js

[Install]
WantedBy=multi-user.target
```

Signal systemd to reload its config:

```
$ systemctl daemon-reload
```

Ask systemd to cat the unit file just to ensure it has picked up our changes:

```
$ systemctl cat demo-api-redis@1
# /etc/systemd/system/demo-api-redis@.service
[Unit]
Description=HTTP Hello World
After=network.target
Wants=redis.service

[Service]
Environment=REDIS_HOST=localhost
User=ubuntu
WorkingDirectory=/home/ubuntu/demo-api-redis
ExecStart=/usr/bin/node index.js

[Install]
WantedBy=multi-user.target
```

And now restart the service. We can see that the service now works:

```
$ systemctl restart demo-api-redis@1
$ systemctl status demo-api-redis@1
● demo-api-redis@1.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2016-06-30 17:17:19 BST; 187ms ago
 Main PID: 27050 (node)
    Tasks: 10 (limit: 512)
   CGroup: /system.slice/system-demo\x2dapi\x2dredis.slice/demo-api-redis@1.service
           └─27050 /usr/bin/node index.js

Jun 30 17:17:19 luke-arch systemd[1]: Started HTTP Hello World.
$ curl localhost:9000
"Hello, world 192.168.1.39! 1 hits."
```

It works because it has triggered Redis to run:

```
$ systemctl status redis
● redis.service - Advanced key-value store
   Loaded: loaded (/usr/lib/systemd/system/redis.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 10:31:54 BST; 3s ago
 Main PID: 28643 (redis-server)
    Tasks: 3 (limit: 512)
   Memory: 6.3M
      CPU: 10ms
   CGroup: /system.slice/redis.service
           └─28643 /usr/bin/redis-server 127.0.0.1:6379 

Jul 01 10:31:54 luke-arch redis-server[28643]:   `-._    `-._`-.__.-'_.-'    _.-'
Jul 01 10:31:54 luke-arch redis-server[28643]:       `-._    `-.__.-'    _.-'
Jul 01 10:31:54 luke-arch redis-server[28643]:           `-._        _.-'
Jul 01 10:31:54 luke-arch redis-server[28643]:               `-.__.-'
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 # Server started, Redis version 3.2.1
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Red
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 * DB loaded from disk: 0.000 seconds
Jul 01 10:31:54 luke-arch redis-server[28643]: 28643:M 01 Jul 10:31:54.216 * The server is now ready to accept connections on port 6379
```

### 3. Writing Unit Files for More Robust Services

What happens if we kill our service?

```
$ systemctl status demo-api-redis@1 | grep "PID"
 Main PID: 28649 (node)
$ sudo kill -9 28649
$ systemctl status demo-api-redis@1
● demo-api-redis@1.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: failed (Result: signal) since Fri 2016-07-01 10:55:49 BST; 2s ago
  Process: 29145 ExecStart=/usr/bin/node index.js (code=killed, signal=KILL)
 Main PID: 29145 (code=killed, signal=KILL)

Jul 01 10:55:39 luke-arch systemd[1]: Started HTTP Hello World.
Jul 01 10:55:40 luke-arch node[29145]: (node:29145) DeprecationWarning: process.EventEmitter is deprecated. Use require('events') instead.
Jul 01 10:55:40 luke-arch node[29145]: Listening on port 9000
Jul 01 10:55:49 luke-arch systemd[1]: demo-api-redis@1.service: Main process exited, code=killed, status=9/KILL
Jul 01 10:55:49 luke-arch systemd[1]: demo-api-redis@1.service: Unit entered failed state.
Jul 01 10:55:49 luke-arch systemd[1]: demo-api-redis@1.service: Failed with result 'signal'.
```

So systemd is not restarting our service when it crashes, but never fear—
systemd has a range of options for configuring this behavior. Adding the
following to the `[Service]` section of our unit file will be fine for our
purposes:

```
Restart=always
RestartSec=500ms
StartLimitInterval=0
```

This tells systemd to always restart the service after a 500ms delay. You can
configure it to give up eventually, but this should be fine for our purposes.
Now reload systemd’s config, restart the service and try killing the
process:

```
$ systemctl daemon-reload
$ systemctl cat demo-api-redis@1
# /etc/systemd/system/demo-api-redis@.service
[Unit]
Description=HTTP Hello World
After=network.target
Wants=redis.service

[Service]
Environment=REDIS_HOST=localhost
User=ubuntu
WorkingDirectory=/home/ubuntu/demo-api-redis
ExecStart=/usr/bin/node index.js

[Install]
WantedBy=multi-user.target
$ systemctl restart demo-api-redis@1
$ systemctl status demo-api-redis@1.service | grep PID
 Main PID: 29145 (code=killed, signal=KILL)
$ sudo kill -9 29145
$ systemctl status demo-api-redis@1
● demo-api-redis@1.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 11:08:41 BST; 2s ago
 Main PID: 29884 (node)
    Tasks: 10 (limit: 512)
   CGroup: /system.slice/system-demo\x2dapi\x2dredis.slice/demo-api-redis@1.service
           └─29884 /usr/bin/node index.js

Jul 01 11:08:41 luke-arch systemd[1]: Stopped HTTP Hello World.
Jul 01 11:08:41 luke-arch systemd[1]: Started HTTP Hello World.
Jul 01 11:08:41 luke-arch node[29884]: (node:29884) DeprecationWarning: process.EventEmitter is deprecated. Use require('events') instead.
Jul 01 11:08:41 luke-arch node[29884]: Listening on port 9000
```

It works! systemd is now restarting our service when it goes down. It will
also start it up automatically if the machine reboots (that’s what it means to
`enable` a service). Go ahead and reboot to prove it!

### 4. Multiple Instances and Load Balancing

Next up, we need to be able to run multiple instances of our service.

You may have noticed that our unit file has an `@` symbol in the filename, and
that we’ve been referring to our service as `demo-api-redis@1`. The `1` after
the `@` symbol is the instance name (it doesn’t have to be a number). We could
run two more instances of our service using something like
`systemctl start demo-api-redis@{2,3}`, but first we need them to bind to
different ports or they’ll clash.

Our sample app takes an environment variable to set the port, so we can use
the instance name to give each service a unique port. Add the following
additional `Environment=` line to the `[Service]` section of the unit file:

```
Environment=LISTEN_PORT=900%i
```

This will mean that `demo-api-redis@1` will get port `9001`, `demo-api-redis@2`
will get port `9002`, and `demo-api-redis@3` will get port `9003`, leaving
`9000` for our load balancer (later).

Once you’ve edited the unit file, you need to reload the config, check that
it’s correct, start two new instances, and restart the existing one:

```
$ systemctl daemon-reload
$ systemctl cat demo-api-redis@1
# /etc/systemd/system/demo-api-redis@.service
[Unit]
Description=HTTP Hello World
After=network.target
Wants=redis.service

[Service]
Environment=REDIS_HOST=localhost
Environment=LISTEN_PORT=900%i
User=luke
WorkingDirectory=/home/ubuntu/demo-api-redis
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=500ms
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
$ systemctl enable demo-api-redis@{2,3}
$ systemctl start demo-api-redis@{2,3}
$ systemctl restart demo-api-redis@1
$ systemctl status demo-api-redis@{1,2,3}
● demo-api-redis@1.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 11:08:41 BST; 56min ago
 Main PID: 29884 (node)
   CGroup: /system.slice/system-demo\x2dapi\x2dredis.slice/demo-api-redis@1.service
           └─29884 /usr/bin/node index.js

Jul 01 11:08:41 luke-arch systemd[1]: Stopped HTTP Hello World.
Jul 01 11:08:41 luke-arch systemd[1]: Started HTTP Hello World.
Jul 01 11:08:41 luke-arch node[29884]: (node:29884) DeprecationWarning: process.EventEmitter is deprecated. Use require('events') instead.
Jul 01 11:08:41 luke-arch node[29884]: Listening on port 9001

● demo-api-redis@2.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 12:04:34 BST; 18s ago
 Main PID: 30747 (node)
   CGroup: /system.slice/system-demo\x2dapi\x2dredis.slice/demo-api-redis@2.service
           └─30747 /usr/bin/node index.js

Jul 01 12:04:34 luke-arch systemd[1]: Started HTTP Hello World.
Jul 01 12:04:34 luke-arch node[30747]: (node:30747) DeprecationWarning: process.EventEmitter is deprecated. Use require('events') instead.
Jul 01 12:04:34 luke-arch node[30747]: Listening on port 9002

● demo-api-redis@3.service - HTTP Hello World
   Loaded: loaded (/etc/systemd/system/demo-api-redis@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 12:04:34 BST; 18s ago
 Main PID: 30753 (node)
   CGroup: /system.slice/system-demo\x2dapi\x2dredis.slice/demo-api-redis@3.service
           └─30753 /usr/bin/node index.js

Jul 01 12:04:34 luke-arch systemd[1]: Started HTTP Hello World.
Jul 01 12:04:34 luke-arch node[30753]: (node:30753) DeprecationWarning: process.EventEmitter is deprecated. Use require('events') instead.
Jul 01 12:04:34 luke-arch node[30753]: Listening on port 9003
```

We should now be able to curl each of these:

```
$ curl localhost:900{1,2,3}
"Hello, world 192.168.1.39! 52 hits.""Hello, world 192.168.1.39! 53 hits.""Hello, world 192.168.1.39! 54 hits."
```

__Next:__ load balancing!

One could use NGINX or HAProxy to balance the traffic across the instances of
our service. However, since I’m claiming that it’s super simple to replace PM2
functionality, I wanted to go with something lighter.

[Balance](https://inlab.de/balance.html) is a tiny (a few-hundred lines of C)
TCP load balancer that’s fast and simple to use. For example:

```
$ balance -f 9000 127.0.0.1:900{1,2,3} &
$ curl localhost:9000
"Hello, world 192.168.1.39! 20 hits."
```

The above one-liner launches balance, listening on port 9000 and balancing
across ports 9001-9003. But we don’t want to run it in the foreground like
this. Let’s write a unit file:

```
$ cat /etc/systemd/system/balance.service
[Unit]
Description=Balance - Simple TCP Load Balancer
After=syslog.target network.target nss-lookup.target

[Service]
ExecStart=/usr/bin/balance -f 9000 127.0.0.1:9001 127.0.0.1:9002 127.0.0.1:9003

[Install]
WantedBy=multi-user.target
$ ps -ef | grep balance
ubuntu    3648  3061  0 12:30 pts/0    00:00:00 balance -f 9000 127.0.0.1:9001 127.0.0.1:9002 127.0.0.1:9003
ubuntu    4014  3061  0 12:36 pts/0    00:00:00 grep --color=auto balance
$ sudo kill -9 3648
[1]+  Killed                  balance -f 9000 127.0.0.1:900{1,2,3}
$ systemctl daemon-reload
$ systemctl enable balance
$ systemctl start balance
$ systemctl status balance
● balance.service - Balance - Simple TCP Load Balancer
   Loaded: loaded (/etc/systemd/system/balance.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-07-01 13:56:46 BST; 3s ago
 Main PID: 32674 (balance)
    Tasks: 1 (limit: 512)
   Memory: 316.0K
      CPU: 10ms
   CGroup: /system.slice/balance.service
           └─32674 /usr/bin/balance -f 9000 127.0.0.1:9001 127.0.0.1:9002 127.0.0.1:9003

Jul 01 13:56:46 luke-arch systemd[1]: Started Balance - Simple TCP Load Balancer.
$ curl localhost:9000
"Hello, world 192.168.1.39! 21 hits."
```

### Recap

What have we learned so far?

- How to write basic unit files for our services
- Ensure our services will be restarted on crash or reboot
- Super-simple local reverse-proxy for load balancing across multiple processes
- We don't need a process manager when we have systemd

Let's explore some basic system administration tasks with systemd's tooling.

### Take-away Tips For Production

- The unit files I'm providing you are a good start but not intended to be
  rock-solid!
- Bring all your existing best-practices to your unit files!
  - systemd doesn't change anything in this respect, but gives you some new
    ways to express the same best practices
- Be 12-factor: use environment variables!
  - You can set envvars in unit files
  - Either individually (e.g. `Environment=NODE_ENV=production` in the
    `[Service]` section
  - Or as a batch (e.g. `EnvironmentFile=/etc/environment` in the
    `[Service]` section
- Watch Lennart's talk from CoreOS Fest Berlin 2016 about security- many
  tips for your unit files!

### For Those Who Finish Early

- Containerise the Node application and modify the unit files to launch
  containers for the Node application and Redis
- Install Etcd and Fleet and run your services using Fleet rather than
  systemd directly
- Imagine `demo-api-redis` is a real application, built and tested in your CI
  pileline
  - Think about how you'd get from code to build artefact and to a
    deploying with systemd
  - Share your thoughts with those near you, and with me
  - How would you perform an update?
  - What if you wanted zero downtime during these updates?
- Note that there is no single/clear answer to any of these questions,
  it will vary based on your application, environment and business. Hence
  discussion is encouraged!

### THANKS!

Thanks for attending my workshop, I hope you found it useful.

Please provide feedback (positive or negative), it helps me improve!
