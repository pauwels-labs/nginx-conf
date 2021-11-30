# nginx-conf
The nginx config for running a full Redact local environment
(client/storage/feed-ui/feed-api)

## nginx.conf
The `stream` block inside nginx.conf sets up the initial split of traffic from
one listening port into a TLS port and a non-TLS port.

```
server {
	listen 1010;
	listen [::]:1010;
	proxy_pass $upstream;
	ssl_preread on;
}
```

The `1010` is your external listening port, TLS and non-TLS requests to both the
Redact Feed UI and API should go to this port. All other ports used as part of
this config should be considered internal and not used by external services
(e.g. the client).

```
 upstream http {
	 server localhost:1000;
 }

 upstream https {
	 server localhost:1001;
 }
```

These blocks set up a port for streaming HTTP requests (1000) and a port for
streaming HTTPS (1001) requests.

```
map $ssl_preread_protocol $upstream {
	default https;
	"" http;
}
```

Finally this block tells nginx how to decide which upstream to send a request to
after receiving it on `1010`. It does this by simply looking at the TLS protocol
attached to the request: all protocol-less (non-TLS) requests go to the http
upstream, all others to the https upstream.

## conf.d/redact-feed.conf
This file directs HTTP and HTTPS requests to the appropriate backend based on
the request's path.

The two server blocks listen on 1001 (TLS) and 1000 (non-TLS). This correlates
to the upstream blocks in nginx.conf.

```
ssl_certificate /home/ajp/projects/redact/redact-feed-api/tls/cert.pem;
ssl_certificate_key /home/ajp/projects/redact/redact-feed-api/tls/key.pem;
ssl_protocols TLSv1.3;
ssl_verify_client optional_no_ca;
```

These four lines provide the parameters for terminating TLS with nginx. The
server certificate and key is provided in the first two lines. We only allow TLS
v1.3 so this is added to further restrict incoming connections. Finally, since
our TLS operations require a client certificate, the `optional_no_ca` parameter
requests that certificate but does not verify it as we allow any mutual TLS
request.

The `location` blocks direct traffic to either the UI or API dependent on
path. Currently, all TLS requests are either for relaying or session creation
and go to the API. **The API IP address can change regularly due to Flask. You
may need to update this IP in the nginx configuration in the future.** All
non-TLS requests are forwarded to the UI.
