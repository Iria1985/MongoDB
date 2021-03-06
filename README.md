## MongoDB
## VARIABLES USADAS
<pre>
db_mongo_compose_name: "{{appli_url}}-db"
db_mongo_database_name: "nombredelaplicacion"
db_mongo_username: "usuarioadmin"
db_mongo_user_password: "passadmin"
db_mongo_user_backup: "usuariobackup"
db_mongo_password_backup: "passusuariobackup"
</pre>

## GESTIÓN USUARIOS

### SCRIPT DE CREACIÓN DE USUARIO CON PERMISOS PARA DUMP

```
#!/bin/bash
mongo admin -u {{db_mongo_username}} -p {{db_mongo_user_password}} <<EOF
use admin
db.createUser({ user: '{{db_mongo_user_backup}}', pwd: '{{db_mongo_password_backup}}', roles: [ { 'role' : 'root', 'db' : 'admin' } ] }); 
EOF
```

### OBTENER LOS USUARIOS DE UNA BASE DE DATOS
<pre>
use basededatos
db.getUsers()
</pre>

### ELIMINAR USUARIO
<pre>
db.dropUser("nombreusuario")
</pre>

## COMANDOS BÁSICOS

### CREAR BASE DE DATOS
<pre>
use test_indus;
</pre>

### INTRODUCIR DATOS EN LA BASE DE DATOS
<pre>
db.test_indus.save({id:'1'});
</pre>

### LISTAR DATOS DE UNA BASE DE DATOS
<pre>
db.test_indus.find();
</pre>

### BORRAR BASE DE DATOS
<pre>
db.dropDatabase()
</pre>

### ELIMINAR UN DATO DENTRO DE UNA BASE DE DATOS
<pre>
db.test_indus.remove({id:'4'})
</pre>

### HACER UN PING A LA BASE MONGO
<pre>
mongo admin -u {{db_mongo_user_backup}} -p {{db_mongo_password_backup}} --authenticationDatabase admin --eval "db.runCommand({ping:1})"
</pre>

## BACKUPS 

### COMANDO MONGO DUMP
<pre>
mongodump -u {{db_mongo_user_backup}} -p {{db_mongo_password_backup}} -h {{ db_mongo_compose_name }} --port {{db_mongo_port}} --gzip --archive=/save/cronned_backup_mongo_db.gz
</pre>

## RESTORE

### COPIAR BACKUP AL CONTENEDOR DE BAS DE DATOS
<pre>
docker cp backup_mongo_db.gz nombrecontenedor:/tmp/
</pre>

### RESTAURAR DESDE EL NODO
<pre>
docker exec -it nombrecontenedor -u usuario -p contraseña --gzip --archive=/tmp/backup_mongo_db.gz 
</pre>

### RESTAURAR DESDE DENTRO DEL CONTENEDOR
``` 
mongorestore -u usuario -p contraseña --gzip --archive=/tmp/backup_mongo_db.gz

((    si restauras un backup que está compuesto de muchos *.gz puedes realizar el siguiente comando:
mongorestore --port 27017 -d basedatos /docker-entrypoint-initdb.d/backup/     ))

COMANDOS MONGO DESDE SHELL
Hay dos formas. Mediante EOF y mediante --eval

	1. EOF: El ejemplo está en el script que puse arriba
	2. EVAL: mongo admin -u usuarioadmin -p passusuarioadmin--eval "db.createUser({ user: 'usuarioacrear', pwd: 'passusuarioacrear', roles: [ { role: 'root', db: 'admin' } ] })";
```

### COMO MONTAR UN REPLICASET CON SHARDING

⚠ Recomendable hacer un backup de la base de datos ⚠
Listado estructura que tenemos montada: 
* Nodo 1: contiene 3 contenedores
  * contenedor-mongors1n1
  * contenedor-mongocfg1
  * contenedor-mongos1
* Nodo 2: contiene 3 contenedores
  * contenedor-mongors1n
  * contenedor-mongocfg2
  * contenedor-mongos2
* Nodo 3: contiene 3 contenedores
  * contenedor-mongors1n3
  * contenedor-mongocfg3
  * contenedor-mongos3
```
1. docker exec -it contenedor-mongocfg1 bash
mongo <<EOF
rs.initiate( {_id: "mongors1conf", configsvr: true, members: [
{ "_id" : 0, host : "contenedor-mongocfg1" },
{ "_id" : 1, host : "contenedor-mongocfg2" },
{ "_id" : 2, host : "contenedor-mongocfg3" }]});
EOF
exit
```
2. docker exec -it contenedor-mongors1n1 bash
```
mongo <<EOF
rs.initiate({_id : 'mongors1', members: [
{ _id : 0, host : 'contenedor-mongors1n1' },
{ _id : 1, host : 'contenedor-mongors1n2' },
{ _id : 2, host : 'contenedor-mongors1n3' }]});
EOF
exit
```
3. docker exec -it contenedor-mongos1 bash
```
mongo <<EOF
sh.addShard('mongors1/contenedor-mongors1n1');
EOF
exit
```
==============================================================================
## TESTEO
<pre>
Si hasta aquí no petó nada, lo siguiente que puedes hacer es dentro 
del terminal mongo de contenedor-mongos1
Hacemos un checkeo general
</pre>
```
docker exec -it contenedor-mongos1 bash
mongo <<EOF
sh.status();
EOF
exit
```
## FIN TESTEO

