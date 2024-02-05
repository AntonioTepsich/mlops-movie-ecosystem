# Init
1- Clonar repositorio
2- Seguir los siguientes pasos


- Tener en cuenta que todo este repositorio fue creado en Ubuntu 22.04
- Usar DBeaver para ver las modificaciones de las Bases de Datos


## Definición de variables de entorno
setear environments
```bash
#cambiar los distintos path acordes a la ubicacion de tu repositorio
postgres_data_folder=/home/antonio/Desktop/Recommender_movie_system/mlops-ecosystem/dbs/data/postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_HOST=localhost:5432
MLFLOW_POSTGRES_DB=mlflow_db_movie
MLFLOW_ARTIFACTS_PATH=/home/antonio/Desktop/Recommender_movie_system/mlops-ecosystem/mlflow_data
```



ubicarse en el path del repositorio 
```bash
# Seteo de variables
set -o allexport && source .env && set +o allexport

# Verificarlo
echo $postgres_data_folder
```

## PSQL DB

### Bajar imagen
```bash
docker pull postgres
```

### Correr imagen
Podes cambiar el nombre del container
```bash
docker run -d \
    --name mlops-postgres \
    -e POSTGRES_PASSWORD=$POSTGRES_PASSWORD \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v $postgres_data_folder:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres
```

### Verificar funcionamiento, y manejo del contenedor
```bash
docker ps

docker ps -a

#cambiar nombre del container si lo modificaste antes
docker exec -it mlops-postgres /bin/bash

#ejecutar este comando dentro
root@08487b094f8a:/#  psql -U postgres

root@08487b094f8a:/#  exit
```

### Create MLFLOW DB
ingresar la contrasena anteriormente seteada en environments
```bash
psql -U postgres -h localhost -p 5432
```

Ejecutar dentro de la base de datos, luego de ejecutar el comando anterior
```sql
CREATE DATABASE mlflow_db_movie;
CREATE USER mlflow_user WITH ENCRYPTED PASSWORD 'mlflow';
GRANT ALL PRIVILEGES ON DATABASE mlflow_db_movie TO mlflow_user;
```


## Mlflow server

### Creación de entorno e instalación de MLFLOW
```bash
conda create -n mlflow-server python=3.9
conda activate mlflow-server

pip install -r requirements.txt
```

```bash
# desde la carpeta del proyecto

mlflow server --backend-store-uri postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@$POSTGRES_HOST/$MLFLOW_POSTGRES_DB --default-artifact-root $MLFLOW_ARTIFACTS_PATH -h 0.0.0.0 -p 8002
```
Abrir browser en http://localhost:8002/

## Airbyte
salgo de este ropositorio y solamente instalo airbyte en otro nuevo archivo
```bash
# clone Airbyte from GitHub
git clone --depth=1 https://github.com/airbytehq/airbyte.git

# switch into Airbyte directory
cd airbyte

# start Airbyte
./run-ab-platform.sh
```
Abrir browser en http://localhost:8000/

username: `airbyte`
password: `password`

### Path de source (csvs)
https://raw.githubusercontent.com/mlops-itba/Datos-RS/main/data/peliculas_0.csv
https://raw.githubusercontent.com/mlops-itba/Datos-RS/main/data/usuarios_0.csv
https://raw.githubusercontent.com/mlops-itba/Datos-RS/main/data/scores_0.csv

### Creación de destination (psql)

```bash
psql -U postgres -h localhost -p 5432
```

```sql
CREATE DATABASE mlops_movie;
CREATE USER airbyte WITH ENCRYPTED PASSWORD 'airbyte';
GRANT ALL PRIVILEGES ON DATABASE mlops_movie TO airbyte;
GRANT ALL ON SCHEMA public TO airbyte;
GRANT USAGE ON SCHEMA public TO airbyte;
ALTER DATABASE mlops OWNER TO airbyte;
```

## dbT
### Crear entorno

```bash
conda create -n mlops-dbt python=3.9
conda activate mlops-dbt
pip install dbt-postgres

dbt --version

dbt init db_postgres
```

En el archivo de configuración `~/.dbt/profiles.yml` se debe configurar la base de datos
```yaml
dbt_elt:
  outputs:
    dev:
      type: postgres
      threads: 1
      host: localhost
      port: 5432
      user: postgres
      pass: mysecretpassword
      dbname: mlops
      schema: target
```

### Testear conexión
```bash
dbt debug

dbt run
```

### Opcional 
Conectar base de datos de MongoDB 
```bash
# Mongo

### Desde cloud gratis
https://cloud.mongodb.com/v2/653ac4dcf923b06a3d61bfcc#/overview

### Desde docker
```bash
docker pull mongo

docker run \
    --name mlops-mongo \
    -v $mongo_data_folder:/data/db \
    -p 27017:27017 \
    mongo

docker exec -it mlops-mongo /bin/bash
```

```bash
mongo
test> show dbs
test> use mlops
test> db.createUser({
    user: "airbyte",
    pwd: "airbyte",
    roles: [ { role: "userAdmin", db: "mlops" } ]
})

test> use admin
test> db.createUser(
  {
    user: "admin",
    pwd: "admin",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```