# Guia de Implantação do Rook/Ceph e Estrutura do Projeto

> **Objetivo:** documentar a implantação do Rook/Ceph em Kubernetes e apresentar a organização dos manifestos utilizados na infraestrutura, no armazenamento distribuído e nos cenários de teste do Trabalho de Conclusão de Curso (TCC).

---

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Implantação do Rook Operator](#3-implantação-do-rook-operator)
   1. [Clonar o repositório](#31-clonar-o-repositório)
   2. [Acessar o diretório](#32-acessar-o-diretório)
   3. [Selecionar a versão](#33-selecionar-a-versão)
   4. [Acessar os exemplos](#34-acessar-os-exemplos)
   5. [Aplicar o common.yaml](#35-aplicar-o-commonyaml)
   6. [Aplicar o crds.yaml](#36-aplicar-o-crdsyaml)
   7. [Aplicar o operator.yaml](#37-aplicar-o-operatoryaml)
   8. [Aplicar o csi-operator.yaml](#38-aplicar-o-csi-operatoryaml)
   9. [Aplicar o cluster-rook.yaml](#39-aplicar-o-cluster-rookyaml)
   10. [Aplicar o toolbox.yaml](#310-aplicar-o-toolboxyaml)
4. [Estrutura do projeto](#4-estrutura-do-projeto)
   1. [Infraestrutura e automação](#41-infraestrutura-e-automação)
   2. [Armazenamento em bloco — RBD](#42-armazenamento-em-bloco--rbd)
   3. [Armazenamento de objetos — RGW](#43-armazenamento-de-objetos--rgw)
   4. [Sistema de arquivos — CephFS](#44-sistema-de-arquivos--cephfs)
   5. [Cargas de trabalho para testes](#45-cargas-de-trabalho-para-testes)

---

# 1. Visão geral

Este repositório contém os manifestos e as configurações de infraestrutura
utilizados para a avaliação de **armazenamento distribuído** e **alta
disponibilidade** com Kubernetes, Rook/Ceph e Vagrant.

A documentação apresentada neste arquivo organiza as instruções de
implantação do Rook Operator e descreve a finalidade dos arquivos utilizados
na infraestrutura e nos cenários de teste do TCC.

---

# 2. Pré-requisitos

Antes de iniciar a implantação, é necessário possuir um ambiente Kubernetes
funcional e as ferramentas necessárias para executar os comandos descritos
neste documento.

Entre os principais requisitos estão:

- Git;
- Kubernetes;
- `kubectl`;
- acesso administrativo ao cluster Kubernetes;
- infraestrutura virtualizada configurada conforme o `Vagrantfile`, quando aplicável.

---

# 3. Implantação do Rook Operator

A implantação utiliza a versão **v1.18.2 do Rook**, conforme definida para o
ambiente de avaliação.

## 3.1 Clonar o repositório

Clone o repositório oficial do Rook:

```bash
git clone https://github.com/rook/rook.git
```

> **Observação:** o Git precisa estar instalado no sistema operacional para
> executar o comando acima.

---

## 3.2 Acessar o diretório

Após o clone, entre no diretório criado:

```bash
cd rook
```

---

## 3.3 Selecionar a versão

Faça o checkout da versão utilizada no projeto:

```bash
git checkout v1.18.2
```

> **Nota:** o projeto considera a versão `v1.18.2` como referência. Versões
> `v1.18.5` e `v1.18.6` também foram consideradas no período de desenvolvimento.

---

## 3.4 Acessar os exemplos

A partir do diretório `/rook`, acesse a pasta de exemplos:

```bash
cd deploy/examples
```

A partir deste ponto, os manifestos necessários para a implantação do
Operator estarão disponíveis nesse diretório.

---

## 3.5 Aplicar o `common.yaml`

O arquivo `common.yaml` contém os recursos básicos necessários para a
execução do Rook.

Entre suas funções estão:

- criação do Namespace `rook-ceph`;
- definição de permissões RBAC;
- criação de `ClusterRoles`;
- criação de `ServiceAccounts`;
- configuração das permissões utilizadas pelo Rook Operator.

Execute:

```bash
kubectl apply -f common.yaml
```

---

## 3.6 Aplicar o `crds.yaml`

O arquivo `crds.yaml` instala as **Custom Resource Definitions (CRDs)** do
Rook/Ceph.

As CRDs permitem que o Kubernetes reconheça recursos específicos do Rook,
como:

- `CephCluster`;
- `CephFilesystem`;
- `CephPool`;
- entre outros recursos relacionados ao Ceph.

Execute:

```bash
kubectl apply -f crds.yaml
```

> **Importante:** as CRDs devem estar disponíveis antes da aplicação dos
> manifestos que utilizam esses recursos personalizados.

---

## 3.7 Aplicar o `operator.yaml`

O `operator.yaml` realiza a implantação do **Rook Operator**.

O Operator atua como controlador do ambiente, monitorando os recursos
definidos no Kubernetes e realizando a configuração e manutenção dos
componentes do Ceph, como:

- Monitors (MONs);
- Object Storage Daemons (OSDs);
- Managers (MGRs);
- demais componentes necessários ao funcionamento do cluster.

Aplique o manifesto:

```bash
kubectl apply -f operator.yaml
```

### Verificação do Operator

Após a implantação, verifique o estado dos Pods:

```bash
kubectl -n rook-ceph get pod
```

O Operator deverá apresentar estado operacional adequado, como `Running`.

### Ajuste da imagem do Operator

Caso o Operator não entre em execução após alguns minutos, verifique a
referência da imagem utilizada no `operator.yaml`.

Abra o arquivo:

```bash
sudo nano operator.yaml
```

No editor, utilize:

```text
Ctrl + W
```

e pesquise por:

```text
image: docker.io
```

Quando aplicável ao ambiente, substitua o registro `docker.io` por
`quay.io`, mantendo o restante da referência da imagem.

Depois, reaplique o manifesto:

```bash
kubectl apply -f operator.yaml
```

Se necessário, remova os Pods antigos do Operator antes de realizar uma nova
implantação e verifique novamente:

```bash
kubectl -n rook-ceph get pod
```

> **Atenção:** a alteração do registro da imagem deve ser feita somente quando
> necessária para o ambiente utilizado e de acordo com a versão do Rook
> adotada no projeto.

---

## 3.8 Aplicar o `csi-operator.yaml`

O CSI Operator é necessário para os componentes relacionados ao
provisionamento e gerenciamento de armazenamento no Kubernetes.

Execute:

```bash
kubectl apply -f csi-operator.yaml
```

---

## 3.9 Aplicar o `cluster-rook.yaml`

O manifesto `cluster-rook.yaml` define o recurso `CephCluster`, responsável
pela configuração do cluster Ceph gerenciado pelo Rook.

Execute:

```bash
kubectl apply -f cluster-rook.yaml
```

Em seguida, verifique o estado do cluster:

```bash
kubectl -n rook-ceph get cephcluster
```

---

## 3.10 Aplicar o `toolbox.yaml`

O Toolbox disponibiliza um Pod utilitário para administração e diagnóstico
do ambiente Ceph.

Ele permite executar comandos de gerenciamento e inspeção diretamente no
cluster.

Execute:

```bash
kubectl apply -f toolbox.yaml
```

---

# 4. Estrutura do Projeto

A organização dos arquivos no repositório representa a **estrutura lógica e
física do projeto**. Essa organização não corresponde, necessariamente, à
ordem em que os manifestos devem ser aplicados no cluster Kubernetes.

A estrutura atual do repositório é:

```text
tcc-infra-kubernetes-ceph/
├── armazenamento/
│   ├── aplicacoes-teste/
│   │   ├── pods-cephfs/
│   │   │   ├── cephfs-deployment.yaml
│   │   │   ├── cephfs-pod1.yaml
│   │   │   └── cephfs-pod2.yaml
│   │   └── postgresql/
│   │       ├── namespace-banco.yaml
│   │       ├── postgres-deployment.yaml
│   │       ├── postgres-pvc.yaml
│   │       └── postgres-service.yaml
│   │
│   ├── cephfs-file/
│   │   ├── cephfs-pvc.yaml
│   │   ├── cephfs-rwx-pvc.yaml
│   │   ├── cephfs-storageclass.yaml
│   │   └── cephfs.yaml
│   │
│   ├── rbd-block/
│   │   ├── pool-ec-metadata.yaml
│   │   ├── pool-ec.yaml
│   │   ├── rbd-ec-pvc.yaml
│   │   ├── rook-ceph-block-rbd.yaml
│   │   └── storageclass-ec-rbd.yaml
│   │
│   └── rgw-object/
│       ├── ceph-rgw-service.yaml
│       ├── ceph-rgw.yaml
│       └── ceph-user-rgw.yaml
│
├── rook-ceph-base/
│   └── rook-ceph.yaml
│
├── vagrant/
│   └── Vagrantfile
│
└── README.md
```

## 4.1 `armazenamento/`

Diretório que concentra os recursos relacionados aos diferentes modelos de
armazenamento avaliados no projeto, além das aplicações utilizadas nos
cenários de teste.

### 4.1.1 `armazenamento/cephfs-file/`

Contém os manifestos relacionados ao sistema de arquivos distribuído
**CephFS**.

- `cephfs.yaml`: criação/configuração do sistema de arquivos CephFS;
- `cephfs-storageclass.yaml`: definição da `StorageClass` do CephFS;
- `cephfs-pvc.yaml`: PVC para utilização do CephFS;
- `cephfs-rwx-pvc.yaml`: PVC com modo de acesso `ReadWriteMany (RWX)`.

### 4.1.2 `armazenamento/rbd-block/`

Contém os manifestos relacionados ao armazenamento em bloco **Ceph RBD**.

- `pool-ec-metadata.yaml`: pool de metadados do armazenamento com EC;
- `pool-ec.yaml`: pool de dados utilizando Erasure Coding;
- `rook-ceph-block-rbd.yaml`: configuração do armazenamento em bloco;
- `storageclass-ec-rbd.yaml`: `StorageClass` para provisionamento de RBD;
- `rbd-ec-pvc.yaml`: PVC utilizado para solicitar um volume RBD.

### 4.1.3 `armazenamento/rgw-object/`

Contém os manifestos relacionados ao armazenamento de objetos por meio do
**Ceph Object Gateway (RGW)**.

- `ceph-rgw.yaml`: criação/configuração do RGW;
- `ceph-user-rgw.yaml`: criação do usuário e das credenciais de acesso;
- `ceph-rgw-service.yaml`: criação do `Service` para acesso ao RGW.

### 4.1.4 `armazenamento/aplicacoes-teste/`

Contém as cargas de trabalho utilizadas para validar o comportamento dos
diferentes tipos de armazenamento.

#### `armazenamento/aplicacoes-teste/pods-cephfs/`

- `cephfs-pod1.yaml`: primeiro Pod do cenário de acesso ao CephFS;
- `cephfs-pod2.yaml`: segundo Pod do cenário de acesso compartilhado;
- `cephfs-deployment.yaml`: aplicação utilizada nos testes de persistência e
  resiliência do CephFS.

#### `armazenamento/aplicacoes-teste/postgresql/`

- `namespace-banco.yaml`: Namespace dedicado ao banco de dados;
- `postgres-pvc.yaml`: PVC utilizado pelo PostgreSQL;
- `postgres-deployment.yaml`: Deployment do PostgreSQL;
- `postgres-service.yaml`: Service do PostgreSQL na porta `5432`.

---

## 4.2 `rook-ceph-base/`

Contém os arquivos relacionados à configuração complementar do ambiente
base do Rook/Ceph.

### `rook-ceph.yaml`

Manifesto complementar utilizado para configurações específicas ou
*overrides* do cluster Rook/Ceph.

---

## 4.3 `vagrant/`

Contém os arquivos de automação da infraestrutura virtualizada.

### `Vagrantfile`

Responsável pela criação e configuração das máquinas virtuais utilizadas
no ambiente de testes, incluindo os nós Kubernetes, recursos de CPU e
memória, endereçamento IP e discos adicionais destinados ao Ceph.

---

# 5. Ordem Cronológica de Aplicação dos Recursos

> **Importante:** a estrutura de diretórios apresentada na seção anterior
> representa a organização do repositório. A sequência desta seção representa
> a **ordem lógica de implantação no cluster**, definida pelas dependências
> entre os recursos.

A partir do momento em que o cluster Rook/Ceph já foi implantado e o
`toolbox.yaml` já foi aplicado, a configuração dos serviços de armazenamento
deve seguir uma sequência cronológica.

## 5.1 Visão geral da sequência

A sequência pode ser representada da seguinte forma:

```text
Rook/Ceph já implantado
        │
        ▼
┌─────────────────────────┐
│ 1. Armazenamento em     │
│    bloco (RBD)          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 2. Sistema de arquivos  │
│    CephFS                │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 3. Armazenamento de     │
│    objetos (RGW)        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 4. PostgreSQL           │
│    (utilizando RBD)     │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 5. Aplicações de teste  │
│    do CephFS             │
└─────────────────────────┘
```

> A ordem abaixo considera as dependências dos recursos e a finalidade dos
> cenários de teste. Dentro de cada tecnologia, os recursos base são criados
> antes dos recursos que dependem deles.

---

# 6. Sequência de Implantação do RBD

O RBD é configurado antes da aplicação que utilizará o volume persistente,
pois a `StorageClass` e o PVC do banco dependem da infraestrutura de bloco
estar disponível.

## 6.1 Criar a pool de metadados

Aplicar:

```bash
kubectl apply -f armazenamento/rbd-block/pool-ec-metadata.yaml
```

Esse recurso cria a pool utilizada para armazenar os metadados necessários
ao funcionamento do esquema de Erasure Coding.

## 6.2 Criar a pool de dados

Aplicar:

```bash
kubectl apply -f armazenamento/rbd-block/pool-ec.yaml
```

Essa pool será utilizada como base para os dados do armazenamento em bloco.

## 6.3 Configurar o recurso RBD

Aplicar:

```bash
kubectl apply -f armazenamento/rbd-block/rook-ceph-block-rbd.yaml
```

Esse manifesto estabelece a configuração do armazenamento em bloco
gerenciado pelo Rook/Ceph.

## 6.4 Criar a StorageClass do RBD

Aplicar:

```bash
kubectl apply -f armazenamento/rbd-block/storageclass-ec-rbd.yaml
```

A `StorageClass` disponibiliza o mecanismo de provisionamento dinâmico que
será utilizado pelos PVCs RBD.

## 6.5 Criar o PVC de teste do RBD

Aplicar:

```bash
kubectl apply -f armazenamento/rbd-block/rbd-ec-pvc.yaml
```

Esse PVC permite validar o provisionamento de um volume persistente baseado
em RBD.

---

# 7. Sequência de Implantação do CephFS

Depois da configuração do armazenamento necessário, pode ser criado o
sistema de arquivos CephFS e seus respectivos volumes persistentes.

## 7.1 Criar o CephFS

Aplicar:

```bash
kubectl apply -f armazenamento/cephfs-file/cephfs.yaml
```

Esse manifesto cria e configura o sistema de arquivos distribuído CephFS.

## 7.2 Criar a StorageClass do CephFS

Aplicar:

```bash
kubectl apply -f armazenamento/cephfs-file/cephfs-storageclass.yaml
```

A `StorageClass` permite o provisionamento dinâmico de volumes baseados em
CephFS.

## 7.3 Criar o PVC padrão do CephFS

Aplicar:

```bash
kubectl apply -f armazenamento/cephfs-file/cephfs-pvc.yaml
```

Esse recurso solicita um volume persistente utilizando a `StorageClass`
do CephFS.

## 7.4 Criar o PVC com acesso RWX

Aplicar:

```bash
kubectl apply -f armazenamento/cephfs-file/cephfs-rwx-pvc.yaml
```

Esse PVC utiliza o modo `ReadWriteMany (RWX)` e será utilizado no cenário
de acesso compartilhado entre múltiplos Pods.

---

# 8. Sequência de Implantação do RGW

O RGW disponibiliza armazenamento de objetos compatível com a API S3.

## 8.1 Criar o Ceph Object Gateway

Aplicar:

```bash
kubectl apply -f armazenamento/rgw-object/ceph-rgw.yaml
```

Esse manifesto cria o serviço do Ceph Object Gateway.

## 8.2 Criar o usuário do RGW

Aplicar:

```bash
kubectl apply -f armazenamento/rgw-object/ceph-user-rgw.yaml
```

Esse recurso cria o usuário e as credenciais necessárias para autenticação
nas operações realizadas por meio da API S3.

## 8.3 Expor o RGW por meio de um Service

Aplicar:

```bash
kubectl apply -f armazenamento/rgw-object/ceph-rgw-service.yaml
```

Esse `Service` disponibiliza o endpoint do RGW para acesso pelas aplicações.

---

# 9. Sequência de Implantação do PostgreSQL

O PostgreSQL depende do armazenamento persistente RBD. Por isso, os recursos
do banco devem ser aplicados somente depois que a infraestrutura RBD e sua
`StorageClass` estiverem disponíveis.

## 9.1 Criar o Namespace do banco

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/postgresql/namespace-banco.yaml
```

O Namespace isola os recursos utilizados no cenário do PostgreSQL.

## 9.2 Criar o PVC do PostgreSQL

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/postgresql/postgres-pvc.yaml
```

O PVC solicita o volume persistente que será utilizado pelo banco.

> Esse recurso depende da `StorageClass` RBD já estar configurada e
> disponível.

## 9.3 Criar o Deployment do PostgreSQL

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/postgresql/postgres-deployment.yaml
```

O Deployment cria a instância do PostgreSQL e monta o volume persistente
solicitado pelo PVC.

## 9.4 Criar o Service do PostgreSQL

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/postgresql/postgres-service.yaml
```

O Service disponibiliza o PostgreSQL dentro do cluster na porta:

```text
5432
```

---

# 10. Sequência de Implantação das Aplicações de Teste do CephFS

Com o CephFS e seus PVCs disponíveis, podem ser iniciados os cenários de
teste de acesso compartilhado e resiliência.

## 10.1 Criar o primeiro Pod

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/pods-cephfs/cephfs-pod1.yaml
```

## 10.2 Criar o segundo Pod

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/pods-cephfs/cephfs-pod2.yaml
```

A execução dos dois Pods permite validar o acesso compartilhado ao sistema
de arquivos e a criação concorrente de arquivos.

## 10.3 Criar o Deployment de teste

Aplicar:

```bash
kubectl apply -f armazenamento/aplicacoes-teste/pods-cephfs/cephfs-deployment.yaml
```

Esse Deployment é utilizado para os testes de persistência, resiliência e
reanexação dos volumes do CephFS durante cenários de falha.

---

# 11. Sequência Completa de Aplicação dos Arquivos

Considerando que a implantação inicial do Rook/Ceph já tenha sido concluída
até o `toolbox.yaml`, a sequência consolidada dos manifests do projeto é:

```text
01. armazenamento/rbd-block/pool-ec-metadata.yaml
02. armazenamento/rbd-block/pool-ec.yaml
03. armazenamento/rbd-block/rook-ceph-block-rbd.yaml
04. armazenamento/rbd-block/storageclass-ec-rbd.yaml
05. armazenamento/rbd-block/rbd-ec-pvc.yaml

06. armazenamento/cephfs-file/cephfs.yaml
07. armazenamento/cephfs-file/cephfs-storageclass.yaml
08. armazenamento/cephfs-file/cephfs-pvc.yaml
09. armazenamento/cephfs-file/cephfs-rwx-pvc.yaml

10. armazenamento/rgw-object/ceph-rgw.yaml
11. armazenamento/rgw-object/ceph-user-rgw.yaml
12. armazenamento/rgw-object/ceph-rgw-service.yaml

13. armazenamento/aplicacoes-teste/postgresql/namespace-banco.yaml
14. armazenamento/aplicacoes-teste/postgresql/postgres-pvc.yaml
15. armazenamento/aplicacoes-teste/postgresql/postgres-deployment.yaml
16. armazenamento/aplicacoes-teste/postgresql/postgres-service.yaml

17. armazenamento/aplicacoes-teste/pods-cephfs/cephfs-pod1.yaml
18. armazenamento/aplicacoes-teste/pods-cephfs/cephfs-pod2.yaml
19. armazenamento/aplicacoes-teste/pods-cephfs/cephfs-deployment.yaml
```

> **Observação:** a ordem apresentada separa claramente a configuração da
> infraestrutura de armazenamento dos recursos que a consomem. Um PVC, por
> exemplo, não deve ser tratado como um recurso independente da
> `StorageClass`/infraestrutura que o provisionará.

---

# 12. Relação entre Estrutura do Repositório e Ordem de Execução

É importante distinguir os dois conceitos utilizados neste projeto:

### Estrutura do repositório

Representa **onde cada arquivo está armazenado e como o projeto está
organizado**:

```text
armazenamento/
├── cephfs-file/
├── rbd-block/
├── rgw-object/
└── aplicacoes-teste/
```

### Ordem de execução

Representa **quando cada recurso deve ser aplicado no cluster**, considerando
suas dependências:

```text
Infraestrutura de armazenamento
        ↓
StorageClasses
        ↓
PVCs
        ↓
Aplicações
        ↓
Serviços e cenários de teste
```

Assim, **a ordem apresentada no README não precisa ser igual à ordem
alfabética nem à ordem visual dos arquivos no explorador do projeto**. O
README deve preservar a estrutura física real do repositório e, em uma seção
separada, documentar a sequência cronológica necessária para a implantação.

---

# 13. Considerações Finais

A organização adotada no projeto foi pensada para facilitar a manutenção e
a identificação dos recursos por tecnologia. Já a sequência de implantação
foi definida com base nas relações de dependência entre os componentes.

Dessa forma, o README apresenta duas visões complementares:

1. **estrutura do projeto**, mostrando a localização dos arquivos;
2. **sequência de implantação**, mostrando a ordem em que os recursos devem
   ser aplicados no cluster.

Essa separação evita que a organização de diretórios seja confundida com a
ordem operacional de implantação dos recursos do Ceph e das aplicações de
teste.