<pre>
⚠ Si has realizado un backup, ahora realizamos un restore ⚠

Copiamos los gz a la carpeta compartida
cp /users/backup/* /users/backup
cd /users/backup
gzip -d *.gz
Restauración BBDD --> Tocaría restaurar la copia de seguridad en contenedor-mongors1n1
docker exec -it contenedor-mongors1n1 bash
mongorestore --port 27017 -d basedatos /docker-entrypoint-initdb.d/backup/
exit
</pre>
=================================================================================
4. docker exec -it contenedor-mongos1 bash
```
mongo <<EOF
sh.enableSharding('psaTracker');
EOF
exit
```
<pre>
Y con esto estaría todo 👌
</pre>

## COMO MONTAR SOLO REPLICACION MONGO
<pre>
3 nodos 3 contenedores mongo, cada 1 se ata a un nodo.
</pre>
```
container_name: "{{db_mongo_compose_name}}_primary"
 environment:
 - "constraint:node=={{ node1 }}"
command: {{db_mongo_command_repl}}

container_name: "{{db_mongo_compose_name}}_secondary"
 environment:
 - "constraint:node=={{ node2 }}"
command: {{db_mongo_command_repl}}

container_name: "{{db_mongo_compose_arbiter}}_primary"
 environment:
 - "constraint:node=={{ node3 }}"
command: {{db_mongo_command_repl}}
```

#### ¿Qué comando estamos ejecutando en los contenedores?
```
db_mongo_command_repl_primary: "/docker-entrypoint-initdb.d/01-init_replica.sh" 
db_mongo_command_repl_secondary: "mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}}"
db_mongo_command_repl_arbiter: "mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}}"
```
<pre>
Variables :)
</pre>
```
db_mongo_config_path: "{{ appli_remote_files_path }}/dbconfig"
db_mongo_config_container_file: "/mongoconf/mongodb.conf"
db_mongo_log_file: "/mongoconf/mongodb.log"
db_mongo_replicaset: "replica"
```
<pre>
Volúmenes que tenemos en los servicios del compose en esos contenedores:
</pre>
```
  "{{db_mongo_compose_name}}_primary":
    volumes:
      - "{{db_mongo_data_path}}/db:{{db_mongo_shared_folder}}"
      - "{{db_mongo_config_path}}:/mongoconf"
      - "{{db_mongo_data_path}}/init:/docker-entrypoint-initdb.d/"
  "{{db_mongo_compose_name}}_secondary":
    volumes:
      - "{{db_mongo_data_path}}/db:{{db_mongo_shared_folder}}"
      - "{{db_mongo_config_path}}:/mongoconf"
  "{{db_mongo_compose_name}}_arbiter":
    volumes:
      - "{{db_mongo_data_path}}/db:{{db_mongo_shared_folder}}"
      - "{{db_mongo_config_path}}:/mongoconf"
```
<pre>
Variables :)
</pre>
```
db_mongo_data_path: "{{appli_remote_root_path}}/data"
db_mongo_shared_folder: "/data/db"
```
<pre>
Scripts y config que necesitamos:
</pre>
```
mongo_keyfile
mongodb.conf
01-init_replica.sh 
02-Replicaset.sh
03-BackupUserCreate.sh
```
#### ¿Qué contenido tienen estos archivos?
<pre>
cat mongo_keyfile
123456789

cat mongodb.conf
security:
  keyFile: /mongoconf/mongo_keyfile
net:
   port: 27017

cat 01-init_replica.sh
</pre>
```
#!/bin/bash
mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}} --bind_ip_all --fork --logpath /mongoconf/mongodb.log
while :; do sleep 1; done 
```
cat 02-Replicaset.sh
```
#!/bin/bash
mongo << EOF
rs.initiate( {_id: "replica", members: [
{ "_id" : 0, host : "contenedor-mongodb_primary:27017" },
{ "_id" : 1, host : "contenedor-mongodb_secondary:27017" },
{ "_id" : 2, host : "contenedor-mongodb_arbiter:27017" }]});
EOF
```
Cat 03-BackupUserCreate.sh
```
#!/bin/bash
mongo << EOF
use admin
db.createUser({ user: '{{db_mongo_user_backup}}', pwd: '{{db_mongo_password_backup}}', roles: [ { 'role' : 'root', 'db' : 'admin' } ] }); 
EOF
```
Importante hay que añadir en el launch después del prepare env las siguientes tasks.
```
- hosts: "{{ target | default('nodes')}}"
  become: yes
  become_method: sudo
  tasks:  
  - name: Giving right owner to dbconfig mongo files
    file:
      path: "{{ db_mongo_config_path }}/mongo_keyfile"
      owner: j588818
      group: 100999
      mode: 0600      

  - name: Giving right owner to dbconfig mongo files
    file:
      path: "{{ db_mongo_config_path }}/mongodb.conf"
      owner: 999
      group: 999
      mode: 0644 
```
Una vez realizado el deploy, nos conectamos al mongo primario.
