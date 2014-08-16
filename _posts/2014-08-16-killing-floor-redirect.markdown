---
layout: post
title:  "How to host a Redirect Server for Killing Floor using NGINX"
date:   2014-08-16 10:00:00
categories: killing floor
---

If you have plenty of uplink bandwidth available it is possible to host a redirect server for [Killing Floor][].
First check if you have bandwidth available, to run a six player server you need approximately 500kbps (kilobits per second) of upload speed, for a ten player server a good practice is to reserve about 1Mbit of upload.
The download speed requirements is low, for a six player server you can estimate 200kbps.

For instance, if you have a ADSL connection the upload speed is limited to 800kbps - in this case you can skip this tutorial and find another home for you redirect server.
But if you have a VDSL connection or fiber cable then probably you have some bandwidth available (e.g. VDSL uploads can reach 16Mbps), there are alot of broadband speed test sites around the Internet where you can check your upload speed.

I choose [Nginx][] because it is lighter and have better performance than [Apache][] ([Nginx][] passes [Apache][] as Web server of choice among top sites - it is used by high-traffic websites such as Facebook, Zynga, Dropbox and Netflix).

But the Internet is like the Wild West today, so we need some careful while hosting the [Killing Floor][] dedicated server together with the Redirect server. At a minimum, the Redirect server must be configured in order to:

- limit the overall number of players downloading files;
- limit the bandwidth used per player, and the number of file requests per second;
- deny parallel downloads ([Killing Floor] only donwloads a single file at a time);
- deny browsers from downloading files;

Without those kind of protections a mal-intentioned individual can easily do a denial of service (players will experiment lag, etc). We need to sacrifice the Redirect server limits in favor of current players who already downloaded the server content.

Also it is possible to redirect the download of certain files to other Redirect servers (a public Redirect server of course).
You can implement a kind of fallback mechanism where if the file is not available the [Nginx][] will automatically proxy the request to a public Redirect server.

With the help of a Mutator running in the dedicated server is also possible to allow only players playing in your server to download files.
You can use this to prevent other servers from using your Redirect server.

[Nginx]: http://nginx.org/
[Apache]: http://httpd.apache.org/
[Killing Floor]: http://www.killingfloorthegame.com/

{% highlight nginx %}
worker_processes  2;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        off;

    server_names_hash_bucket_size 128;

    # Size Limits & Buffer Overflows
    client_body_buffer_size  1K;
    client_header_buffer_size 1k;
    client_max_body_size 1k;
    large_client_header_buffers 2 1k;

    # Timeouts
    client_body_timeout   10;
    client_header_timeout 10;
    keepalive_timeout     30;
    send_timeout          10;
    keepalive_requests    10;

    geo $limited {
        default         1;
        127.0.0.1       0;
        10.0.0.0/8      0;
        172.16.0.0/12   0;
        192.168.0.0/16  0;
    }

    map $limited $limit {
        0 "";
        1 $binary_remote_addr;
    }

    limit_conn_zone $limit zone=conn_limit_per_ip:10m;
    limit_req_zone $limit zone=req_limit_per_ip:10m rate=3r/s;
    limit_traffic_rate_zone rate $limit 10m;
    lua_shared_dict clients 10m;

    server {

        limit_conn conn_limit_per_ip 1;
        limit_req zone=req_limit_per_ip burst=6;

        server_tokens off;

        # remove the "Server: nginx" header
        more_set_headers 'Server:';

        listen       80;
        server_name  localhost;

        location @fallback_403 {
            default_type 'text/html';
            echo "<html>";
            echo "<head><title>403 Forbidden</title></head>";
            echo "<body bgcolor=\"white\">";
            echo "<center><h1>403 Forbidden</h1></center>";
            echo "</body>";
            echo "</html>";
        }
        error_page 401 403 @fallback_403;

        location @fallback_404 {
            default_type 'text/html';
            echo "<html>";
            echo "<head><title>404 Not Found</title></head>";
            echo "<body bgcolor=\"white\">";
            echo "<center><h1>404 Not Found</h1></center>";
            echo "</body>";
            echo "</html>";
        }
        error_page 404 @fallback_404;

        location @fallback_5xx {
            default_type 'text/html';
            echo "<html>";
            echo "<head><title>500 Internal Server Error</title></head>";
            echo "<body bgcolor=\"white\">";
            echo "<center><h1>500 Internal Server Error</h1></center>";
            echo "</body>";
            echo "</html>";
        }
        error_page 500 501 502 503 504 @fallback_5xx;

        location / {
            include    ./conf/mysite.rules; # see also http block naxsi include line
            root   html;
            index  index.html index.htm;
            deny all;
        }

        location /set {
            default_type 'text/plain';

            # allow only localhost access
            allow 127.0.0.1;
            deny all;

            # add IP in the whitelist (e.g. http://127.0.0.1/set?ip=192.168.0.1)
            set_by_lua $res '
                local clients = ngx.shared.clients
                local args = ngx.req.get_uri_args()
                clients:set("ip_"..args.ip, 1, 7200)
                return args.ip
            ';

            # allow only UNREAL user agent
            if ($http_user_agent !~* Unreal) {
                return 403;
            }

            # return results
            add_header 'Content-Location' 'KF: $res';
            return 200;
        }

        location /redirect {
            include    ./conf/mysite.rules; # see also http block naxsi include line
            limit_traffic_rate rate 256k;

            #autoindex on;

            # allow only whitelisted IP's
            access_by_lua '
                local clients = ngx.shared.clients
                local allow = clients:get("ip_"..ngx.var.remote_addr) or tonumber(ngx.var.limited) == 0
                if not allow then
                    ngx.sleep(0.3)
                    allow = clients:get("ip_"..ngx.var.remote_addr)
                    if not allow then
                        ngx.exit(ngx.HTTP_FORBIDDEN)
                    end
                end
            ';

            # allow only UNREAL user agent
            if ($http_user_agent !~* Unreal) {
                return 403;
            }
        }
    }
}
{% endhighlight %}

