# iredmail-kubernetes
Running the mail server with UI on minimal resources

An unloaded server can run on about 10m CPU, 512Mi Memory.
After installation will be available:
roundcube (web ui): EMAIL_SERVER_HOSTNAME (ex: email.example.com)
iredmail (admin ui): EMAIL_SERVER_HOSTNAME/iredadmin (ex: email.example.com/iredadmin)
smtp/pop3/imap server: EMAIL_SERVER_HOSTNAME (ex: email.example.com)
first user: postmaster@FIRST_MAIL_DOMAIN (ex: postmaster@example.com)
first user password: FIRST_MAIL_DOMAIN_ADMIN_PASSWORD (ex: password)

You will also need to set up 
MX record (to receive mail) for the domain: FIRST_MAIL_DOMAIN (ex: example.com)
A record (to get letsencrypt certificates) for the domain: EMAIL_SERVER_HOSTNAME (ex: email.example.com)
SPF, DKIM and DMARK records for better delivery service

This example binds the server to a node and is not a highly available deployment option. (due to the use of local storage)

```bash
# Env
export NODE_HOSTNAME="worker-1.example.com"
export EMAIL_SERVER_HOSTNAME="email.example.com"
export FIRST_MAIL_DOMAIN="example.com"
export FIRST_MAIL_DOMAIN_ADMIN_PASSWORD="password"

mkdir /home/iredmail
cd /home/iredmail
kubectl create namespace iredmail
```

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

tee /home/iredmail/ingress-configmap.yaml<<EOL
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  25: "iredmail-server/iredmail:25"
  465: "iredmail-server/iredmail:465"
  587: "iredmail-server/iredmail:587"
  143: "iredmail-server/iredmail:143"
  993: "iredmail-server/iredmail:993"
  110: "iredmail-server/iredmail:110"
  995: "iredmail-server/iredmail:995"
EOL

kubectl apply -f ingress-configmap.yaml
```

Create ingress for email server (In order for the certificates to be successfully created, you must already have a DNS record of type A configured for the address EMAIL_SERVER_HOSTNAME)
```bash
tee /home/iredmail/ingress.yaml<<EOL
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: iredmail-server
  name: iredmail-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  tls:
  - hosts:
    - $EMAIL_SERVER_HOSTNAME
    secretName: iredmail-certs
  rules:
  - host: $EMAIL_SERVER_HOSTNAME
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: iredmail
            port:
              number: 443
EOL

kubectl apply -f ingress.yaml
```

Also the iredmail server will require additional certificates, so you need to add them.
```bash
kubectl patch deployment cert-manager -n cert-manager --type "json" -p '[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value": "--feature-gates=AdditionalCertificateOutputFormats=true"},
]'

kubectl patch deployment cert-manager-webhook -n cert-manager --type "json" -p '[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value": "--feature-gates=AdditionalCertificateOutputFormats=true"},
]'

# (der is not needed)
kubectl patch certificate iredmail-certs -n iredmail-server --type "json" -p '[
  {"op":"add","path":"/spec/additionalOutputFormats","value": [{"type": "CombinedPEM"},{"type": "DER"}]},
]'
```

Create persistent storage (You can experiment with other storage classes, but network storage will be quite slow.)
```bash
mkdir /mnt/iredmail
chmod 0777 -R /mnt/iredmail
sudo chown -R nobody:nogroup /mnt/iredmail

tee /home/iredmail/storage.yaml<<EOL
apiVersion: v1
kind: PersistentVolume
metadata:
  name: iredmail
  namespace: iredmail-server
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/iredmail
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - $NODE_HOSTNAME
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iredmail
  namespace: iredmail-server
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
EOL

kubectl apply -f storage.yaml
```

Create an iredmail deployment and service ()
```bash
tee /home/iredmail/iredmail.yaml<<EOL
apiVersion: v1
kind: Service
metadata:
  namespace: iredmail-server
  name: iredmail
