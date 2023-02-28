# iredmail-kubernetes
Running the mail server with UI on minimal resources

You will need to forward the ports via ingress to the iredmail-server service. To do this, you need to force ingress to listen to traffic on these ports.
```bash
# patch nginx
kubectl patch svc ingress-nginx-controller -n ingress-nginx --type "json" -p '[
  {"op":"add","path":"/spec/ports/-","value":{"name": "smtp", "port": 25, "targetPort": 25}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "smtp-ssl", "port": 465, "targetPort": 465}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "smtp-tls", "port": 587, "targetPort": 587}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "imap-tls", "port": 143, "targetPort": 143}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "imap-ssl", "port": 993, "targetPort": 993}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "pop3-tls", "port": 110, "targetPort": 110}},
  {"op":"add","path":"/spec/ports/-","value":{"name": "pop3-ssl", "port": 995, "targetPort": 995}}
]'

kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type "json" -p '[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value": "--tcp-services-configmap=$(POD_NAMESPACE)/tcp-services"}
]'

kubectl apply -f configmap.yaml
```
