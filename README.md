# Add Fail2ban to Postal Server's SMTP server in Docker container.

This is a small tutorial on how to prevent brute force attacks to the SMTP server container included into [Postal Server](https://github.com/postalserver/postal) using Fail2ban. As logs are sent to stdout, it is necessary to write them to a file so File2ban can monitor it. This can be achieved changing the postal configuration to write logs to a file, binding container's log's folder to another in the host and set up a new filter in Fail2ban to read the binded log file.

Tested in Ubuntu Server 22.04.1 LTS with Postal v2.11.2.

Corrections, comments and suggestions are welcome!


## Postal Server configuration

In ```/opt/postal/config/postal.yml``` set stdout to false:
```yaml
[...]

logging:
  # Specify options for the logging
  stdout: false

[...]
```

Add a ```docker-compose.override.yml``` in ```/opt/postal/install``` folder in order to extend smtp server volumes. The goal is to bind the log's folder inside the container to an accessible directory to fail2ban, installed on the host.
 ```/opt/postal/log/``` is the path where the log files will be on the host and ```/opt/postal/app/log/``` is the path where log files are located in the _postal-smtp_ docker.

```yaml
version: "3.9"
services:
  smtp:
    volumes:
      - /opt/postal/log/:/opt/postal/app/log/

```

As it is not recommended to run the docker as root and in order to not changing the original docker image, it is necessary to change log's folder/files ownership to the same uid running inside the container, otherwise container won't start and "Permission Denied" messages will be shown when running ```postal logs smtp```

Checking inside container:
```bash
postal@postal-test:~/app/log$ ls -l
total 4
-rw-r----- 1 postal postal    0 Oct  3 18:59 rails.log
-rw-r----- 1 postal postal 1185 Oct  3 20:16 smtp_server.log

postal@postal:~/app/log$ id postal
uid=999(postal) gid=999(postal) groups=999(postal)
```
So in the host should exist a user with the same uid (999) in order to set the ownership with the same uid. In this case it doesn't matter if the user's name is not the same as the host.
```bash
$ sudo chown -R 999:999 /opt/postal/log
$ sudo chmod -R 0600 /opt/postal/log
```

Restart Postal and check all services has started.

```bash
$ sudo postal stop
$ sudo postal start
$ sudo postal status
NAME                COMMAND                  SERVICE             STATUS              PORTS
postal-cron-1       "/docker-entrypoint.…"   cron                running             
postal-requeuer-1   "/docker-entrypoint.…"   requeuer            running             
postal-smtp-1       "/docker-entrypoint.…"   smtp                running             
postal-web-1        "/docker-entrypoint.…"   web                 running             
postal-worker-1     "/docker-entrypoint.…"   worker              running             

```

## Fail2ban configuration
Add this at the end of ```/etc/fail2ban/jail.conf```

```ini
[postal-smtp]
enabled = true
logpath = /opt/postal/log/smtp_server.log
bantime = 960
findtime = 960
maxretry = 3
```

Create a new file ```/etc/fail2ban/filter.d/postal-smtp.conf``` with these contents:
```ini
[Definition]
failregex = WARN: AUTH failure for ::ffff:<HOST>
```
Reload fail2ban to apply new configuration and check:
```bash
$ sudo fail2ban-server reload
OK  

$ sudofail2ban-server status
Status
|- Number of jail:	2
`- Jail list:	postal-smtp, sshd
```

## Logrotate config
As logs aren't managed by docker anymore, it's necessary to set up a log rotation scheme.  
Create new file ```/etc/logrotate.d/postal-smtp```:
```
/opt/postal/log/*.log {

    weekly
    rotate 8
    compress
    notifempty
    copytruncate
    delaycompress
    missingok
    create 640 999 999
```

### Refs
[](https://github.com/postalserver/postal)  
[](https://www.fail2ban.org)
[](https://github.com/postalserver/postal/discussions/2166)  
[]([https://github.com/postalserver/postal/pull/1187)  
[](https://github.com/postalserver/postal/discussions/1639)  
[](https://www.the-lazy-dev.com/en/install-fail2ban-with-docker/)  
[](https://techflare.blog/permission-problems-in-bind-mount-in-docker-volume/)  