Realizzare una connessione https con i pod Nginx

https://8gwifi.org/docs/kube-nginx.jsp

~~~~~~~
Guida:

1. Creare un self signed certificate per example.com
openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout tls.key -out tls.crt -subj "/CN=example.com" -days 365

2. verranno salvati due file tls.key tls.crt

3. creare un secret a partire da questo certificato:
kubectl create secret generic nginx-certs-keys --from-file=./tls.crt --from-file=./tls.key

4.Salvare un file dal nome default.conf:

server {
      listen 80 default_server;
      listen [::]:80 default_server ipv6only=on;
      listen 443 ssl;

      root /usr/share/nginx/html;
      index index.html;

      server_name localhost;
      ssl_certificate /etc/nginx/ssl/tls.crt;
      ssl_certificate_key /etc/nginx/ssl/tls.key; 
      ssl_session_timeout 1d;
      ssl_session_cache shared:SSL:50m;
      ssl_session_tickets off;
      # modern configuration. tweak to your needs.
      ssl_protocols TLSv1.2;
      ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
      ssl_prefer_server_ciphers on; 
      # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
      add_header Strict-Transport-Security max-age=15768000;
      # OCSP Stapling ---
      # fetch OCSP records from URL in ssl_certificate and cache them
      ssl_stapling on;
      ssl_stapling_verify on;
      location / {
              try_files $uri $uri/ =404;
      }
}

5. creare un configmap a partire da quel file:
kubectl create configmap nginxconfigmap --from-file=default.conf

6. creare il file yaml nginx-app.yaml per deployare nginx con questo configmap e secret, esposto tramite nodeport:

apiVersion: v1
kind: Service
metadata:
  name: nginxsvc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    app: nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: secret-volume
        secret:
           secretName: nginx-certs-keys 
      - name: configmap-volume
        configMap:
          name: nginxconfigmap 
      containers:
      - name: nginxhttps
        image: ymqytw/nginxhttps:1.5
        command: ["/home/auto-reload-nginx.sh"]
        ports:
        - containerPort: 443
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /index.html
            port: 80
          initialDelaySeconds: 30
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume


7. deployare su minikube/k8s lo yaml:
kubectl apply -f nginx-app.yaml

8. vedere quali sono gli indirizzi http e https esposti:
minikube service list

|-------------|------------|--------------|----------------------------|
|  NAMESPACE  |    NAME    | TARGET PORT  |            URL             |
|-------------|------------|--------------|----------------------------|
| default     | kubernetes | No node port |
| default     | nginxsvc   | http/80      | http://192.168.64.19:31366 |
|             |            | https/443    | http://192.168.64.19:30459 |
| kube-system | kube-dns   | No node port |
|-------------|------------|--------------|----------------------------|

9. minikube ip
192.168.64.19


10. chiamare il servizio:
curl  -k  https://<node-ip>:<nodeport>
curl -kI https://192.168.64.19:30459

11. esportare minikube-ip su /etc/hosts:
echo "$(minikube ip) example.com" | sudo tee -a /etc/hosts
cat /etc/hosts | tail -1

12. chiamare il servizio con il nome del dominio:
curl -kI https://example.com:30459

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Praticamente questo dimostra come un ingress di k8s non sia nient'altro che una implementazione nativa con le API di k8s di un ingress controller (un nginx ad esempio) esposto tramite un service di tipo loadbalancer o nodeport. Qui abbiamo fatto lo stesso: creato un ingress ma senza usare le API per l'ingress.
