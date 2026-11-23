# reservas-app

Proyecto realizado para el curso de Contenedores, Docker y Orquestación con Kubernetes.

La aplicación permite reservar salas eligiendo una fecha y una franja horaria.

Para desarrollarla utilicé React y Vite para el frontend, Node.js y Express para la API y MySQL 8.4 para guardar las reservas.

También utilicé Docker, Docker Compose, Kubernetes y Helm para ejecutar y desplegar la aplicación.

## Estructura del proyecto

- `backend/`: código de la API y su Dockerfile.
- `frontend/`: aplicación React y su Dockerfile.
- `docker-compose.yaml`: configuración de Docker Compose.
- `k8s/`: manifiestos de Kubernetes.
- `helm/reservas-app/`: chart de Helm.
- `ARQUITECTURA.md`: respuestas a las preguntas del proyecto final.
- `INFORME.md`: pruebas y resultados de funcionamiento.

## 1. Ejecutar con Docker Compose

Para ejecutar la aplicación necesito tener instalados Docker y Docker Compose.

Desde la carpeta principal del proyecto ejecuto:

```bash
docker compose up -d --build
```

Para comprobar que los contenedores estén funcionando:

```bash
docker compose ps
```

Después puedo ingresar a la aplicación desde el navegador utilizando el puerto del frontend configurado en `docker-compose.yaml`.

Para detener los contenedores:

```bash
docker compose down
```

Los datos de MySQL quedan guardados en un volumen, por lo que no se pierden al detener y volver a iniciar los contenedores.

## 2. Ejecutar con Kubernetes y Helm

Para esta opción necesito tener acceso al clúster Kubernetes y Helm instalado.

El clúster también debe contar con el almacenamiento necesario y un Ingress Controller configurado.

Primero compruebo la conexión:

```bash
kubectl get nodes
```

Para instalar la aplicación utilizo el chart que está en `helm/reservas-app/`.

Las credenciales de MySQL las configuro mediante un archivo privado llamado `values-tp8.yaml`, que no subo al repositorio.

Desde la carpeta principal del proyecto ejecuto:

```bash
helm upgrade --install reservas-app ./helm/reservas-app \
  --namespace reservas-app \
  --create-namespace \
  -f ~/Curso_Docker/privado/values-tp8.yaml \
  --wait \
  --timeout 5m
```

Para comprobar la instalación:

```bash
helm status reservas-app -n reservas-app
```

También puedo ver los recursos creados:

```bash
kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app
```

Una vez que el Ingress esté funcionando y tenga configurado el nombre en el archivo `hosts`, puedo ingresar desde:

`http://reservas.local`

## 3. Configuración

En Kubernetes utilicé ConfigMaps para la configuración de la API y el frontend, y un Secret para las credenciales de MySQL.

También configuré un PVC para conservar los datos de la base de datos cuando se recrea el Pod.

La API tiene un HPA que permite aumentar o disminuir las réplicas según el consumo de CPU.

## 4. Helm

El chart de Helm contiene las plantillas de los doce objetos utilizados en el proyecto.

Desde `values.yaml` puedo modificar configuraciones como las imágenes, las réplicas y el HPA.

También puedo actualizar la aplicación con `helm upgrade` y volver a una versión anterior con `helm rollback`.

## 5. Documentación

En `ARQUITECTURA.md` dejé las respuestas a las preguntas del proyecto final y los problemas que tuve durante los prácticos.

En `INFORME.md` incluí las pruebas y los resultados de funcionamiento.

