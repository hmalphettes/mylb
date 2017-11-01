My Load Balancer
================

Test NGINX as a Load balancer and TLS termination.
Use docker-compose to start the containers involved.

Explanation:
============
The docker-nginx container runs NGINX as a load balancer and TLS termination.
The container listens to the docker daemon events stream.
When a container is started with the environment variable `VIRTUAL_HOST=foo.com` it generates an NGINX configuration that will proxy the requests to `foo.com` to the corresponding container.
When there is more than one container, it will load balance them. Round robin.

This is the key snippet of the NGINX configuration template:
https://github.com/jwilder/nginx-proxy/blob/master/nginx.tmpl#L119
```
# {{ $host }}
upstream {{ $upstream_name }} {

{{ range $container := $containers }}
	{{ $addrLen := len $container.Addresses }}
(...)
{{ end }}
}
```

Side note: Add a DynDNS container based on the same idea.
==========================================================
I have developed a DynDNS container using the same idea and the same environment variables: https://github.com/hmalphettes/docker-route53-dyndns

When `nginx-proxy` and `docker-route-53-dyndns` are running together, the environment variable `VIRTUAL_HOST` will make the container register the current public IP of the host on the AWS Route53 DNS and will expose the endpoint through NGINX.

I have actually used this for some setups: https://hub.docker.com/r/hmalphettes/nginx-proxy/builds/

When running a microservice on Kubernetes that needs to be exposed as a public API, one could consider using ambassador or contour.
Reference: https://www.getambassador.io/about/why-ambassador

Test with Vagrant on Macos:
===========================
```
git clone --depth 1 https://github.com/hmalphettes/mylb
cd mylb
vagrant up
```

Test from ssh:
```
vagrant ssh
# note the id of the containers:
docker ps | grep share_whoami | awk '{print $1}'
# the id of the worker hit by the request is returned; round-robin
curl -k https://mylb.sg
curl -k https://mylb.sg
curl -k https://mylb.sg
curl -k https://mylb.sg
curl -k https://mylb.sg
```
Round robin is not the best algo for a load balancer; the best is usually to randomly select one of the healthy workers.

Test from your browser:
- `sudo bash -c 'echo "127.0.0.1 mylb.sg" >> /etc/hosts'`
- `open https://mylb.sg:8443`; ignore the TLS warnings: this is a self-signed certificate.
- refresh the page a few times. It displays the container-ID of the 3 workers one after the other.



