# Cria-o-de-um-Cluster-do-MongoDB-no-Docker

Configuração de um ReplicaSet no MongoDB com Docker

DENTRO DO TERMINAL DO DOCKER

 1 -> Criar a rede do cluster
Este comando cria uma rede Docker para que os nós do MongoDB possam se comunicar entre si.

docker network create mongoClusterTrabalho

 2 -> Criar os 4 nós do ReplicaSet
Execute cada comando individualmente para criar os nós do cluster:

docker run -d --rm -p 27018:27017 --name mongo10 --network mongoClusterTrabalho mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo10

docker run -d --rm -p 27019:27017 --name mongo20 --network mongoClusterTrabalho mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo20

docker run -d --rm -p 27020:27017 --name mongo30 --network mongoClusterTrabalho mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo30

docker run -d --rm -p 27021:27017 --name mongo40 --network mongoClusterTrabalho mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo40


3 -> Abrir o terminal do primeiro nó e obter a string de conexão
A string de conexão será necessária para o MongoDB Compass.

docker exec -it mongo10 mongosh


Execute o seguinte comando dentro do terminal do MongoDB:

db.runCommand({hello:1})


A saída do comando exibirá a string de conexão, que será algo como:
	"mongodb://127.0.0.1:27018/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.8"

Salve essa string, pois será utilizada no MongoDB Compass.

---

 **DENTRO DO TERMINAL DO 1° NÓ (mongo10)**

1 -> Criar o ReplicaSet
No terminal do primeiro nó, execute:
rs.initiate({
  _id: "myReplicaSet",
  members:[
    {_id:0, host: "mongo10"},
    {_id:1, host: "mongo20"},
    {_id:2, host: "mongo30"},
    {_id:3, host: "mongo40"}
  ]
})


2 -> Sair do terminal do 1° nó
exit

## *DENTRO DO TERMINAL DO DOCKER*

1 -> Testar e verificar os membros do ReplicaSet;
Execute o seguinte comando para verificar o status do cluster:

docker exec -it mongo10 mongosh --eval "rs.status()"

## *DENTRO DO MONGODB COMPASS*

1 -> Conectar ao cluster através do 1° nó
Abra o MongoDB Compass, vá para New Connection e insira a string de conexão obtida anteriormente no campo URI.
Sugestão: renomeie o NAME para "mongo10" para facilitar a identificação.

---

## DENTRO DO TERMINAL DO 1° NÓ, NO MONGODB COMPASS

1 -> Verificar qual nó é o primário
O nó primário é o responsável por aceitar gravações. Execute:
	rs.isMaster().primary
Se o retorno for "mongo10:27018", significa que o nó mongo10 é o primário.

2 -> Criar um database
use CorporeSystem


3 -> Inserir um documento de teste (apenas no nó primário)

db.cliente.insertOne({codigo:1, nome: "Ana Maria Teste"})

4 -> Verificar os registros inseridos

db.cliente.find()

---

**DENTRO DO TERMINAL DO DOCKER

1 -> Derrubar o nó primário
docker stop mongo10


2 -> Verificar o status do cluster
É necessário identificar qual nó assumiu a posição de primário.

docker exec -it mongo20 mongosh --eval "rs.status()"


O nó que estiver com stateStr: "PRIMARY" será o novo primário.

---

DENTRO DO TERMINAL DO 1° NÓ, NO MONGODB COMPASS

1 -> Tentar inserir no nó desligado (deve dar erro)

db.cliente.insertOne({codigo:6, nome: "Teste dos Santos Junior"})

Erro esperado: *ECONNREFUSED*.

---

**DENTRO DO MONGODB COMPASS

1 -> Conectar ao novo nó primário

- Vá para *New Connection* no MongoDB Compass.
- Utilize a string de conexão alterando a porta para a do novo nó primário.
- Renomeie o *NAME* para o nome do novo nó primário.

---

**DENTRO DO TERMINAL DO NOVO NÓ PRINCIPAL, NO MONGODB COMPASS

1 -> Verificar qual é o nó primário

rs.isMaster().primary
Deve retornar o nome do nó recém-promovido.

2 -> Selecionar o banco de dados
use prod

3 -> Inserir um novo documento de teste

db.cliente.insertOne({codigo:3, nome: "Ana Clara Farias"})


4 -> Verificar os registros inseridos

db.cliente.find()


**DENTRO DO TERMINAL DO DOCKER

1 -> Restaurar o nó que foi derrubado

docker run -d --rm -p 27018:27017 --name mongo10 --network mongoClusterTrabalho mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo10


Após a restauração, o nó voltará a fazer parte do cluster automaticamente.

Digitar o comando abaixo para verificar o estado do cluster:

docker exec -it mongo20 mongosh --eval "rs.status()"
