# cloudops-kubernetes
Projeto com objetivo de executar uma aplicação PostgREST integrada a um PostgreSQL em Kubernetes, utilizando recursos como Namespace, Deployment, Service, Secret, ConfigMap, PVC e probes de saúde.

## Tecnologias
- Kubernetes
- PostgREST
- PostgreSQL 16
- kubectl
- curl
- minikube

## Funcionamento

A aplicação é composta por dois Deployments principais, PostgREST e PostgreSQL, e funcionam da seguinte forma:

- O PostgreSQL é executado por um Deployment e utiliza um PersistentVolumeClaim (PVC) para armazenar os dados. O `postgres-service` fornece um endereço estável para que outros recursos do cluster possam acessar o banco.

- O PostgREST também é executado por um Deployment, com 3 réplicas. Ele se conecta ao PostgreSQL utilizando o nome do Service (`postgres-service`) na string de conexão, em vez de utilizar diretamente o IP do Pod.

- A API é exposta externamente através do `postgrest-service`, que utiliza um NodePort na porta 30001. As requisições recebidas são encaminhadas para uma das réplicas do PostgREST, que consulta ou altera os dados no PostgreSQL.

- As credenciais utilizadas pelo PostgreSQL e pelo PostgREST são armazenadas em um Secret, enquanto a configuração de inicialização do banco é definida em um ConfigMap.

- Além disso foram configuradas liveness e readiness probs para o Deployment do PostgREST. A readiness probe verifica se o Pod está pronto para receber tráfego, enquanto a liveness probe verifica se a aplicação continua funcionando corretamente, permitindo que o Kubernetes reinicie o container caso ele apresente uma falha persistente.

## Criando o ambiente

#### 1. Crie o arquivo `k8s/secrets.yml` 

O arquivo `k8s/secrets.yml` define variáveis de ambiente usadas pelos pods, como são informações sensíveis não foi enviado para o repositório. Crie o arquivo com as informações abaixo e substitua as variáveis com os valores adequados

```yml 
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: cloudops-kubernetes
type: Opaque
stringData:
  POSTGRES_USER: your-user
  POSTGRES_PASSWORD: your-password
  POSTGRES_DB: your-db
  PGRST_DB_URI: postgres://your-user:your-password@postgres-service:5432/your-db
```

#### 2. Aplique os manifests

Após criar o arquivo de secrets, o comando seguinte cria todos os objetos dos manifests:

```bash
kubectl apply -f k8s/
```

#### 3. Verifique

Para verificar se todos os objetos dos manifests foram criados rode o comando:

```bash
kubectl get all -n cloudops-kubernetes
```

## Testando a aplicação

#### Encontrando o IP

Para acessar a API primeiro é preciso saber o IP do Node, com o seguinte comando:

```bash
kubectl get nodes -o wide
```

Com o valor de `INTERNAL IP` ou `EXTERNAL IP` (depende do ambiente) podemos acessar a API com esse IP:
(Substitua \<NodeIP> pelo IP encontrado)

```bash
curl <NodeIP>:30001/
```

A tabela criada na aplicação é `users`, que está disponível em `/users`, podendo ser acessada:

```bash
curl <NodeIP>:30001/users
```

#### Inserindo dados

Para inserir um novo usuário, utilize uma requisição POST:

```bash 
curl -X POST <NodeIP>:30001/users -H "Content-Type: application/json" -d '{"name":"User","email":"user@example.com"}'
```

Depois, consulte a tabela novamente:

```bash
curl <NodeIP>:30001/users
```

O usuário inserido deve aparecer na resposta.

#### Testando persistência

Para testar se o volume do banco criado é realmente persistente, podemos deletar o Pod, e ver se os dados ainda existem.

Primeiro é necessário descobrir o nome do Pod

```bash
kubectl get pods -n cloudops-kubernetes
```

O resultado deverá aparecer algo como:
```console
NAME                         READY   STATUS    RESTARTS   AGE
postgresql-db69568db-6sv7q   1/1     Running   0          28m
postgrest-644cf6dc47-44gc6   1/1     Running   0          28m
postgrest-644cf6dc47-4zmnj   1/1     Running   0          28m
postgrest-644cf6dc47-v48ct   1/1     Running   0          28m
```

O Pod do banco de dados é o postgresql-xxxxxxxxx-xxxxx, podemos deletar com:

```bash
kubectl delete pod -n cloudops-kubernetes postgresql-xxxxxxxxx-xxxxx
```

E ao esperar o Pod subir novamente, e os Pods da API ficarem `ready` novamente, testar se os dados ainda persistem com:

```bash
curl <NodeIP>:30001/users
```

#### Testando HPA
Para testar o HPA é necessário habilitar o `metrics-server` da sua ferramenta, por conta do HPA usar as métricas registradas.

