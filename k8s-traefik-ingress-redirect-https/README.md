# K8s Traefik Ingress Redirect to HTTPS
In K8s Nginx Ingress Controller, to force redirect requests to HTTPS, you simple add an annotation to Ingress object:
```yaml
kind: Ingress
...
spec:
  metadata:
    annotations:
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
...
```
but in Traefik Ingress, there's no straightforward of doing this unlike Nginx.

To redirect requests to HTTPS permanently, we need to create a Traefik Middleware first, then add annotation to the Traefik Ingress object:

**File:** `redirect-https.yaml`
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

**File:** `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: letsencrypt-issuer
    # value template: {namespace}-{middleware-name}@kubernetescrd
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
  name: monger
  namespace: default
spec:
  ingressClassName: traefik
  rules:
  - host: demo.sareno.dev
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: myservice
            port:
              number: 8000
  tls:
  - secretName: demo.sareno.dev
    hosts:
    - demo.sareno.dev
```

## Links
- https://doc.traefik.io/traefik/middlewares/overview/
- https://doc.traefik.io/traefik/middlewares/http/redirectscheme/
- https://itobey.dev/redirect-http-to-https-traefik-middleware-kubernetes/