spec:
  selector:
    app: iredmail
  ports:
    - name: "80"
      port: 80
      targetPort: 80
    - name: "443"
      port: 443
      targetPort: 443
    - name: "110"
      port: 110
      targetPort: 110
    - name: "995"
      port: 995
      targetPort: 995
    - name: "143"
      port: 143
      targetPort: 143
    - name: "993"
      port: 993
      targetPort: 993
    - name: "25"
      port: 25
      targetPort: 25
    - name: "465"
      port: 465
      targetPort: 465
    - name: "587"
      port: 587
      targetPort: 587
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: iredmail-server
  name: iredmail
  labels:
    app: iredmail
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: iredmail
  template:
    metadata:
      labels:
        app: iredmail
    spec:
      containers:
        - name: iredmail
          image: iredmail/mariadb:stable
          env:
            - name: FIRST_MAIL_DOMAIN
              value: $FIRST_MAIL_DOMAIN
            - name: FIRST_MAIL_DOMAIN_ADMIN_PASSWORD
              value: $FIRST_MAIL_DOMAIN_ADMIN_PASSWORD
            - name: HOSTNAME
              value: $$EMAIL_SERVER_HOSTNAME
            - name: MLMMJADMIN_API_TOKEN
              value: $(openssl rand -base64 32)
            - name: ROUNDCUBE_DES_KEY
              value: $(openssl rand -base64 24)
          ports:
            - containerPort: 80
            - containerPort: 443
            - containerPort: 110
            - containerPort: 995
            - containerPort: 143
            - containerPort: 993
            - containerPort: 25
            - containerPort: 465
            - containerPort: 587
          resources:
            limits:
              cpu: "1"
              # if you plan to use clamav you need to increase the resources to 2Gi
              memory: "1000Mi"
            requests:
              cpu: "0.1"
              memory: "500Mi"
          volumeMounts:
            - mountPath: /var/vmail/backup/mysql
              subPath: backup_mysql
              name: iredmail-data
            - mountPath: /var/vmail/vmail1
              subPath: vmail1
              name: iredmail-data
            - mountPath: /var/vmail/mlmmj
              subPath: mlmmj
              name: iredmail-data
            - mountPath: /var/vmail/mlmmj-archive
              subPath: mlmmj-archive
              name: iredmail-data
            - mountPath: /var/vmail/imapsieve_copy
              subPath: imapsieve_copy
              name: iredmail-data
            - mountPath: /opt/iredmail/custom
              subPath: custom
              name: iredmail-data
            - mountPath: /opt/iredmail/ssl
              subPath: ssl
              name: iredmail-data
            - mountPath: /var/lib/mysql
              subPath: mysql
              name: iredmail-data
            - mountPath: /var/lib/clamav
              subPath: clamav
              name: iredmail-data
            - mountPath: /var/lib/spamassassin
              subPath: spamassassin
              name: iredmail-data
            - mountPath: /var/spool/postfix
              subPath: postfix
              name: iredmail-data
            - mountPath: /opt/iredmail/ssl/cert.pem
              subPath: tls.crt
              name: iredmail-certs
            - mountPath: /opt/iredmail/ssl/key.pem
              subPath: tls.key
              name: iredmail-certs
            - mountPath: /opt/iredmail/ssl/combined.pem
              subPath: tls-combined.pem
              name: iredmail-certs
      hostname: localhost
      restartPolicy: Always
      volumes:
        - name: iredmail-data
          persistentVolumeClaim:
            claimName: iredmail
        - name: iredmail-certs
          secret:
            secretName: iredmail-certs
EOL

kubectl apply -f iredmail.yaml
```

Turn off clamav:
```bash
# create new service file
mkdir /mnt/iredmail/configs
mkdir /mnt/iredmail/configs/supervisor
tee /mnt/iredmail/configs/supervisor/clamav.conf<<EOF
[program:clamav]
command=/usr/sbin/clamd -c /etc/clamav/clamd.conf --foreground
priority=999
startsecs=0
autostart=false
autorestart=false
stdout_syslog=true
stderr_syslog=true
EOF

sudo tee /mnt/iredmail/custom/amavisd/amavisd.conf<<EOF
# disable clamav

@bypass_virus_checks_maps = 1;
@bypass_spam_checks_maps = 1;
EOF

# patch deployment
kubectl patch deployment iredmail -n iredmail-server --type "json" -p '[
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":
    {"mountPath":"/etc/supervisor/conf.d/clamav.conf","subPath":"configs/supervisor/clamav.conf","name":"local-data"}
  }
]'

# run in pod (kubectl exec POD_NAME -n iredmail-server -- COMMAND)
chown root:amavis /opt/iredmail/custom/amavisd/amavisd.conf
chmod 0655 /opt/iredmail/custom/amavisd/amavisd.conf
```
