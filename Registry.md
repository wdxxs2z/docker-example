# docker registry_v2
docker registry_v2的搭建和排错文档，nginx+registry源码搭建，有别于网上所有的搭建方法，方便registry日后调优

## 搭建过程

* CA证书的制作(openssl)
* nginx的搭建及配置
* registry源码编译及配置
* 验证及排错

### CA证书的制作
1.首先我们去**/etc/ssl/openssl.cnf**下修改下参数，必须在生成证书之前修改，否则无意义</br>

		[ CA_default ]
		dir             = /etc/ssl/demoCA       # Where everything is kept
		certs           = $dir/certs            # Where the issued certs are kept
		crl_dir         = $dir/crl              # Where the issued crl are kept
		database        = $dir/index.txt        # database index file.
		#unique_subject = no                    # Set to 'no' to allow creation of
												# several ctificates with same subject.
		new_certs_dir   = $dir/newcerts         # default place for new certs.

		certificate     = $dir/certs/cacert.pem         # The CA certificate
		serial          = $dir/serial           # The current serial number
		crlnumber       = $dir/crlnumber        # the current crl number
												# must be commented out to leave a V1 CRL
		crl             = $dir/crl.pem          # The current CRL
		private_key     = $dir/private/cakey.pem# The private key
		RANDFILE        = $dir/private/.rand    # private random number file
		
		[ v3_req ]

		# Extensions to add to a certificate request

		basicConstraints = CA:FALSE
		
		keyUsage = nonRepudiation, digitalSignature, keyEncipherment
		
		#这个很重要，否则在后面会报registry endpoint...x509: cannot validate certificate for ... because it doesn't contain any IP SANs
		subjectAltName=IP:192.168.172.150
		
2.制作证书</br>

证书的配置文件都在**Ubuntu**的路径在/etc/ssl下</br>
		
		cd /etc/ssl
		mkdir demoCA demoCA/certs demoCA/crl demoCA/newcerts demoCA/private
		touch /etc/ssl/demoCA/index.txt
		echo 01 > /etc/ssl/demoCA/serial
		cd /etc/ssl/demoCA
		openssl req -newkey rsa:4096 -nodes -sha256 -keyout cakey.pem -x509 -days 365 -out cacert.pem
		mv cacert.pem certs/ && mv cakey.pem private/
		
