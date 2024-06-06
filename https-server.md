 # Support Https and Reverse Proxy for Http RPC URL

  - This [doc](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04) guides you to support https
  - This [doc](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-reverse-proxy-on-ubuntu-22-04) guides you to support reserve proxy

  Also there is a valid reserve proxy settings you can refer to:
  ```
  	location / {
		#try_files $uri $uri/ =404;
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;

        proxy_pass          http://142.132.154.16:8545;
        proxy_read_timeout  90;

        proxy_redirect      off;        
	}
  ```