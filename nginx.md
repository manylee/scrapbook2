# restart, stop, start

~~~bash
ps aux | egrep '(PID|nginx)'
#kill main one
sudo  /opt/nginx/sbin/nginx

#or

sudo /etc/init.d/nginx restart

# the best

sudo service nginx restart
                   status
                   start
                   stop
~~~


# several servers NginX

you can have several sites hosted managed by Nginx

```
   server {
        listen 80 default;  # he will have the port 80 by default
        server_name foo1
        root /home/tomi/projects/foo1;
        index  index.html index.htm;
    }
    
    server {
        listen 80;
        listen 81;          # localshost:81 will trigger this
        server_name foo2
        root /home/tomi/projects/foo2;
        index  index.html index.htm;
    }
```


# nginx with multiple Self signed certificates for same IP 

stolen from: https://www.digitalocean.com/community/articles/how-to-set-up-multiple-ssl-certificates-on-one-ip-with-nginx-on-ubuntu-12-04
http://nginx.org/en/docs/http/configuring_https_servers.html#sni

first you need to check if your nginx supports SNI (single name identifier)

```bash
nginx -V 

# or   /opt/nginx/sbin/nginx -V

# and it should say  TLS SNI support enabled

```

than you can generate certificates on separate domains


```bash
# location depending if you have repo nginx(/etc/nginx) or you built it yourself(/opt/nginx)

mkdir -p /etc/nginx/ssl/my_appication_name.com/
mkdir -p /etc/nginx/ssl/my_appication_name.london/


cd /etc/nginx/ssl/my_appication_name.com/
sudo openssl genrsa -des3 -out server.key 1024          # generate private server key with password
sudo openssl req -new -key server.key -out server.csr   # generate signing request form key
                                                        # this will promt you to fill in some inforation
                                                        # about "sign" company
                                                        # most import is the Common Name !!! 
# Common Name []:my_application_name.com                # enter here the official name, domain or IP 

sudo cp server.key server.key.org
sudo openssl rsa -in server.key.org -out server.key     # remove password from key

sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt  # sign the certificate (365 days)


# same for the other tld
cd /etc/nginx/ssl/my_appication_name.london/
sudo openssl genrsa -des3 -out server.key 1024 
sudo openssl req -new -key server.key -out server.csr
# Common Name []:my_application_name.london
sudo cp server.key server.key.org
sudo openssl rsa -in server.key.org -out server.key 
sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

```

now in your virtual host 

```bash
sudo vim /etc/nginx/sites-available/my_appication_name.com


server {
        listen   443;                       #this will tell to listen to ssl
        server_name my_appication_name.com;

        root /usr/share/nginx/www;
        index index.html index.htm;

        ssl on;
        ssl_certificate /etc/nginx/ssl/my_appication_name.com/server.crt;   # use the .crt not the .csr
        ssl_certificate_key /etc/nginx/ssl/example.org/server.key;          # the one without the password
}

sudo vim /etc/nginx/sites-available/my_appication_name.london


server {
        listen   443;                 
        server_name my_appication_name.london;

        root /usr/share/nginx/www;
        index index.html index.htm;

        ssl on;
        ssl_certificate /etc/nginx/ssl/my_appication_name.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.org/server.key;
}
```

you have to have loading of those virtual host fileas activated in your main nginx config `/etc/nginx/conf/nginx.conf` !!

or 

```bash
sudo ln -s /etc/nginx/sites-available/my_appication_name.com /etc/nginx/sites-enabled/my_appication_name.com
sudo ln -s /etc/nginx/sites-available/my_appication_name.london /etc/nginx/sites-enabled/my_appication_name.london
```

finally restartyour nginx

#restrict access

restrict access: http://wiki.nginx.org/NginxHttpAuthBasicModule#auth_basic_user_file



# nginx config files

The main nginx config file is `/etc/nginx/nginx.conf` but keep this for system preferences. Inside of it you have
a line:

```
include /etc/nginx/sites-enabled/*; 
```

...which include configurations for your sites. 

NginX by default takes the `/etc/nginx/sites-enabled/default` one as the main site.



# nginx with unicorn rails

as seen in: http://railscasts.com/episodes/293-nginx-unicorn , https://github.com/railscasts/293-nginx-unicorn

```bash
# /etc/nginx/sites-enabled/default

# this translates to to the proxypas http://unicorn
upstream unicorn {

  # unicorn server is pointing socket to /tmp/unicorn.my_cool_app_name.sock
  server unix:/tmp/unicorn.my_cool_app_name.sock fail_timeout=0;                                                           
}
                                                                                                                      
server {

  server_name  my_cool_app_name;                                                                                           

  # public & precompiled assets
  root /home/deploy/apps/app_name/current/public;   
  
  # log
  access_log  /var/log/nginx/localhost.access.log;                                                                    
  
  # this is telling NginX to try to fetch
  #   
  #  1  /home/deploy/.../current/public/index.html
  #  2  /home/deploy/.../current/public/whatever_requested
  #  3  unicorn location, wich eqls to unicorn socket
  #
  try_files $uri/index.html $uri @unicorn;                                                                            
  
  # unicorn location
  location @unicorn {                                                                                                 
    proxy_pass http://unicorn;
    
    # if you want to try this with webrick you can do 
    #   
    #   proxy_pass http:/0.0.0.0:3000
    #
  }                                                                                                                   
  
  # use error pages from public/500.html for this statuses
  error_page 500 502 503 504 /500.html;                                                                               
}  

```


# simple static index.html file configuration

```
# /etc/nginx/sites-enabled/default
server {
  # if you running Varden you may need option below 
  # listen   80;

  server_name  my_app_name;

  access_log  /var/log/nginx/localhost.access.log;

  location / {
    root /home/deploy/apps/my_app_name/;
    index  index.html index.htm;
  }
}
```

#x forwarded for

!!!!!unfinished

realy good example ngnix.conf file is at http://brainspl.at/nginx.conf.txt
http://forum.nginx.org/read.php?2,97154
https://www.chiliproject.org/boards/1/topics/545

```bash
#log_format  main  '$remote_addr - $remote_user [$time_local]"$request" '
#                  '$status $body_bytes_sent "$http_referer" '
#                  '"$http_user_agent" "$http_x_forwarded_for"';

```

!!!! unfinished







netstat -anp|grep 3000  
[emerg]: bind() to unix:/tmp/nginx-staging.sock failed (98: Address already in use)