注意这里的domain设置成自己的域名即可，比如我的是*.192.168.172.150.xip.io</br>

		You are about to be asked to enter information that will be incorporated
		into your certificate request.
		What you are about to enter is what is called a Distinguished Name or a DN.
		There are quite a few fields but you can leave some blank
		For some fields there will be a default value,
		If you enter '.', the field will be left blank.
		-----
		Country Name (2 letter code) [XX]:CN
		State or Province Name (full name) []:beijing
		Locality Name (eg, city) [Default City]:beijing
		Organization Name (eg, company) [Default Company Ltd]:self
		Organizational Unit Name (eg, section) []:
		Common Name (eg, your name or your server's hostname) []:*.192.168.172.150.xip.io
		Email Address []:jackyuan@126.com
		
OK,至此，根证书等制作完成

### nginx的搭建及配置

1.选择版本安装,最好是高版本，否则add header功能没法使用</br>

		cd ~
		wget http://nginx.org/download/nginx-1.9.4.tar.gz
		tar zxvf nginx-1.9.4.tar.gz
		cd ./nginx-1.4.6 && \
		./configure --user=www --group=www --prefix=/opt/nginx \
		--with-pcre \
		--with-http_stub_status_module \
		--with-http_ssl_module \
		--with-http_addition_module  \
		--with-http_realip_module \
		--with-http_flv_module && \
		make && \
		make install
		
2.生成nginx的ssl证书,并加入进openssl本身的证书数据库</br>

		mkdir -p /etc/nginx/ssl
		cd /etc/nginx/ssl
		openssl genrsa -out nginx.key 4096
		
		openssl req -new -key nginx.key -out nginx.csr
		#上面这一步的配置要和跟设置的一样，尤其是domain那块
		openssl ca -in nginx.csr -out nginx.crt
		
在这里如果不在之前配置好CA的配置，则会出现demoCA无法打开等错误，所以要注意。</br>

3.生成htpassword,用户名和密码都为admin

		htpasswd -cb /opt/nginx/conf/.htpasswd admin admin

4.修改nginx配置

		user  www www;
		worker_processes  auto;

		error_log   /var/log/nginx_error.log error;
		#error_log  logs/error.log  notice;
		#error_log  logs/error.log  info;

		#pid        logs/nginx.pid;


		worker_rlimit_nofile 51200;

		events {
			use epoll;
			worker_connections  51200;
			multi_accept on;
		}

		http {
			include       mime.types;
			default_type  application/octet-stream;

			log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
							  '$status $body_bytes_sent "$http_referer" '
							  '"$http_user_agent" "$http_x_forwarded_for"';

			access_log  /var/log/nginx_access.log  main;

			server_names_hash_bucket_size 128;
			client_header_buffer_size 32k;
			large_client_header_buffers 4 32k;

			sendfile        on;
			tcp_nopush     on;
			tcp_nodelay    on;

			#keepalive_timeout  0;
			keepalive_timeout  65;

			#gzip  on;
		   
			upstream registry {
				server 192.168.172.150:5000;
			}    

			server {
				listen       443;
				server_name  192.168.172.150;
				
				ssl          on;
				ssl_certificate /etc/nginx/ssl/nginx.crt;
				ssl_certificate_key /etc/nginx/ssl/nginx.key;

				client_max_body_size 0;

				chunked_transfer_encoding on;

				location /v2/ {
				  auth_basic "Registry realm";
				  auth_basic_user_file /opt/nginx/conf/.htpasswd;
				  add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;
				  
				  proxy_pass                          http://registry;
				  proxy_set_header  Host              \$http_host;   # required for docker client's sake
				  proxy_set_header  X-Real-IP         \$remote_addr; # pass on real client's IP
				  proxy_set_header  X-Forwarded-For   \$proxy_add_x_forwarded_for;
				  proxy_set_header  X-Forwarded-Proto $scheme;
				  proxy_read_timeout                  900;
				}

		#        location / {
		#            auth_basic "docker registry";
		#            auth_basic_user_file /opt/nginx/conf/.htpasswd;
		#            
		#            root   html;
		#            index  index.html index.htm;
		#
		#            proxy_pass   http://registry;
		#            proxy_set_header Host $http_host;
		#            proxy_set_header X-Real-IP $remote_addr;
		#            proxy_set_header Authorization "";

		#            client_body_buffer_size 128k;
		#            proxy_connect_timeout   90;
		#            proxy_send_timeout      90;
		#            proxy_read_timeout      90;
		#            proxy_buffer_size       8k;
		#            proxy_buffers        4 32k;
		#            proxy_busy_buffers_size 64k;
		#            proxy_temp_file_write_size 64k;
		#        }

		#        location /_ping {
		#         auth_basic off;
		#          proxy_pass http://registry;
		#        }

		#        location /v1/_ping {
		#         auth_basic off;
		#          proxy_pass http://registry;
		#        }

				error_page   500 502 503 504  /50x.html;
				location = /50x.html {
					root   html;
				}
			}
		}
		
这里要说明两个地方：proxy_set_header  X-Forwarded-Proto $scheme;注意这里没有\$scheme，如果配置则会报：</br>
**unsupported protocol scheme ""**

还有一个地方：如果之前有配置过V1的，可以保留，但最好分离出来，我是直接注掉了。

5.验证及启动启动nginx</br>

		/opt/nginx/sbin/nginx -t
		/opt/nginx/sbin/nginx -s /opt/nginx/conf/nginx.conf
		
### registry源码编译及配置
1.checkout 源码：</br>

		git clone https://github.com/docker/distribution
		cd distribution
		git checkout v2.1.1
		godep restore ./...
		
2.安装registry,这里图省事，要编译重新设定下gopath即可</br>

		go get github.com/docker/distribution
		
3.配置registry,config-example.yml</br>

		version: 0.1
		log:
		  fields:
			service: registry
		storage:
			cache:
				layerinfo: inmemory
			filesystem:
				rootdirectory: /home/jojo/registry
		http:
			addr: :5000
			secret: admin
		#    tls: 
		#      certificate: /etc/ssl/demoCA/certs/cacert.pem
		#      key: /etc/ssl/demoCA/private/cakey.pem
		#proxy:
		#  remoteurl: https://api.192.168.172.150.xip.io
		#  username: admin
		#  password: admin
		
这里注意我注掉的部分，因为前方已经有一层代理了，我们这里就没有必要设置tls了，否则，后端 会报**tls: first record does not look like a TLS handshake**</br>

4.配置docker,增加一行</br>

		vi /etc/default/docker
		DOCKER_OPTS="--insecure-registry api.192.168.172.150.xip.io --tlsverify --tlscacert /etc/ssl/demoCA/certs/cacert.pem"
		
5.启动docker和registry</br>

		service docker start
		registry /home/jojo/register/config-example.yml
		
### 验证及排错

1.验证联通性：</br>

		curl -i -k -v https://admin:admin@api.192.168.172.150.xip.io/v2/
		
		* Hostname was NOT found in DNS cache
		*   Trying 192.168.172.150...
		* Connected to api.192.168.172.150.xip.io (192.168.172.150) port 443 (#0)
		* successfully set certificate verify locations:
		*   CAfile: none
		  CApath: /etc/ssl/certs
		* SSLv3, TLS handshake, Client hello (1):
		* SSLv3, TLS handshake, Server hello (2):
		* SSLv3, TLS handshake, CERT (11):
		* SSLv3, TLS handshake, Server key exchange (12):
		* SSLv3, TLS handshake, Server finished (14):
		* SSLv3, TLS handshake, Client key exchange (16):
		* SSLv3, TLS change cipher, Client hello (1):
		* SSLv3, TLS handshake, Finished (20):
		* SSLv3, TLS change cipher, Client hello (1):
		* SSLv3, TLS handshake, Finished (20):
		* SSL connection using ECDHE-RSA-AES256-GCM-SHA384
		* Server certificate:
		*        subject: C=CN; ST=beijing; O=self; OU=self; CN=*.192.168.172.150.xip.io; emailAddress=jack@126.com
		*        start date: 2015-09-18 13:54:11 GMT
		*        expire date: 2016-09-17 13:54:11 GMT
		*        issuer: C=CN; ST=beijing; L=beijing; O=self; OU=self; CN=*.192.168.172.150.xip.io; emailAddress=jack@126.com
		*        SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
		* Server auth using Basic with user 'admin'
		> GET /v2/ HTTP/1.1
		> Authorization: Basic YWRtaW46YWRtaW4=
		> User-Agent: curl/7.35.0
		> Host: api.192.168.172.150.xip.io
		> Accept: */*
		> 
		< HTTP/1.1 200 OK
		HTTP/1.1 200 OK
		* Server nginx/1.9.4 is not blacklisted
		< Server: nginx/1.9.4
		Server: nginx/1.9.4
		< Date: Fri, 18 Sep 2015 17:02:00 GMT
		Date: Fri, 18 Sep 2015 17:02:00 GMT
		< Content-Type: application/json; charset=utf-8
		Content-Type: application/json; charset=utf-8
		< Content-Length: 2
		Content-Length: 2
		< Connection: keep-alive
		Connection: keep-alive
		< Docker-Distribution-Api-Version: registry/2.0
		Docker-Distribution-Api-Version: registry/2.0
		< Docker-Distribution-Api-Version: registry/2.0
		Docker-Distribution-Api-Version: registry/2.0
		
2.验证docker login</br>

		docker login -u admin -p admin -e jackyuan@126 https://api.192.168.172.150.xip.io/v2/
		
		WARNING: login credentials saved in /root/.docker/config.json
		Login Succeeded
		
3.push镜像到私有registry</br>

		docker tag 91e54dfb1179 api.192.168.172.150.xip.io/ubuntu:trusty
		docker push api.192.168.172.150.xip.io/ubuntu:trusty
		
		The push refers to a repository [api.192.168.172.150.xip.io/ubuntu] (len: 1)
		91e54dfb1179: Image already exists 
		d74508fb6632: Image successfully pushed 
		c22013c84729: Image successfully pushed 
		d3a1f33e8a5a: Image successfully pushed 
		Digest: sha256:a731c12a4d21af384c4659666f177cd1e871646b95b9440d709ec4ee176145b2
		
4.查看是否上传</br>

		curl -i -k -v https://admin:admin@api.192.168.172.150.xip.io/v2/_catalog
		
		{"repositories":["ubuntu"]}
		
		cd /home/jojo/registry/docker/registry/v2 && ls
		blobs  repositories
		
这里就不往里面看了，深入registry后可以继续看里面的数据结构。