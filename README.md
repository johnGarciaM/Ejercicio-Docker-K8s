# **_CONTENEDORES Y K8S EN LOCAL_**

## **Ejemplo práctico**

### el siguiente ejemplo es una un programa que levanta un servidor de nginx en ubuntu y publica un simple html.

#### **pasos**:

1. Crear html de prueba.

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Imagen Docker NGINX</title>
  </head>
  <body>
    <h1>Test de Imágen</h1>
  </body>
</html>
```

2. Crear DockerFile.

```docker
# definimos el SO
FROM ubuntu
# instalamos nginx para ubuntu
RUN apt update && apt upgrade -y \
    && apt install nginx -y
# el html local a la carpeta de nginx
COPY ./index.html /var/www/mi-web/html/
# configuración de nginx
COPY ./mi-web /etc/nginx/sites-enabled/
# encender el servicio de nginx
CMD service nginx start && tail -F /var/log/nginx/error.log
# exponer el puerto nginx 81
EXPOSE 81
```

3. Crear configuración de nginx.

```
server {
    # Puerto en que se desea publicar nginx
    listen 81;
    listen [::]:81;
    # Posibles dominios
    server_name mi-web.com www.mi-web.com;
    # Se le indica la ubicación de la app.
    root /var/www/mi-web/html;
    # Nombre del archivo html
    index index.html;

    location / {
            try_files $uri $uri/ =404;
    }
}
```

4. crear el archivo para el deployment.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ubuntu-nginx-01-deployment
  namespace: default
  labels:
    app: ubuntu-nginx-01
spec:
  selector:
    matchLabels:
      app: ubuntu-nginx-01
  replicas: 1
  template:
    metadata:
      labels:
        app: ubuntu-nginx-01
    spec:
      containers:
        - name: ubuntu-nginx-01-container
          image: localhost:5000/ubuntu-nginx-01:latest
          ports:
            - containerPort: 81
          env:
            - name: NGINX_PORT
              value: "81"
          resources:
            requests:
              cpu: 2m
              memory: 30Mi
            limits:
              cpu: 2m
              memory: 30Mi
```

- Crear el servicio

```yml
apiVersion: v1
kind: Service
metadata:
  name: ubuntu-nginx-01-service
spec:
  type: LoadBalancer
  selector:
    app: ubuntu-nginx-01
  ports:
    - name: http
      port: 80
      targetPort: 81
```

## Instrucciones:

- Construir la imágen.

```bash
# construir la imagen
docker build --no-cache -t ubuntu-nginx-01 -f DockerFile .
```

- Contruir el contenedor.

```bash
# crear el contenedor
# emparejar los puertos para levantar servidor nginx.

docker run -d --name ubuntu-nginx-01 -p 8082:81 ubuntu-nginx-01

# modo interactivo
docker run -it ubuntu-nginx-01

# Detener contenedor
docker stop ubuntu-nginx-01

# Reiniciar contenedor
docker restart ubuntu-nginx-01

# Elimnar contenedor (probar que la imágen quedó bien construida)
docker rm ubuntu-nginx-01

# Eliminar imagen (para el ejemplo de kubernetes no es necesario borrarla)
docker rmi ubuntu-nginx-01
```

- Construir un registro local

```yml
# Registro local
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Etiquetar imagen
docker tag ubuntu-nginx-01:latest localhost:5000/ubuntu-nginx-01:latest


# Empujar imagen
docker push localhost:5000/ubuntu-nginx-01:latest

# Probar que si exista en el registro local
curl http://localhost:5000/v2/_catalog
```

- Construir el deployment

```bash
# Crear el deployment en el cluster
kubectl apply -f deployment.yaml

# Crear el servicio en el cluster
kubectl apply -f service.yaml

# Obtener el servicio
kubectl get services

# Obtener el pod
kubectl get pods -o wide

# Obtener el deployments
kubectl get deployments

# Describir el pod
kubectl describe pod ubuntu-nginx-01

# Borrar el deployment en el cluster
kubectl delete deployment/ubuntu-nginx-01-deployment service/ubuntu-nginx-01-service
```
