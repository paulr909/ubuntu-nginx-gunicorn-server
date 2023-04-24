# NGINX Config for HTTP or HTTPS

## HTTP Only

    upstream app_server {
    server unix:/home/app-name/app-name/run/gunicorn.sock fail_timeout=0;
    }

    server {
            listen 80;
            server_name website-name.com;
            keepalive_timeout 5;
            client_max_body_size 4G;
            access_log /home/app-name/app-name/logs/nginx-access.log;
            error_log /home/app-name/app-name/logs/nginx-error.log;

            location /static/ {
                alias /home/app-name/app-name/staticfiles/;
            }

            # checks for static file, if not found proxy to app
            location / {
                try_files $uri @proxy_to_app;
            }

            location /favicon.ico {
                alias /home/app-name/app-name/favicon.ico;
            }

            error_page 404 /custom_404.html;
            location = /custom_404.html {
                root /usr/share/nginx/html;
                internal;
            }

            error_page 500 502 503 504 /custom_50x.html;
            location = /custom_50x.html {
                root /usr/share/nginx/html;
                internal;
            }

            location @proxy_to_app {
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_redirect off;
                proxy_pass http://app_server;
                proxy_intercept_errors on;
            }
    }

## HTTPS 

    upstream app_server {  
	server unix:/home/app-name/app-name/run/gunicorn.sock fail_timeout=0;
	}
	
	server { 
	        listen 443 ssl;
            listen [::]:443 ssl;
            include snippets/self-signed.conf;
            include snippets/ssl-params.conf;

            server_name website-name.com;
            keepalive_timeout 5;
            client_max_body_size 4G;
            access_log /home/app-name/app-name/logs/nginx-access.log;
            error_log /home/app-name/app-name/logs/nginx-error.log;

            location /static/ {
                alias /home/app-name/app-name/staticfiles/;
            }

            # checks for static file, if not found proxy to app
            location / {
                try_files $uri @proxy_to_app;
            }

            location = /favicon.ico {
                alias /home/app-name/app-name/favicon.ico;

            }

            error_page 404 /custom_404.html;
            location = /custom_404.html {
                root /usr/share/nginx/html;
                internal;
            }

            error_page 500 502 503 504 /custom_50x.html;
            location = /custom_50x.html {
                root /usr/share/nginx/html;
                internal;
            }

            location @proxy_to_app {
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_redirect off;
                proxy_pass http://app_server;
                proxy_intercept_errors on;
            }

        }

    server {
            listen 80;
            listen [::]:80;
            server_name website-name.com;

            return 301 https://$server_name$request_uri;
    }
			

## Custom Error Pages

We will put our custom error pages in the /usr/share/nginx/html/ directory where Ubuntu's Nginx sets its default document root.

[https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-to-use-custom-error-pages-on-ubuntu-22-04](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-to-use-custom-error-pages-on-ubuntu-22-04)