Para verificar se o `metrics-server` está habilitado:

```bash
kubectl top pods -n cloudops-kubernetes
```

O HPA está configurado para no mínimo 3 réplicas, podendo aumentar até 6. O critério de scale up é se passar de 50% do uso de CPU.

Para acompanhar em tempo real:

```bash
kubectl get hpa -n cloudops-kubernetes -w
```

E com uma ferramenta de estresse (como o ApacheBench do exemplo) pode-se gerar carga e ver as replicas sendo criadas

Exemplo com ApacheBench:

```bash 
ab -n 100000 -c 500 <NodeIP>:30001/users
```

Enquanto essa carga é gerada é possível ver com o comando anterior as replicas aumentando

## Evidências

Algumas evidências da API funcionando e persistência dos dados. Os exemplos com curl foi utilizado o `jq` para melhor visualização da resposta pelo terminal

#### Recursos em execução
Esse print mostra todos os recursos criados com o manifest
![Mostrando todos os recursos em execução](./docs/screenshots/get-all.png)

#### Testando a API (Inserção de dados)
Aqui é verificado se a API consegue buscar dados no banco e se é possível inserir dados

![Inserindo dados na API](./docs/gifs/teste-api-insercao.gif)

#### Testando persistência
Aqui o Pod de banco de dados vai ser deletado, e quando subir novamente podemos ver se os dados persistiram

![Testando persistência do volume](./docs/gifs/teste-persistencia-volume.gif)

#### Teste do HPA do Postgrest
Aqui foi realizado um teste com apache-ab para estressar a aplicação e ver se o HPA cria novas réplicas

![Testando HPA no Deployment do PostgREST](./docs/gifs/hpa-teste.gif)

## Reflexões

Algumas reflexões sobre cada nível do desafio

#### 1. Namespace e primeiro contato
**Reflexão:** ao deletar esse Pod avulso, ele volta sozinho? Por quê? Isso te diz algo sobre por que raramente criamos Pods diretamente.

> Um pod criado diretamente não é recriado automaticamente quando deletado, quem faz isso normalmente é um controlador como um Deployment ou ReplicaSet, por isso raramente Pods são criados diretamente

#### 2. Banco de dados com persistência
**Reflexão:** qual a diferença entre montar um PVC e um emptyDir? O que aconteceria com os dados em cada caso ao deletar o Pod? 

> O emptyDir tem seus dados vinculados ao Pod, e são perdidos quando o Pod é deletado. Já quando um PVC é criado, mesmo quando o Pod é deletado os dados ainda persistem

#### 3. Banco de dados com persistência
**Reflexão:** ao inspecionar o Secret com -o yaml , o valor aparece "embaralhado". Isso é criptografia de verdade ou apenas codificação? O que isso significa para a segurança real? 

> O valor é codificado em Base64, e não criptografado de verdade. Por isso mesmo usando secrets é importante não versionar as credenciais

#### 4. A API conectada ao banco (a integração)
**Reflexão:** por que usamos o nome do Service do Postgres na string de conexão, em vez do IP do Pod? O que aconteceria com a conexão se você usasse o IP e o Pod do banco fosse recriado? 

> Utilizamos o nome do Service (postgres-service) porque ele fornece um endereço estável para acessar o PostgreSQL. O IP pertence ao Pod e pode mudar quando o Pod é recriado, por isso o Service é utilizado para encontrar a aplicação mesmo quando o Pod é substituído

#### 5. Expor a API e provar a persistência
**Reflexão:** quantos componentes tiveram que funcionar em conjunto para esse dado sobreviver? (PVC, Deployment, Service, Secret, a API...) O que isso mostra sobre como o Kubernetes coordena as peças?

> Para o dado sobreviver à exclusão do Pod do PostgreSQL, vários componentes precisam funcionar em conjunto. O Deployment recria o Pod, o PVC mantém os dados, o Service permite que a API encontre o banco e o Secret fornece as credenciais de conexão. Isso mostra como o Kubernetes coordena diferentes recursos para manter a aplicação funcionando mesmo quando um Pod é removido.

#### 6. Health Checks e escala
**Reflexão:** qual a diferença prática entre liveness e readiness? Por que escalar a API para várias réplicas é seguro, mas escalar o banco desse jeito (com o mesmo PVC) não seria?

> A readiness probe indica se o Pod está pronto para receber tráfego, enquanto a liveness probe verifica se a aplicação continua funcionando e pode provocar a reinicialização do container em caso de falhas persistentes. 
> 
> A API pode ter várias réplicas porque seus Pods são stateless e podem acessar o mesmo banco de dados. Já o PostgreSQL é stateful e precisa manter seus dados de forma consistente em um armazenamento persistente, portanto simplesmente criar várias réplicas usando o mesmo PVC não é uma estratégia adequada.
