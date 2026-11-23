# Secret de MySQL

El objeto reservas-db-secret debe existir en el namespace
reservas-app antes de aplicar los Deployments.

Claves necesarias:

- MYSQL_USER
- MYSQL_PASSWORD
- MYSQL_ROOT_PASSWORD

Crear el Secret mediante kubectl create secret generic
utilizando las credenciales correspondientes al entorno.

No guardar contraseñas reales en Git.

Comprobar:

    kubectl get secret reservas-db-secret -n reservas-app
