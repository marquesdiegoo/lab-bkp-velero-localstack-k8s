
## 🧪 **Lab: Backup de Cluster Kubernetes Local com Velero e LocalStack S3**

Vamos configurar um ambiente local para realizar backups do seu cluster Kubernetes usando o Velero, com o LocalStack executando na sua máquina local para simular o serviço S3 da AWS. Este processo é ideal para testes e desenvolvimento.

---

## 🧰 Pré-requisitos

1. **Docker** – Plataforma para criação e gerenciamento de containers.
   🔗 [Instalação do Docker](https://docs.docker.com/engine/install/)

2. **Kind** – Ferramenta para criar clusters Kubernetes locais utilizando containers Docker.
   🔗 [Instalação do Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

3. **kubectl** – Ferramenta de linha de comando para interagir com clusters Kubernetes.
   🔗 [Instalação do kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)([Google Cloud][1])

4. **Velero** – Ferramenta para backup e restauração de clusters Kubernetes.
   🔗 [Instalação do Velero](https://velero.io/docs/main/basic-install/)

5. **LocalStack** – Simulador local de serviços da AWS, incluindo o S3.
   🔗 [Instalação do LocalStack](https://docs.localstack.cloud/getting-started/installation/)

6. **aws-cli** – Interação com a api AWS localstack, incluindo o S3.
   🔗 [Instalação do aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

---

## 🧱 Arquitetura do Lab

```text
[Kubernetes local]
  ├─ LocalStack (S3 simulado)
  ├─ Velero
  └─ Backup e restore cluster/testes consumindo S3 
```

---

## 🔧 Etapas

### 1. Criar o Cluster Kubernetes com Kind

```bash
kind create cluster --name velero-lab 
```

---

### 2. Executar o LocalStack na Máquina Local

Inicie o LocalStack:

```bash
localstack start
```

Isso iniciará o LocalStack com o serviço S3 disponível na url http://localhost:4566 do seu host local.

#### Atenção:  

#### Entre em outro terminal para continuar

Criar o aquivo de credencia e config:

```bash
aws configure
AWS Access Key ID [None]: fake 
AWS Secret Access Key [None]: fake
Default region name [None]: us-east-1
Default output format [None]: json
```

---

### 3. Criar um Bucket S3 no LocalStack

Utilize o AWS CLI para criar um bucket S3 no LocalStack:

```bash
aws s3 mb s3://bucket-backup --endpoint-url=http://localhost:4566
```

Certifique-se de que o AWS CLI esteja configurado com as credenciais padrão (`AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY`).

Verificar o Bucket criado: 
```bash
aws s3 ls s3://bucket-backup --recursive --human-readable --summarize --endpoint-url=http://localhost:4566
```

---

### 4. Instalar o Velero

Baixe e instale o Velero:

```bash
curl -L https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz | tar -xz
sudo mv velero-v1.14.0-linux-amd64/velero /usr/local/bin/
```

---

### 5. Configurar o Velero com o LocalStack

Crie um arquivo de credenciais chamado `credentials-velero` com o seguinte conteúdo:

```ini
[default]
aws_access_key_id = fake
aws_secret_access_key = fake
```

Instale o Velero no cluster:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket bucket-backup \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=us-east-1,s3ForcePathStyle=true,s3Url=http://host.docker.internal:4566
```

Aqui, `host.docker.internal` é usado para que o Velero dentro do cluster Kind possa acessar o LocalStack na máquina host.

---


### 6. Criar o POD para restaurar o backup 

Comando para criar o pod para o restore:

```bash
kubectl run nginx --image nginx:alpine
```

```bash
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15s
```

### 7. Realizar um Backup com o Velero

Crie um backup dos recursos do cluster:

```bash
velero backup create backup-teste --include-namespaces default
```

Verifique o status do backup:

```bash
velero backup get
```

```ini
velero backup get
NAME           STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
backup-teste   Completed   0        0          2025-06-01 14:15:27 -0300 -03   29d       default            <none>
```

Verificando o Bucket com o backup:

```ini
aws s3 ls s3://bucket-backup --recursive --human-readable --summarize --endpoint-url=http://localhost:4566
2025-06-01 14:15:28   29 Bytes backups/backup-teste/backup-teste-csi-volumesnapshotclasses.json.gz
2025-06-01 14:15:28   29 Bytes backups/backup-teste/backup-teste-csi-volumesnapshotcontents.json.gz
2025-06-01 14:15:28   29 Bytes backups/backup-teste/backup-teste-csi-volumesnapshots.json.gz
2025-06-01 14:15:28   27 Bytes backups/backup-teste/backup-teste-itemoperations.json.gz
2025-06-01 14:15:28    3.2 KiB backups/backup-teste/backup-teste-logs.gz
2025-06-01 14:15:28   29 Bytes backups/backup-teste/backup-teste-podvolumebackups.json.gz
2025-06-01 14:15:28  152 Bytes backups/backup-teste/backup-teste-resource-list.json.gz
2025-06-01 14:15:28   49 Bytes backups/backup-teste/backup-teste-results.gz
2025-06-01 14:15:28   27 Bytes backups/backup-teste/backup-teste-volumeinfo.json.gz
2025-06-01 14:15:28   29 Bytes backups/backup-teste/backup-teste-volumesnapshots.json.gz
2025-06-01 14:15:28    2.5 KiB backups/backup-teste/backup-teste.tar.gz
2025-06-01 14:15:28    2.9 KiB backups/backup-teste/velero-backup.json

Total Objects: 12
   Total Size: 9.0 KiB
```

---


### 8. Deletando o POD para o restore


Comando para deletar o POD para o restore em seguida. 

```bash
kubectl delete pod nginx
```

```bash
kubectl get pods
No resources found in default namespace.
```

### 9. Restaurar um Backup com o Velero

Para restaurar o backup:

```bash
velero restore create --from-backup backup-teste
```

Verifique o status da restauração:

```bash
velero restore get
NAME                          BACKUP         STATUS      STARTED                         COMPLETED                       ERRORS   WARNINGS   CREATED                         SELECTOR
backup-teste-20250601143023   backup-teste   Completed   2025-06-01 14:30:23 -0300 -03   2025-06-01 14:30:24 -0300 -03   0        1          2025-06-01 14:30:23 -0300 -03   <none>
```

```bash
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          26s
```

---

## 🧼 Cleanup

```bash
kind delete cluster --name velero-lab
```
```bash
localstack stop
```

---

## 🔄 Considerações

* Certifique-se de que o Velero e o LocalStack estejam na mesma rede ou que o serviço do LocalStack seja acessível pelo Velero.


## 📝 Conclusão

Ao integrar ferramentas como Kind, Velero e LocalStack, é possível criar um ambiente local robusto para simular backups de clusters Kubernetes utilizando o serviço S3 da AWS. Essa abordagem oferece uma solução eficiente para testes e desenvolvimento, permitindo que equipes validem processos de backup e restauração sem depender de recursos em nuvem.
