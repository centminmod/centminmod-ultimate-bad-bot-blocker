# Contents

* [General Notes](#general-notes)
* [Install](#install)
* [Post Install Checks](#post-install-checks)
  * [Check Nginx.conf Added Includes](#check-nginxconf-added-include-files)
    * [nginx.conf fix](#nginxconf-fix)
  * [Check Vhosts Added Includes](#check-vhosts-added-include-files)
    * [quick diff](#quick-diff)
    * [extended diff](#extended-diff)
* [cronjob](#cronjob)
* [Example Outputs](#example-output-from-install-commands)
  * [dry runs](#dry-runs)
  * [live runs](#live-runs)
    * [actual installed files](#actual-installed-files)
* [Testing](#testing)
* [Rate Limits](#rate-limits)
* [White Listing](#white-listing)
* [Black Listing](#black-listing)
* [Bad Referrers](#bad-referrers)
* [blockbots.conf](#blockbotsconf)
* [Customize Configurations](#customize-configurations)
* [globalblacklist.conf](#globalblacklistconf)
* [ngxtop](#ngxtop)
    
## General Notes

Installation commands for Mitchell Krog developed [Ultimate Bad Bot Blocker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker) are for [Centmin Mod 123.09beta01](https://centminmod.com/) or higher LEMP stack on CentOS 6/7 specifically due to the differences in [Centmin Mod's Nginx structure](https://centminmod.com/configfiles.html). If you have existing [bad bot blocking & rate limiting](https://community.centminmod.com/threads/blocking-bad-or-aggressive-bots.6433/) setup, you will need to remove or comment out those include files with hash # in front of them first for

* `include /usr/local/nginx/conf/botlimit.conf;` in `/usr/local/nginx/conf/nginx.conf`
* `include /usr/local/nginx/conf/blockbots.conf;` within each of your Centmin Mod Nginx vhost config files within directory at `/usr/local/nginx/conf/conf.d`

Instructions below are provided as is with no support provided by me. For issues with false postives blocks etc, you will need to contact the official developer on their [Ultimate Bad Bot Blocker issue tracker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/issues)

## Install

Actual install commands for Nginx Ultimate Bad Bot Blocker installed at `/usr/local/nginx/conf/ultimate-badbot-blocker` and where the global bad bot blacklisting is contained in `/usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf`.

```
# download and install
wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/install-ngxblocker -O /usr/local/sbin/install-ngxblocker
chmod +x /usr/local/sbin/install-ngxblocker
mkdir -p /usr/local/nginx/conf/ultimate-badbot-blocker
# backup nginx.conf and conf.d directory before install
cp -a /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf-backup-b4-badbot
cp -a /usr/local/nginx/conf/conf.d/ /usr/local/nginx/conf/conf.d-backup-b4-badbot

# dry run
install-ngxblocker -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
# live run
install-ngxblocker -x -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
# fix duplicate directives
sed -i 's|^server_names_hash_|#server_names_hash_|g' /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf

# dry run
setup-ngxblocker -e conf -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -v /usr/local/nginx/conf/conf.d -m /usr/local/nginx/conf/nginx.conf
# live run
setup-ngxblocker -x -e conf -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -v /usr/local/nginx/conf/conf.d -m /usr/local/nginx/conf/nginx.conf
```

## Post Install Checks

### check nginx.conf added include files


You can check nginx.conf config file at `/usr/local/nginx/conf/nginx.conf` against the backed up copy at `/usr/local/nginx/conf/nginx.conf-backup-b4-badbot` which you would of backed up in initial install commands above.

`setup-ngxblocker` doesn't setup `/usr/local/nginx/conf/nginx.conf` includes properly as it inserted include files in wrong place in centmin mod nginx.conf outside of http{] context

```
nginx -t
nginx: [emerg] "server_names_hash_bucket_size" directive is not allowed here in /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf:15
nginx: configuration file /usr/local/nginx/conf/nginx.conf test failed
```

sdiff side by side compare of backed up copy versus live copy of nginx.conf

```
sdiff -w 200 -s /usr/local/nginx/conf/nginx.conf-backup-b4-badbot /usr/local/nginx/conf/nginx.conf
                                                                                                   >            # Bad Bot Blocker
                                                                                                   >            include /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf; 
                                                                                                   >            include /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf;
                                                                                                   >  
```

universal diff compare with 10 line context output of backed up copy versus live copy of nginx.conf

```
diff -U10 /usr/local/nginx/conf/nginx.conf-backup-b4-badbot /usr/local/nginx/conf/nginx.conf 
--- /usr/local/nginx/conf/nginx.conf-backup-b4-badbot   2018-03-07 01:39:33.843255502 +0000
+++ /usr/local/nginx/conf/nginx.conf    2018-04-02 16:52:06.659257479 +0000
@@ -1,11 +1,15 @@
 user              nginx nginx;
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf;
+ 
 worker_processes 2;
 worker_priority -10;
 
 worker_rlimit_nofile 260000;
 timer_resolution 100ms;
 
 pcre_jit on;
 include /usr/local/nginx/conf/dynamic-modules.conf;
```

#### nginx.conf fix

fix is to move the 3 lines within http{} context after existing `variables_hash_max_size 2048;` line

```
http {
 map_hash_bucket_size 128;
 map_hash_max_size 4096;
 server_names_hash_bucket_size 128;
 server_names_hash_max_size 2048;
 variables_hash_max_size 2048;

 # Bad Bot Blocker
 include /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf;
 include /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf;
```

then recheck and test nginx conf and you will get a new error for duplicate `server_names_hash_bucket_size` as `/usr/local/nginx/conf/nginx.conf` also lists this directive and it's being duplicated in thie bad bot installer's include file on line 15 of `/usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf`

```
nginx -t
nginx: [emerg] "server_names_hash_bucket_size" directive is duplicate in /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf:15
nginx: configuration file /usr/local/nginx/conf/nginx.conf test failed
```

#### fix duplicate directives

commented instructions in `/usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf` already mention to comment out line 15 if it conflicts with your existing nginx.conf settings

```
cat -n /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf
     1  ##############################################################################                                                                
     2  #       _  __     _                                                          #
     3  #      / |/ /__ _(_)__ __ __                                                 #
     4  #     /    / _ `/ / _ \\ \ /                                                 #
     5  #    /_/|_/\_, /_/_//_/_\_\                                                  #
     6  #       __/___/      __   ___       __     ___  __         __                #
     7  #      / _ )___ ____/ /  / _ )___  / /_   / _ )/ /__  ____/ /_____ ____      #
     8  #     / _  / _ `/ _  /  / _  / _ \/ __/  / _  / / _ \/ __/  '_/ -_) __/      #
     9  #    /____/\_,_/\_,_/  /____/\___/\__/  /____/_/\___/\__/_/\_\\__/_/         #
    10  #                                                                            #
    11  ##############################################################################                                                                
    12
    13  # Version 1.1
    14
    15  server_names_hash_bucket_size 128;
    16  server_names_hash_max_size 4096;
    17  limit_req_zone $binary_remote_addr zone=flood:50m rate=90r/s;
    18  limit_conn_zone $binary_remote_addr zone=addr:50m;
    19
    20  # ****************************************************************************
    21  # NOTE: IF you are using a system like Nginx-Proxy from @JWilder
    22  # ****************************************************************************
    23  # Repo URL: https://github.com/jwilder/nginx-proxy
    24  # You will need to comment out the first line here as follows. 
    25  #     #server_names_hash_bucket_size 128;
    26  # You will also need to modify the nginx.tmpl file to add the default include
    27  #     include /etc/nginx/conf.d/*
    28  # ****************************************************************************
```

So edit `/usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf` line 15 to comment it out with a hash in front. Also comment out `server_names_hash_max_size` on line 16 as nginx.conf also already has that. Above instructions have been updated for sed edit and commenting out so you wouldn't need to manually do this anyway.

```
sed -i 's|^server_names_hash_|#server_names_hash_|g' /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf
```

```
cat -n /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf
     1  ##############################################################################                                                                
     2  #       _  __     _                                                          #
     3  #      / |/ /__ _(_)__ __ __                                                 #
     4  #     /    / _ `/ / _ \\ \ /                                                 #
     5  #    /_/|_/\_, /_/_//_/_\_\                                                  #
     6  #       __/___/      __   ___       __     ___  __         __                #
     7  #      / _ )___ ____/ /  / _ )___  / /_   / _ )/ /__  ____/ /_____ ____      #
     8  #     / _  / _ `/ _  /  / _  / _ \/ __/  / _  / / _ \/ __/  '_/ -_) __/      #
     9  #    /____/\_,_/\_,_/  /____/\___/\__/  /____/_/\___/\__/_/\_\\__/_/         #
    10  #                                                                            #
    11  ##############################################################################                                                                
    12
    13  # Version 1.1
    14
    15  #server_names_hash_bucket_size 128;
    16  #server_names_hash_max_size 4096;
    17  limit_req_zone $binary_remote_addr zone=flood:50m rate=90r/s;
    18  limit_conn_zone $binary_remote_addr zone=addr:50m;
    19
    20  # ****************************************************************************
    21  # NOTE: IF you are using a system like Nginx-Proxy from @JWilder
    22  # ****************************************************************************
    23  # Repo URL: https://github.com/jwilder/nginx-proxy
    24  # You will need to comment out the first line here as follows. 
    25  #     #server_names_hash_bucket_size 128;
    26  # You will also need to modify the nginx.tmpl file to add the default include
    27  #     include /etc/nginx/conf.d/*
    28  # ****************************************************************************
```

then edit `/usr/local/nginx/conf/nginx.conf` to raise the value of `server_names_hash_max_size` from 2048 to 4096

```
http {
 map_hash_bucket_size 128;
 map_hash_max_size 4096;
 server_names_hash_bucket_size 128;
 server_names_hash_max_size 4096;
 variables_hash_max_size 2048;
```

recheck nginx config

```
nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

### check vhosts added include files

You can check vhosts' config files in `/usr/local/nginx/conf/conf.d` against the backed up copy of them at `/usr/local/nginx/conf/conf.d-backup-b4-badbot` which you would of backed up in initial install commands above.

#### quick diff

```
diff -qr /usr/local/nginx/conf/conf.d-backup-b4-badbot /usr/local/nginx/conf/conf.d   
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/demodomain.com.conf and /usr/local/nginx/conf/conf.d/demodomain.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain1.com.conf and /usr/local/nginx/conf/conf.d/domain1.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain2.com.conf and /usr/local/nginx/conf/conf.d/domain2.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.conf and /usr/local/nginx/conf/conf.d/http2.domain1.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.ssl.conf and /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub1.domain1.com.conf and /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub2.domain2.com.conf and /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf differ
Files /usr/local/nginx/conf/conf.d-backup-b4-badbot/virtual.conf and /usr/local/nginx/conf/conf.d/virtual.conf differ
```

#### extended diff

for the most part centmin mod nginx generated vhosts had the include files added in correct place above the first `location /` context with exception of 2 vhost files where added include files weren't added directly above `location /` but were added further up in vhost config files below. Though technically it is still correct so nothing needed to correct.

* `/usr/local/nginx/conf/conf.d/demodomain.com.conf`
* `/usr/local/nginx/conf/conf.d/virtual.conf`

diff compare

```
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot /usr/local/nginx/conf/conf.d
```

diff compare output

```
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot /usr/local/nginx/conf/conf.d
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/demodomain.com.conf /usr/local/nginx/conf/conf.d/demodomain.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/demodomain.com.conf   2018-03-07 01:39:33.105393742 +0000
+++ /usr/local/nginx/conf/conf.d/demodomain.com.conf    2018-04-02 16:52:06.858264720 +0000
@@ -10,8 +10,12 @@
 server {
             listen   80;
             server_name www.demodomain.com;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
 # limit_conn limit_per_ip 16;
 # ssi  on;
 
             access_log /home/nginx/domains/demodomain.com/log/access.log ;
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain1.com.conf /usr/local/nginx/conf/conf.d/domain1.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain1.com.conf      2018-04-02 16:20:00.996652290 +0000
+++ /usr/local/nginx/conf/conf.d/domain1.com.conf       2018-04-02 16:52:07.268279631 +0000
@@ -36,8 +36,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain2.com.conf /usr/local/nginx/conf/conf.d/domain2.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/domain2.com.conf      2018-04-02 16:20:11.412014562 +0000
+++ /usr/local/nginx/conf/conf.d/domain2.com.conf       2018-04-02 16:52:07.466286830 +0000
@@ -36,8 +36,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.conf /usr/local/nginx/conf/conf.d/http2.domain1.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.conf        2018-04-02 16:24:20.198637890 +0000
+++ /usr/local/nginx/conf/conf.d/http2.domain1.com.conf 2018-04-02 16:52:08.071308824 +0000
@@ -36,8 +36,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.ssl.conf /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/http2.domain1.com.ssl.conf    2018-04-02 16:24:20.205638109 +0000
+++ /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf     2018-04-02 16:52:08.285316598 +0000
@@ -65,8 +65,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub1.domain1.com.conf /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub1.domain1.com.conf 2018-04-02 16:20:26.498539308 +0000
+++ /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf  2018-04-02 16:52:07.664294028 +0000
@@ -36,8 +36,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub2.domain2.com.conf /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/sub2.domain2.com.conf 2018-04-02 16:20:39.443989582 +0000
+++ /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf  2018-04-02 16:52:07.870301518 +0000
@@ -36,8 +36,12 @@
   # server and/or vhost site
   #include /usr/local/nginx/conf/cloudflare.conf;
   include /usr/local/nginx/conf/503include-main.conf;
 
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
   location / {
   include /usr/local/nginx/conf/503include-only.conf;
 
 # block common exploits, sql injections etc
diff -r -U4 /usr/local/nginx/conf/conf.d-backup-b4-badbot/virtual.conf /usr/local/nginx/conf/conf.d/virtual.conf
--- /usr/local/nginx/conf/conf.d-backup-b4-badbot/virtual.conf  2018-03-07 01:39:33.106393555 +0000
+++ /usr/local/nginx/conf/conf.d/virtual.conf   2018-04-02 16:52:07.060272069 +0000
@@ -1,8 +1,12 @@
 server {
             listen 80 default_server backlog=2048 reuseport fastopen=256;
             server_name centos7.localdomain;
             root   html;
+       # Bad Bot Blocker
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf; 
+       include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;
+ 
 
         access_log              /var/log/nginx/localhost.access.log     combined buffer=8k flush=1m;
         error_log               /var/log/nginx/localhost.error.log      error;
 
```

Once all fixed you can restart nginx server

```
service nginx restart
```

or centmin mod cmd shortcut

```
ngxrestart
```

## cronjob

setup cronjob via `cronjob -e` command to invoke nano text editor replacing `yourname@youremail.com` with your email address for update notifications

```
00 */8 * * * /usr/local/sbin/update-ngxblocker -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -e yourname@youremail.com
```

you can do a manual update check too

without email notification add `-n`


```
/usr/local/sbin/update-ngxblocker -n -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
```

example without email notifictaion

```
/usr/local/sbin/update-ngxblocker -n -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d

LOCAL Version: 3.2018.04.1080
Updated: Mon Apr  2 15:35:14 SAST 2018

REMOTE Version: 3.2018.04.1080
Updated: Mon Apr  2 15:35:14 SAST 2018

Latest Blacklist Already Installed: 3.2018.04.1080
```

with email notification

```
/usr/local/sbin/update-ngxblocker -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -e yourname@youremail.com
```

example with email notification

```
/usr/local/sbin/update-ngxblocker -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -e yourname@youremail.com

LOCAL Version: 3.2018.04.1080
Updated: Mon Apr  2 15:35:14 SAST 2018

REMOTE Version: 3.2018.04.1080
Updated: Mon Apr  2 15:35:14 SAST 2018

Latest Blacklist Already Installed: 3.2018.04.1080

Emailing report to: yourname@youremail.com
```

## example output from install commands

### dry runs

```
install-ngxblocker -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

** Dry Run ** | not updating files | run  as 'install-ngxblocker -x' to install files.

Creating directory: /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/conf.d/globalblacklist.conf            [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf
Downloading [FROM]=>  [REPO]/conf.d/botblocker-nginx-settings.conf  [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/bots.d/blockbots.conf              [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf
Downloading [FROM]=>  [REPO]/bots.d/ddos.conf                   [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf
Downloading [FROM]=>  [REPO]/bots.d/custom-bad-referrers.conf   [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/custom-bad-referrers.conf
Downloading [FROM]=>  [REPO]/bots.d/bad-referrer-words.conf     [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/bad-referrer-words.conf
Downloading [FROM]=>  [REPO]/bots.d/blacklist-domains.conf      [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-domains.conf
Downloading [FROM]=>  [REPO]/bots.d/blacklist-ips.conf          [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-ips.conf
Downloading [FROM]=>  [REPO]/bots.d/blacklist-user-agents.conf  [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-user-agents.conf
Downloading [FROM]=>  [REPO]/bots.d/whitelist-domains.conf      [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-domains.conf
Downloading [FROM]=>  [REPO]/bots.d/whitelist-ips.conf          [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/setup-ngxblocker      [TO]=>  /usr/local/sbin/setup-ngxblocker
Downloading [FROM]=>  [REPO]/update-ngxblocker     [TO]=>  /usr/local/sbin/update-ngxblocker
```

```
setup-ngxblocker -e conf -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -v /usr/local/nginx/conf/conf.d -m /usr/local/nginx/conf/nginx.conf
Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

** Dry Run ** | not updating files | run  as 'setup-ngxblocker -x' to setup files.

inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf;            => /usr/local/nginx/conf/nginx.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf;  => /usr/local/nginx/conf/nginx.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/demodomain.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/demodomain.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/virtual.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/virtual.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/http2.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/http2.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf

Whitelisting ip:  xxx.xxx.xxx.xxx   => /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf

Web directory not found ('/var/www'): not whitelisting domains.

Checking for missing includes:

Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

Nothing to update for directory: /usr/local/nginx/conf/ultimate-badbot-blocker
Nothing to update for directory: /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
Nothing to update for directory: /usr/local/sbin
Setting mode: 700 => /usr/local/sbin/install-ngxblocker
Setting mode: 700 => /usr/local/sbin/setup-ngxblocker
Setting mode: 700 => /usr/local/sbin/update-ngxblocker
```

### live runs

```
install-ngxblocker -x -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

Creating directory: /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/conf.d/globalblacklist.conf            [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf...OK
Downloading [FROM]=>  [REPO]/conf.d/botblocker-nginx-settings.conf  [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf...OK

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/bots.d/blockbots.conf              [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/ddos.conf                   [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/custom-bad-referrers.conf   [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/custom-bad-referrers.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/bad-referrer-words.conf     [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/bad-referrer-words.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/blacklist-domains.conf      [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-domains.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/blacklist-ips.conf          [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-ips.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/blacklist-user-agents.conf  [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-user-agents.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/whitelist-domains.conf      [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-domains.conf...OK
Downloading [FROM]=>  [REPO]/bots.d/whitelist-ips.conf          [TO]=>  /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf...OK

REPO = https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master

Downloading [FROM]=>  [REPO]/setup-ngxblocker      [TO]=>  /usr/local/sbin/setup-ngxblocker...OK
Downloading [FROM]=>  [REPO]/update-ngxblocker     [TO]=>  /usr/local/sbin/update-ngxblocker...OK
Setting mode: 700 => /usr/local/sbin/install-ngxblocker
Setting mode: 700 => /usr/local/sbin/setup-ngxblocker
Setting mode: 700 => /usr/local/sbin/update-ngxblocker
```

```
setup-ngxblocker -x -e conf -c /usr/local/nginx/conf/ultimate-badbot-blocker -b /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d -v /usr/local/nginx/conf/conf.d -m /usr/local/nginx/conf/nginx.conf
Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf;            => /usr/local/nginx/conf/nginx.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf;  => /usr/local/nginx/conf/nginx.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/demodomain.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/demodomain.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/virtual.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/virtual.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/sub1.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/sub2.domain2.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/http2.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/http2.domain1.com.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf;           => /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf
inserting: include /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf;                => /usr/local/nginx/conf/conf.d/http2.domain1.com.ssl.conf

Whitelisting ip:  xxx.xxx.xxx.xxx   => /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf

Web directory not found ('/var/www'): not whitelisting domains.

Checking for missing includes:

Checking url: https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/include_filelist.txt

Nothing to update for directory: /usr/local/nginx/conf/ultimate-badbot-blocker
Nothing to update for directory: /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d
Nothing to update for directory: /usr/local/sbin
Setting mode: 700 => /usr/local/sbin/install-ngxblocker
Setting mode: 700 => /usr/local/sbin/setup-ngxblocker
Setting mode: 700 => /usr/local/sbin/update-ngxblocker
```

#### actual installed files

```
ls -lah /usr/local/sbin/ | grep ngxblocker
-rwx------   1 root root 9.5K Apr  2 16:17 install-ngxblocker
-rwx------   1 root root  13K Apr  2 16:27 setup-ngxblocker
-rwx------   1 root root  12K Apr  2 16:27 update-ngxblocker
```

```
ls -lah /usr/local/nginx/conf/ultimate-badbot-blocker
total 240K
drwxr-xr-x  3 root root   83 Apr  2 16:27 .
drwxr-xr-x. 8 root root 4.0K Apr  2 16:25 ..
-rw-------  1 root root 1.8K Apr  2 16:27 botblocker-nginx-settings.conf
drwxr-xr-x  2 root root 4.0K Apr  2 16:27 bots.d
-rw-------  1 root root 228K Apr  2 16:27 globalblacklist.conf
```
```
ls -lah /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/
total 44K
drwxr-xr-x 2 root root 4.0K Apr  2 16:27 .
drwxr-xr-x 3 root root   83 Apr  2 16:27 ..
-rw------- 1 root root 3.5K Apr  2 16:27 bad-referrer-words.conf
-rw------- 1 root root 2.4K Apr  2 16:27 blacklist-domains.conf
-rw------- 1 root root 7.0K Apr  2 16:27 blacklist-ips.conf
-rw------- 1 root root 3.3K Apr  2 16:27 blacklist-user-agents.conf
-rw------- 1 root root 2.1K Apr  2 16:27 blockbots.conf
-rw------- 1 root root 2.6K Apr  2 16:27 custom-bad-referrers.conf
-rw------- 1 root root 1.8K Apr  2 16:27 ddos.conf
-rw------- 1 root root 2.5K Apr  2 16:27 whitelist-domains.conf
-rw------- 1 root root 1.7K Apr  2 16:27 whitelist-ips.conf
```

## Testing

Run the following commands one by one from a terminal on another linux machine against your own domain name. substitute http://domain1.com in the examples below with your REAL domain name. Should respond with HTTP 200 status OK code

```
curl -I -A "Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.96 Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" http://domain1.com
```

```
curl -I -A "Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.96 Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" http://domain1.com
HTTP/1.1 200 OK
Date: Mon, 02 Apr 2018 17:54:54 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 6040
Last-Modified: Mon, 02 Apr 2018 16:20:00 GMT
Connection: keep-alive
Vary: Accept-Encoding
ETag: "5ac25830-1798"
Server: nginx centminmod
X-Powered-By: centminmod
Accept-Ranges: bytes
```

Should respond with: curl: (52) Empty reply from server

```
curl -I http://domain1.com -e http://100dollars-seo.com
```

```
curl -I http://domain1.com -e http://100dollars-seo.com
curl: (52) Empty reply from server
```

## Rate Limits

Rate limiting configuration settings are located in `/usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf` and `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf`

* `limit_req_zone` is setup with zone named `flood` with 90 requests/s rate limit with nodelay burst of 200 requests
* `limit_conn_zone` is setup with connection limit set in `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf` at 200 simultaneous IP connections

The global bad bot blacklisting is contained in `/usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf`. You can see the source master list at [https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/blob/master/conf.d/globalblacklist.conf](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/blob/master/conf.d/globalblacklist.conf).

Next to each entry is a number 0, 1, 2 or 3 which denote the following:

```
### Note that: 
### 0 = allowed - no limits
### 1 = allowed or rate limited less restrictive
### 2 = rate limited more
### 3 = block completely
```

```
cat /usr/local/nginx/conf/ultimate-badbot-blocker/botblocker-nginx-settings.conf
##############################################################################                                                                
#       _  __     _                                                          #
#      / |/ /__ _(_)__ __ __                                                 #
#     /    / _ `/ / _ \\ \ /                                                 #
#    /_/|_/\_, /_/_//_/_\_\                                                  #
#       __/___/      __   ___       __     ___  __         __                #
#      / _ )___ ____/ /  / _ )___  / /_   / _ )/ /__  ____/ /_____ ____      #
#     / _  / _ `/ _  /  / _  / _ \/ __/  / _  / / _ \/ __/  '_/ -_) __/      #
#    /____/\_,_/\_,_/  /____/\___/\__/  /____/_/\___/\__/_/\_\\__/_/         #
#                                                                            #
##############################################################################                                                                

# Version 1.1

#server_names_hash_bucket_size 128;
#server_names_hash_max_size 4096;
limit_req_zone $binary_remote_addr zone=flood:50m rate=90r/s;
limit_conn_zone $binary_remote_addr zone=addr:50m;

# ****************************************************************************
# NOTE: IF you are using a system like Nginx-Proxy from @JWilder
# ****************************************************************************
# Repo URL: https://github.com/jwilder/nginx-proxy
# You will need to comment out the first line here as follows. 
#     #server_names_hash_bucket_size 128;
# You will also need to modify the nginx.tmpl file to add the default include
#     include /etc/nginx/conf.d/*
# ****************************************************************************
```

```
cat /usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/ddos.conf
#######################################################################

### VERSION INFORMATION #
###################################################
### Version: V3.2017.01
### Updated: Sun Jan  29 11:35:32 SAST 2017
###################################################
### VERSION INFORMATION ##

##############################################################################                                                                
#       _  __     _                                                          #
#      / |/ /__ _(_)__ __ __                                                 #
#     /    / _ `/ / _ \\ \ /                                                 #
#    /_/|_/\_, /_/_//_/_\_\                                                  #
#       __/___/      __   ___       __     ___  __         __                #
#      / _ )___ ____/ /  / _ )___  / /_   / _ )/ /__  ____/ /_____ ____      #
#     / _  / _ `/ _  /  / _  / _ \/ __/  / _  / / _ \/ __/  '_/ -_) __/      #
#    /____/\_,_/\_,_/  /____/\___/\__/  /____/_/\___/\__/_/\_\\__/_/         #
#                                                                            #
##############################################################################                                                                

# Author: Mitchell Krog <mitchellkrog@gmail.com> - https://github.com/mitchellkrogza/

# Include this in a vhost file within a server {} block using and include statement like below

# server {
#                       #Config stuff here
#                       include /etc/nginx/bots.d/blockbots.conf
#                       include /etc/nginx/bots.d/ddos.conf
#                       #Other config stuff here
#                }

#######################################################################

limit_conn addr 200;
limit_req zone=flood burst=200 nodelay;
```

## White Listing

Domain and IP whitelisting are set in respective include files at:

* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-domains.conf`
* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf`

## Black Listing

Domain and IP blacking are set in respective include files at:

* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-domains.conf`
* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-ips.conf`

## Bad Referrers

custom bad referrers and bad referrer words are set in respective include files at:

* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/custom-bad-referrers.conf`
* `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/bad-referrer-words.conf`

## Blockbots.conf

The bad bot blocking logic is located in `/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blockbots.conf`

```
# Author: Mitchell Krog <mitchellkrog@gmail.com> - https://github.com/mitchellkrogza/

### VERSION INFORMATION #
###################################################
### Version: V3.2017.02
### Updated: Mon Aug  21 11:29:32 SAST 2017
###################################################
### VERSION INFORMATION ##


##############################################################################                                                                
#       _  __     _                                                          #
#      / |/ /__ _(_)__ __ __                                                 #
#     /    / _ `/ / _ \\ \ /                                                 #
#    /_/|_/\_, /_/_//_/_\_\                                                  #
#       __/___/      __   ___       __     ___  __         __                #
#      / _ )___ ____/ /  / _ )___  / /_   / _ )/ /__  ____/ /_____ ____      #
#     / _  / _ `/ _  /  / _  / _ \/ __/  / _  / / _ \/ __/  '_/ -_) __/      #
#    /____/\_,_/\_,_/  /____/\___/\__/  /____/_/\___/\__/_/\_\\__/_/         #
#                                                                            #
##############################################################################                                                                

# Include this in a vhost file within a server {} block using and include statement like below

# server {
#                       #Config stuff here
#                       include /etc/nginx/bots.d/blockbots.conf
#                       include /etc/nginx/bots.d/ddos.conf
#                       #Other config stuff here
#                }

#######################################################################

# BOTS
# ****
#limit_conn bot1_connlimit 100;
limit_conn bot2_connlimit 10;
#limit_req  zone=bot1_reqlimitip burst=50;
limit_req  zone=bot2_reqlimitip burst=10;
if ($bad_bot = '3') {
  return 444;
  }

# BAD REFER WORDS
# ***************
if ($bad_words) {
  return 444;
}


# REFERERS
# ********
if ($bad_referer) {
  return 444;
}

# IP BLOCKS
# *********
if ($validate_client) {
  return 444;
}

#######################################################################
```

## Customize Configurations

You can now customize any of the following files below to suit your environment or requirements. These include files never get modified during an update using the auto update script `/usr/local/sbin/update-ngxblocker` outlined in above [cronjob](#cronjob) section so whatever customizations you do here will never be overwritten during an update.

```
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-ips.conf
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/whitelist-domains.conf
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-user-agents.conf
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/blacklist-ips.conf
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/bad-referrer-words.conf
/usr/local/nginx/conf/ultimate-badbot-blocker/bots.d/custom-bad-referrers.conf
```

## globalblacklist.conf

Nginx Ultimate Bad Bot Blocker is installed at `/usr/local/nginx/conf/ultimate-badbot-blocker` and where the global bad bot blacklisting is contained in `/usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf`. You can see the source master list at [https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/blob/master/conf.d/globalblacklist.conf](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/blob/master/conf.d/globalblacklist.conf).

Example of Good bots allowed

```
# ***********************************************
# Allow Good User-Agent Strings We Know and Trust
# ***********************************************

# START GOOD BOTS ### DO NOT EDIT THIS LINE AT ALL ###
  "~*\badidxbot\b"    0;
  "~*\bAdsBot-Google\b"   0;
  "~*\baolbuild\b"    0;
  "~*\bbingbot\b"   0;
  "~*\bbingpreview\b"   0;
  "~*\bDoCoMo\b"    0;
  "~*\bduckduckgo\b"    0;
  "~*\bfacebookexternalhit\b"   0;
  "~*\bFeedfetcher-Google\b"    0;
  "~*\bGooglebot\b"   0;
  "~*\bGooglebot-Image\b"   0;
  "~*\bGooglebot-Mobile\b"    0;
  "~*\bGooglebot-News\b"    0;
  "~*\bGooglebot/Test\b"    0;
  "~*\bGooglebot-Video\b"   0;
  "~*\bGoogle-HTTP-Java-Client\b"   0;
  "~*\bGravityscan\b"   0;
  "~*\bgsa-crawler\b"   0;
  "~*\bJakarta\ Commons\b"    0;
  "~*\bKraken/0.1\b"    0;
  "~*\bLinkedInBot\b"   0;
  "~*\bMediapartners-Google\b"    0;
  "~*\bmsnbot\b"    0;
  "~*\bmsnbot-media\b"    0;
  "~*\bSAMSUNG\b"   0;
  "~*\bSlackbot\b"    0;
  "~*\bSlackbot-LinkExpanding\b"    0;
  "~*\bslurp\b"   0;
  "~*\bteoma\b"   0;
  "~*\bTwitterBot\b"    0;
  "~*\bWordpress\b"   0;
  "~*\byahoo\b"   0;
# END GOOD BOTS ### DO NOT EDIT THIS LINE AT ALL ###
```

User Agent strings allowed but rated limited

```
# ***************************************************
# User-Agent Strings Allowed Through but Rate Limited
# ***************************************************

# Some people block libwww-perl, it used widely in many valid (non rogue) agents
# I allow libwww-perl as I use it for monitoring systems with Munin but it is rate limited

# START ALLOWED BOTS ### DO NOT EDIT THIS LINE AT ALL ###
  "~*\bjetmon\b"    1;
  "~*\blibwww-perl\b"   1;
  "~*\bLynx\b"    1;
  "~*\bmunin\b"   1;
  "~*\bPresto\b"    1;
  "~*\bWget/1.15\b"   1;
# END ALLOWED BOTS ### DO NOT EDIT THIS LINE AT ALL ###
```

User Agent strings allowed but more aggressive/restrictive rate limiting

```
# **************************************************************
# Rate Limited User-Agents who get a bit aggressive on bandwidth
# **************************************************************

# START LIMITED BOTS ### DO NOT EDIT THIS LINE AT ALL ###
  "~*\bAlexa\b"   2;
  "~*\barchive.org\b"   2;
  "~*\bBaidu\b"   2;
  "~*\bBUbiNG\b"    2;
  "~*\bFlipboardProxy\b"    2;
  "~*\bia_archiver\b"   2;
  "~*\bMSIE\ 7.0\b"   2;
  "~*\bProximic\b"    2;
  "~*\bR6_CommentReader\b"    2;
  "~*\bR6_FeedFetcher\b"    2;
  "~*\bRED/1\b"   2;
  "~*\bRPT-HTTPClient\b"    2;
  "~*\bsfFeedReader/0.9\b"    2;
  "~*\bSpaidu\b"    2;
  "~*\bUptimeRobot/2.0\b"   2;
  "~*\bYandexBot\b"   2;
  "~*\bYandexImages\b"    2;
# END LIMITED BOTS ### DO NOT EDIT THIS LINE AT ALL ###
```

bad bots listed in `/usr/local/nginx/conf/ultimate-badbot-blocker/globalblacklist.conf` section under title:

```
# *********************************************
# Bad User-Agent Strings That We Block Outright
# *********************************************
```

## ngxtop

You can use [ngxtop](https://github.com/lebinh/ngxtop) to analyse your Centmin Mod Nginx access logs and keep an eye on Nginx HTTP 444 status codes and also other user agent strings, HTTP status codes etc as outlined [here](https://community.centminmod.com/threads/ngxtop-real-time-metrics-for-nginx.285/).

CentOS 7 ngxtop install

```
yum -y install python-pip
pip install --upgrade pip
pip install ngxtop
```

Check for HTTP 444 status codes `/home/nginx/domains/domain1.com/log/access.log` access log

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow
```

Only one entry for 444 exists due to test `domain1.com` test command `curl -I http://domain1.com -e http://100dollars-seo.com` only ran once

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow
running for 0 seconds, 1 records processed: 2434.30 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|       1 |            0.000 |     0 |     0 |     1 |     0 |

Detailed:
| request_path   |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|----------------+---------+------------------+-------+-------+-------+-------|
| /              |       1 |            0.000 |     0 |     0 |     1 |     0 |
```

alternative use native `-i 'status == 444'` flag

```
cat /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow -i 'status == 444'
```

```
cat /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow -i 'status == 444'
running for 0 seconds, 1 records processed: 369.97 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|       1 |            0.000 |     0 |     0 |     1 |     0 |

Detailed:
| request_path   |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|----------------+---------+------------------+-------+-------+-------+-------|
| /              |       1 |            0.000 |     0 |     0 |     1 |     0 |
```

print request url, HTTP Status code and http user agent

```
cat /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow -i 'status == 444' print request status http_user_agent
```

```
cat /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow -i 'status == 444' print request status http_user_agent
running for 0 seconds, 1 records processed: 335.92 req/sec

request, status, http_user_agent:
| request         |   status | http_user_agent   |
|-----------------+----------+-------------------|
| HEAD / HTTP/1.1 |      444 | curl/7.29.0       |
```

group by remote IP address via `--group-by remote_addr`

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow --group-by remote_addr
```

output where `192.168.0.1` is the visitor ip address

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow --group-by remote_addr
running for 0 seconds, 1 records processed: 1597.22 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|       1 |            0.000 |     0 |     0 |     1 |     0 |

Detailed:
| remote_addr   |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------------+---------+------------------+-------+-------+-------+-------|
| 192.168.0.1 |       1 |            0.000 |     0 |     0 |     1 |     0 |
```

group by user agent string via `--group-by http_user_agent`

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow --group-by http_user_agent
```

the test command was run via curl hence curl user agent string

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | ngxtop --no-follow --group-by http_user_agent
running for 0 seconds, 1 records processed: 1402.31 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|       1 |            0.000 |     0 |     0 |     1 |     0 |

Detailed:
| http_user_agent   |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|-------------------+---------+------------------+-------+-------+-------+-------|
| curl/7.29.0       |       1 |            0.000 |     0 |     0 |     1 |     0 |
```

filter by specific date using grep i.e. April 2nd would filter by `'02/Apr'`

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | grep '02/Apr' | ngxtop --no-follow --group-by http_user_agent
```

```
grep ' 444 ' /home/nginx/domains/domain1.com/log/access.log | grep '02/Apr' | ngxtop --no-follow --group-by http_user_agent
running for 0 seconds, 1 records processed: 1742.54 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|       1 |            0.000 |     0 |     0 |     1 |     0 |

Detailed:
| http_user_agent   |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|-------------------+---------+------------------+-------+-------+-------+-------|
| curl/7.29.0       |       1 |            0.000 |     0 |     0 |     1 |     0 |
```