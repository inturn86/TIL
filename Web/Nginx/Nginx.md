

### Nginx 설치

```
$ sudo apt update
$ sudo apt install nginx
```

Nginx 시작하면서 에러 발생

```
Reading package lists... Done
Building dependency tree
Reading state information... Done
nginx is already the newest version (1.14.0-0ubuntu1.11).
0 upgraded, 0 newly installed, 0 to remove and 239 not upgraded.
2 not fully installed or removed.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] Y
Setting up nginx-core (1.14.0-0ubuntu1.11) ...
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
invoke-rc.d: initscript nginx, action "start" failed.
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Fri 2024-05-17 10:49:42 KST; 6ms ago
     Docs: man:nginx(8)
  Process: 5339 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=1/FAILURE)

May 17 10:49:42 nginx-server systemd[1]: Starting A high performance web server and a reverse proxy server...
May 17 10:49:42 nginx-server nginx[5339]: nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by                                                                                                                        protocol)
May 17 10:49:42 nginx-server nginx[5339]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 17 10:49:42 nginx-server systemd[1]: nginx.service: Control process exited, code=exited status=1
May 17 10:49:42 nginx-server systemd[1]: nginx.service: Failed with result 'exit-code'.
May 17 10:49:42 nginx-server systemd[1]: Failed to start A high performance web server and a reverse proxy server.
dpkg: error processing package nginx-core (--configure):
 installed nginx-core package post-installation script subprocess returned error exit status 1
dpkg: dependency problems prevent configuration of nginx:
 nginx depends on nginx-core (<< 1.14.0-0ubuntu1.11.1~) | nginx-full (<< 1.14.0-0ubuntu1.11.1~) | nginx-light (<< 1.14                                                                                                                       .0-0ubuntu1.11.1~) | nginx-extras (<< 1.14.0-0ubuntu1.11.1~); however:
  Package nginx-core is not configured yet.
  Package nginx-full is not installed.
  Package nginx-light is not installed.
  Package nginx-extras is not installed.
 nginx depends on nginx-core (>= 1.14.0-0ubuntu1.11) | nginx-full (>= 1.14.0-0ubuntu1.11) | nginx-light (>= 1.14.0-0ub                                                                                                                       untu1.11) | nginx-extras (>= 1.14.0-0ubuntu1.11); however:
  Package nginx-core is not configured yet.
  Package nginx-full is not installed.
  Package nginx-light is not installed.
  Package nginx-extras is not installed.

dpkg: error processing package nginx (--configure):
 dependency problems - leaving unconfigured
No apport report written because the error message indicates its a followup error from a previous failure.
                                                                                                          Errors were                                                                                                                        encountered while processing:
 nginx-core
 nginx
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

위와 같이 에러가 발생한 경우 nginx log를 확인하여 어떤 상황에서 에러가 발생했는지 확인한다.

```
$ vi /var/log/nginx/error.log
```

발생한 에러는 다음과 같다.
```
2024/05/17 10:55:11 [emerg] 6623#6623: socket() [::]:80 failed (97: Address family not supported by protocol)
```

해당 에러는 ipv6 가 비활성화 되어 있다면 발생한다.

이런 경우 ipv6 리스닝 부분을 삭제하거나 주석 처리하면 된다.

```
$ vi /etc/nginx/sites-enabled/default
```

```
server {
        listen 80 default_server;
        #listen [::]:80 default_server;
}
```

설치 완료 후 서비스 실행한다.
```
$ service nginx start
$ service nginx status
```

정상적으로 실행되었다면 아래와 같이 active (running) 이라는 문구를 확인할 수 있다.
```
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2024-05-17 10:57:22 KST; 29min ago
     Docs: man:nginx(8)
  Process: 7049 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 7043 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 7054 (nginx)
    Tasks: 5 (limit: 19137)
   CGroup: /system.slice/nginx.service
           ├─7054 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           ├─7057 nginx: worker process
           ├─7059 nginx: worker process
           ├─7060 nginx: worker process
           └─7061 nginx: worker process
```

그리고 nginx 가 설치된 공인 IP의 80 포트로 접근하면 아래와 같이 외부에서 접근 가능한지 여부를 확인할 수 있다.

`ex) http://{공인IP}:80`


![[Pasted image 20240517113128.png]]

### Nginx 로드밸런싱 설정

Nginx는 Http 요청이 올 경우 site-enabled 폴더 안의 파일들을 기준으로 로드밸런싱을 처리한다.
해당 경로로 이동하여 기존 설정인 default 파일을 삭제하고 필요로하는 로드밸런싱 설정을 추가한다.

```
$ cd /etc/nginx/sites-enabled
$ rm -rf default
$ vi {설정파일명}
```

```
upstream backend{
		# 원하는 알고리즘 설정과 로드밸런싱 서버 정보
        server 1.1.1.1:8080;
        server 2.2.2.2:8080;
}

server{
		# port 설정
        listen 80;
        location / {
		        # 상단 upstream의 명칭과 일치
                proxy_pass http://backend;
        }
}
```

설정 추가 후 아래 커맨드로 설정을 갱신한다.
```
$ nginx -s reload
```

이렇게 작업 후 Nginx 공인 IP로 요청을 전송하면 각 서버로 로드밸런싱 된다.