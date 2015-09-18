# docker registry_v2
docker registry_v2�Ĵ���Ŵ��ĵ���nginx+registryԴ�����б����������еĴ����������registry�պ����

## �����

* CA֤�������(openssl)
* nginx�Ĵ������
* registryԴ����뼰����
* ��֤���Ŵ�

### CA֤�������
1.��������ȥ**/etc/ssl/openssl.cnf**���޸��²���������������֤��֮ǰ�޸ģ�����������</br>

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
		
		#�������Ҫ�������ں���ᱨregistry endpoint...x509: cannot validate certificate for ... because it doesn't contain any IP SANs
		subjectAltName=IP:192.168.172.150
		
2.����֤��</br>

֤��������ļ�����**Ubuntu**��·����/etc/ssl��</br>
		
		cd /etc/ssl
		mkdir demoCA demoCA/certs demoCA/crl demoCA/newcerts demoCA/private
		touch /etc/ssl/demoCA/index.txt
		echo 01 > /etc/ssl/demoCA/serial
		cd /etc/ssl/demoCA
		openssl req -newkey rsa:4096 -nodes -sha256 -keyout cakey.pem -x509 -days 365 -out cacert.pem
		mv cacert.pem certs/ && mv cakey.pem private/
		
ע�������domain���ó��Լ����������ɣ������ҵ���*.192.168.172.150.xip.io</br>

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
		
OK,���ˣ���֤����������

### nginx�Ĵ������

1.ѡ��汾��װ,����Ǹ߰汾������add header����û��ʹ��</br>

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
		
2.����nginx��ssl֤��,�������openssl�����֤�����ݿ�</br>

		mkdir -p /etc/nginx/ssl
		cd /etc/nginx/ssl
		openssl genrsa -out nginx.key 4096
		
		openssl req -new -key nginx.key -out nginx.csr
		#������һ��������Ҫ�͸����õ�һ����������domain�ǿ�
		openssl ca -in nginx.csr -out nginx.crt
		
�������������֮ǰ���ú�CA�����ã�������demoCA�޷��򿪵ȴ�������Ҫע�⡣</br>

3.����htpassword,�û��������붼Ϊadmin

		htpasswd -cb /opt/nginx/conf/.htpasswd admin admin

4.�޸�nginx����

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
		
����Ҫ˵�������ط���proxy_set_header  X-Forwarded-Proto $scheme;ע������û��\$scheme�����������ᱨ��</br>
**unsupported protocol scheme ""**

����һ���ط������֮ǰ�����ù�V1�ģ����Ա���������÷������������ֱ��ע���ˡ�

5.��֤����������nginx</br>

		/opt/nginx/sbin/nginx -t
		/opt/nginx/sbin/nginx -s /opt/nginx/conf/nginx.conf
		
### registryԴ����뼰����
1.checkout Դ�룺</br>

		git clone https://github.com/docker/distribution
		cd distribution
		git checkout v2.1.1
		godep restore ./...
		
2.��װregistry,����ͼʡ�£�Ҫ���������趨��gopath����</br>

		go get github.com/docker/distribution
		
3.����registry,config-example.yml</br>

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
		
����ע����ע���Ĳ��֣���Ϊǰ���Ѿ���һ������ˣ����������û�б�Ҫ����tls�ˣ����򣬺�� �ᱨ**tls: first record does not look like a TLS handshake**</br>

4.����docker,����һ��</br>

		vi /etc/default/docker
		DOCKER_OPTS="--insecure-registry api.192.168.172.150.xip.io --tlsverify --tlscacert /etc/ssl/demoCA/certs/cacert.pem"
		
5.����docker��registry</br>

		service docker start
		registry /home/jojo/register/config-example.yml
		
### ��֤���Ŵ�

1.��֤��ͨ�ԣ�</br>

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
		
2.��֤docker login</br>

		docker login -u admin -p admin -e jackyuan@126 https://api.192.168.172.150.xip.io/v2/
		
		WARNING: login credentials saved in /root/.docker/config.json
		Login Succeeded
		
3.push����˽��registry</br>

		docker tag 91e54dfb1179 api.192.168.172.150.xip.io/ubuntu:trusty
		docker push api.192.168.172.150.xip.io/ubuntu:trusty
		
		The push refers to a repository [api.192.168.172.150.xip.io/ubuntu] (len: 1)
		91e54dfb1179: Image already exists 
		d74508fb6632: Image successfully pushed 
		c22013c84729: Image successfully pushed 
		d3a1f33e8a5a: Image successfully pushed 
		Digest: sha256:a731c12a4d21af384c4659666f177cd1e871646b95b9440d709ec4ee176145b2
		
4.�鿴�Ƿ��ϴ�</br>

		curl -i -k -v https://admin:admin@api.192.168.172.150.xip.io/v2/_catalog
		
		{"repositories":["ubuntu"]}
		
		cd /home/jojo/registry/docker/registry/v2 && ls
		blobs  repositories
		
����Ͳ������濴�ˣ�����registry����Լ�������������ݽṹ��