<!-- .slide: data-timing="1" --> 
# Backup slides for questions  <!-- .element class="hidden" -->

<!-- Note -->
This slide is intentionally blank.


<!-- .slide: data-timing="1" --> 
## Why S3 for Open edX,  
## and not Swift?

<!-- Note -->
The Open edX platform does support both S3 and OpenStack Swift for object storage purposes (it uses [django-storages](https://django-storages.readthedocs.io/)), but its S3 support is much more mature and complete.

So, since the Ceph Object Gateway gives us the option of running *both* APIs, we decided we'd go for the S3 API, because that just means a lot less work that we needed to do on the Open edX side of things.


<!-- .slide: data-timing="1" --> 
## Why not use HTTPS termination in Octavia?

<!-- Note -->
In principle we *could* terminate HTTPS in Octavia, rather than in Caddy like we do today.
OCCM, that is the OpenStack Cloud Controller Manager, [supports this through a service annotation](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/expose-applications-using-loadbalancer-type-service.md).

The way that works, it would look like this:


<!-- .slide: data-timing="1" --> 
### YAML example for service annotation <!-- .element class="hidden" -->

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: loadbalanced-service
  annotations:
    'loadbalancer.openstack.org/default-tls-container-ref': >
       'https://example.com/path/to/barbican-secret'
    'loadbalancer.openstack.org/x-forwarded-for': true
spec:
  type: LoadBalancer
  # [...]
```
  
<!-- Note -->
We could use this, but it wouldn't work.
That's because this way we can set the `X-Forwarded-For` HTTP header, but *not* `X-Forwarded-Port` which we also need, to make Open edX work.

That in turn is because although the service annotation gives the option to set `X-Forwarded-For` (as you can see in the example), and *Octavia* also lets us set `X-Forwarded-Port`, we cannot combine the two for a `LoadBalancer` service with OCCM.

We could work around this with a lot of manual hacking, but decided not to.
That's why we terminate HTTPS with Caddy, and just use TCP load balancing through Octavia.
