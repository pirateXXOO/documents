# varnish配置文件



```shell
# new 4.0 format.
vcl 4.0;

import directors;

# Default backend definition. Set this to point to your content server.

probe webprobe {  # 健康状态检查
    .url = "/index.html";
    .interval = 2s;
    .timeout = 2s;
    .window = 5;
    .threshold = 3;
}

backend default {
    .host = "172.16.71.4";
    .port = "80";
    .probe = webprobe;
}


backend static {
    .host = "172.16.71.5";
    .port = "80";
    .probe = webprobe;
}


acl purgers {    
    "127.0.0.0"/8;
#   "172.16.0.0"/16;
}

sub vcl_init {
    new websrvs = directors.round_robin();
    websrvs.add_backend(default);
    websrvs.add_backend(static);
}

sub vcl_recv {
    set req.backend_hint = websrvs.backend();
    #if (req.url ~ "(?i)^/login") { 
    #   return(pass);
    #}
    # 动静分离
    #if (req.url ~ "(?i)\.php$"){
    #   set req.backend_hint = default;
    #} else {
    #   set req.backend_hint = static;
    #}
    if (req.method == "PURGE") {
        if (!client.ip ~ purgers){
            return(synth(05,"Purging is not allowed to "+client.ip));
        }
        return(purge);
    }
    # Happens before we check if we have this in cache already.
    #
    # Typically you clean up the request here, removing cookies you don't need,
    # rewriting the request, etc.
}

sub vcl_backend_response {
    # Happens after we have read the response headers from the backend.
    #
    # Here you clean the response headers, removing silly Set-Cookie headers
    # and other mistakes your backend does.
}

sub vcl_deliver {
    # Happens when we have all the pieces we need, and are about to send the
    # response to the client.
    #
    # You can do accounting or modifying the final object here.
    if (obj.hits>0) {
        set resp.http.X-Cache = "Hit via " + server.ip;
    } else {
        set resp.http.X-Cache = "Miss via " + server.ip;
    }
}
sub vcl_purge{
    return(synth(200,"Purged"));
}

```

