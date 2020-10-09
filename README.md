"# checkpoint-api-gateway-skybox"


Notes
----------
useradd nginx
yum install openssl openssl-devel  gcc make automake autoconf libtool pcre pcre-devel libxml2 libxml2-devel curl curl-devel httpd-devel yajl-devel.x86_64


mkdir nginx_modsecurity
cd nginx_modsecurity/
wget https://nginx.org/download/nginx-1.19.3.tar.gz
wget https://www.modsecurity.org/tarball/2.9.3/modsecurity-2.9.3.tar.gz
tar -xvzf modsecurity-2.9.3.tar.gz
tar -xvzf nginx-1.19.3.tar.gz
cd modsecurity-2.9.3
./configure --enable-standalone-module
make
cd ..
cd nginx-1.19.3
./configure --add-module=../modsecurity-2.9.3/nginx/modsecurity --user=nginx --group=nginx --prefix=/opt/nginx --with-http_gzip_static_module --with-http_stub_status_module --with-http_ssl_module --with-pcre --with-file-aio --with-http_realip_module --without-http_scgi_module --without-http_uwsgi_module --with-http_realip_module


# vi /usr/lib/systemd/system/nginx.service

[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/opt/nginx/logs/nginx.pid
ExecStartPre=/opt/nginx/sbin/nginx -t
ExecStart=/opt/nginx/sbin/nginx
ExecReload=/opt/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target






cp /root/nginx_modsecurity/modsecurity-2.9.3/modsecurity.conf-recommended /opt/nginx/conf/modsecurity.conf
cp /root/nginx_modsecurity/modsecurity-2.9.3/unicode.mapping /opt/nginx/conf/

cd /opt/nginx/conf
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout cert.key -out cert.crt

edit /opt/nginx/nginx.conf
add/amend lines:
#===========================================================================
# HTTPS server
 #
 server {
     listen       443 ssl;
     server_name  localhost;

     ssl_certificate      cert.crt;
     ssl_certificate_key  cert.key;

     ssl_session_cache    shared:SSL:1m;
     ssl_session_timeout  5m;

     ssl_ciphers  HIGH:!aNULL:!MD5;
     ssl_prefer_server_ciphers  on;

     location / {
             proxy_pass https://10.1.2.3;
             ModSecurityEnabled on;
             ModSecurityConfig modsecurity.conf;
#===========================================================================


edit /opt/nginx/conf/modsecurity.conf
add lines:
#===========================================================================
#############################################
## Checkpoint Custom Rules
############################################

# Whitelisting URLs
SecRule REQUEST_URI "/login$"                            "id:101,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-gateways-and-servers$"        "id:102,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-generic-objects$"             "id:103,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-application-sites$"           "id:104,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-application-site-categories$" "id:105,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-package$"                     "id:106,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-access-rulebase$"             "id:107,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-nat-rulebase$"                "id:108,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-service-groups$"              "id:109,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-hosts$"                       "id:110,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-networks$"                    "id:111,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-groups$"                      "id:112,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-groups-with-exclusion$"       "id:113,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-data-center-objects$"         "id:114,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-simple-gateways$"             "id:115,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/show-task$"                        "id:116,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/discard$"                          "id:117,phase:1,pass,ctl:ruleRemoveById=600-699"
SecRule REQUEST_URI "/logout$"                           "id:118,phase:1,pass,ctl:ruleRemoveById=600-699"


# Whitelist run-script URL + Whitelist script command
SecRule REQUEST_URI "/run-script$" "id:119,phase:2,chain"
SecRule ARGS:script "netstat -rne" "ctl:ruleRemoveById=600-699"


# Drop Rules
SecRule ARGS:script ".*" "id:600,phase:2,log,deny,status:403"
SecRule REQUEST_URI ".*" "id:601,phase:2,log,deny,status:403"
##########################################


#===========================================================================



systemctl daemon-reload
systemctl start nginx
