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

<h2>OBTENER LOS USUARIOS DE UNA BASE DE DATOS </h2>
<pre>
use basededatos
db.getUsers()
</pre>

<h2> ELIMINAR USUARIO <h2>
<pre>
db.dropUser("nombreusuario")
</pre>

<h1>COMANDOS BÁSICOS</h1>

CREAR BASE DE DATOS
<pre>
use test_indus;
</pre>

INTRODUCIR DATOS EN LA BASE DE DATOS
<pre>
db.test_indus.save({id:'1'});
</pre>

LISTAR DATOS DE UNA BASE DE DATOS
db.test_indus.find();

BORRAR BASE DE DATOS
db.dropDatabase()

ELIMINAR UN DATO DENTRO DE UNA BASE DE DATOS
db.test_indus.remove({id:'4'})

HACER UN PING A LA BASE MONGO
mongo admin -u {{db_mongo_user_backup}} -p {{db_mongo_password_backup}} --authenticationDatabase admin --eval "db.runCommand({ping:1})"

BACKUPS 

COMANDO MONGO DUMP
mongodump -u {{db_mongo_user_backup}} -p {{db_mongo_password_backup}} -h {{ db_mongo_compose_name }} --port {{db_mongo_port}} --gzip --archive=/save/cronned_backup_mongo_db.gz

RESTORE

COPIAR BACKUP AL CONTENEDOR DE BAS DE DATOS
docker cp backup_mongo_db.gz nombrecontenedor:/tmp/

RESTAURAR DESDE EL NODO
docker exec -it nombrecontenedor -u usuario -p contraseña --gzip --archive=/tmp/backup_mongo_db.gz 

ESTAURAR DESDE DENTRO DEL CONTENEDOR
mongorestore -u usuario -p contraseña --gzip --archive=/tmp/backup_mongo_db.gz

((    si restauras un backup que está compuesto de muchos *.gz puedes realizar el siguiente comando:
mongorestore --port 27017 -d basedatos /docker-entrypoint-initdb.d/backup/     ))

COMANDOS MONGO DESDE SHELL
Hay dos formas. Mediante EOF y mediante --eval

	1. EOF: El ejemplo está en el script que puse arriba
	2. EVAL: mongo admin -u usuarioadmin -p passusuarioadmin--eval "db.createUser({ user: 'usuarioacrear', pwd: 'passusuarioacrear', roles: [ { role: 'root', db: 'admin' } ] })";

COMO MONTAR UN REPLICASET CON SHARDING (Ejemplo: PSATracker)

												⚠ Recomendable hacer un backup de la base de datos ⚠
												Listado estructura que tenemos montada: 
													w Nodo 1: contiene 3 contenedores
														w contenedor-mongors1n1
														w contenedor-mongocfg1
														w contenedor-mongos1
													w Nodo 2: contiene 3 contenedores
														w contenedor-mongors1n2
														w contenedor-mongocfg2
														w contenedor-mongos2
													w Nodo 3: contiene 3 contenedores
														w contenedor-mongors1n3
														w contenedor-mongocfg3
														w contenedor-mongos3

	1. docker exec -it contenedor-mongocfg1 bash
mongo <<EOF
rs.initiate( {_id: "mongors1conf", configsvr: true, members: [
{ "_id" : 0, host : "contenedor-mongocfg1" },
{ "_id" : 1, host : "contenedor-mongocfg2" },
{ "_id" : 2, host : "contenedor-mongocfg3" }]});
EOF
exit
	2. docker exec -it contenedor-mongors1n1 bash
mongo <<EOF
rs.initiate({_id : 'mongors1', members: [
{ _id : 0, host : 'contenedor-mongors1n1' },
{ _id : 1, host : 'contenedor-mongors1n2' },
{ _id : 2, host : 'contenedor-mongors1n3' }]});
EOF
exit

3. docker exec -it contenedor-mongos1 bash
mongo <<EOF
sh.addShard('mongors1/contenedor-mongors1n1');
EOF
exit

=================================================================================
											TESTEO
											Si hasta aquí no petó nada, lo siguiente que puedes hacer es dentro 
											del terminal mongo de contenedor-mongos1
											Hacemos un checkeo general
											docker exec -it contenedor-mongos1 bash
											mongo <<EOF
											sh.status();
											EOF
											exit
											FIN TESTEO
=================================================================================
						⚠ Si has realizado un backup, ahora realizamos un restore ⚠
						
						Copiamos los gz a la carpeta compartida
						cp /users/backup/* /users/backup
						cd /users/backup
						gzip -d *.gz
						Restauración BBDD --> Tocaría restaurar la copia de seguridad en contenedor-mongors1n1
						docker exec -it contenedor-mongors1n1 bash
						mongorestore --port 27017 -d basedatos /docker-entrypoint-initdb.d/backup/
						exit
=================================================================================
	4. docker exec -it contenedor-mongos1 bash
mongo <<EOF
sh.enableSharding('psaTracker');
EOF
exit
Y con esto estaría todo 👌

COMO MONTAR SOLO REPLICACION MONGO (Ejemplo: MuseSTD)

3 nodos 3 contenedores mongo, cada 1 se ata a un nodo.
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


¿Qué comando estamos ejecutando en los contenedores?

db_mongo_command_repl_primary: "/docker-entrypoint-initdb.d/01-init_replica.sh" 
db_mongo_command_repl_secondary: "mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}}"
db_mongo_command_repl_arbiter: "mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}}"

Variables :)

db_mongo_config_path: "{{ appli_remote_files_path }}/dbconfig"
db_mongo_config_container_file: "/mongoconf/mongodb.conf"
db_mongo_log_file: "/mongoconf/mongodb.log"
db_mongo_replicaset: "replica"


Volúmenes que tenemos en los servicios del compose en esos contenedores:
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

Variables :)
db_mongo_data_path: "{{appli_remote_root_path}}/data"
db_mongo_shared_folder: "/data/db"

Scripts y config que necesitamos:
mongo_keyfile
mongodb.conf
01-init_replica.sh 
02-Replicaset.sh
03-BackupUserCreate.sh

¿Qué contenido tienen estos archivos?

cat mongo_keyfile
123456789

cat mongodb.conf
security:
  keyFile: /mongoconf/mongo_keyfile
net:
   port: 27017

cat 01-init_replica.sh 
#!/bin/bash
mongod --smallfiles --config {{db_mongo_config_container_file}} --replSet {{db_mongo_replicaset}} --bind_ip_all --fork --logpath /mongoconf/mongodb.log
while :; do sleep 1; done 

cat 02-Replicaset.sh
#!/bin/bash
mongo << EOF
rs.initiate( {_id: "replica", members: [
{ "_id" : 0, host : "contenedor-mongodb_primary:27017" },
{ "_id" : 1, host : "contenedor-mongodb_secondary:27017" },
{ "_id" : 2, host : "contenedor-mongodb_arbiter:27017" }]});
EOF

Cat 03-BackupUserCreate.sh
#!/bin/bash
mongo << EOF
use admin
db.createUser({ user: '{{db_mongo_user_backup}}', pwd: '{{db_mongo_password_backup}}', roles: [ { 'role' : 'root', 'db' : 'admin' } ] }); 
EOF

Importante hay que añadir en el launch después del prepare env las siguientes tasks.
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

Una vez realizado el deploy, nos conectamos al mongo primario.

</pre>
