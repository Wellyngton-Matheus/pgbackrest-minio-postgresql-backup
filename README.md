# 🗄️ Backup Distribuído PostgreSQL com pgBackRest e MinIO (S3)

Este repositório documenta a instalação, configuração e validação de uma **arquitetura de backup distribuído** para bancos **PostgreSQL**, utilizando o **pgBackRest** integrado a armazenamento em objetos compatível com **S3** por meio do **MinIO**.

> **Escopo:** material prático (lab) para estudo de caso e replicação do experimento.  
> **Produção:** exige hardening, segregação de funções, credenciais seguras, TLS e políticas de retenção/testes periódicos.

---

## ✅ Objetivos

- Centralizar backups em object storage (S3/MinIO) com **pgBackRest**
- Executar **backups full e incremental** + **archive de WAL**
- Permitir **restore** em uma nova VM (simulação de desastre)
- Reproduzir um cenário controlado, com dados fictícios (pgbench)

---

## 🧩 Visão Geral da Arquitetura

Fluxo geral:

```text
PostgreSQL (Dados + WAL)
   |
   |  pgBackRest (Full / Incremental / WAL)
   v
MinIO (S3 Object Storage)
   |
   |  Restore
   v
Nova VM PostgreSQL (validação / DR)
```

A arquitetura permite:

- Backups completos e incrementais
- Arquivamento contínuo de WAL (para restore consistente / PITR, se configurado)
- Armazenamento centralizado em object storage
- Restauração em ambiente isolado (simulação de desastre)

---

## 🧱 Tecnologias Utilizadas

- **PostgreSQL 17**
- **pgBackRest**
- **MinIO**
- **Debian GNU/Linux 13**
- **pgbench** (geração de dados fictícios)

---

## 🔐 Avisos Importantes (não ignore)

- **NÃO** comite senhas no repositório. Use variáveis de ambiente, vault ou arquivos fora do versionamento.
- Para produção, recomenda-se:
  - Usuário dedicado (ex.: `minio-user`) e permissões mínimas
  - TLS (API e Console), firewall e rede interna
  - Políticas/ACLs no MinIO (bucket policy/IAM)
  - Rotina de testes de restore (backup “sem restore” não prova nada)
  - Monitoramento (ex.: Zabbix / Prometheus) e alertas

---

# 🛠️ Instalação do MinIO (Debian 13) — Lab

## Pré-requisitos

- Debian 13
- Acesso `root` ou `sudo`
- Portas liberadas:
  - **9000** (API S3)
  - **9001** (Console)

> Em produção, prefira expor apenas o necessário e com TLS.

---

## 1) Instalar o MinIO (pacote `.deb`)

Baixe e instale o pacote (exemplo de versão):

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20241013133411.0.0_amd64.deb
sudo dpkg -i minio_20241013133411.0.0_amd64.deb
sudo systemctl enable minio
```

---

## 2) Preparar disco (XFS) e ponto de montagem

Instale utilitários do XFS:

```bash
sudo apt update
sudo apt install -y xfsprogs
```

Formate o disco (**confirme o device antes**):

```bash
sudo mkfs.xfs /dev/sdb
```

Crie o diretório de montagem:

```bash
sudo mkdir -p /mnt/volume01
```

---

## 3) Montagem persistente via `/etc/fstab`

Obtenha o UUID do disco:

```bash
sudo blkid
```

Edite o `/etc/fstab` e adicione ao final (**substitua pelo UUID real**):

```fstab
UUID=uuid_do_bloco  /mnt/volume01  xfs  defaults,nofail  0  0
```

Recarregue e monte:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Valide:

```bash
df -h | grep volume01
```

---

## 4) (Opcional — somente se necessário) Ajustar usuário do serviço

> **Lab:** pode rodar como `root`.  
> **Produção:** crie usuário dedicado e aplique permissões mínimas.

Arquivo do service:

```bash
sudo nano /etc/systemd/system/multi-user.target.wants/minio.service
```

Exemplo (LAB):

```ini
User=root
Group=root
```

---

## 5) Criar `/etc/default/minio`

Crie/edite o arquivo:

```bash
sudo nano /etc/default/minio
```

Conteúdo (exemplo):

```bash
MINIO_VOLUMES="/mnt/volume01"
MINIO_OPTS="--console-address :9001"

# ⚠️ Troque por credenciais fortes e NÃO versionadas
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="SENHA-AQUI"
```

> Dica: em produção, prefira `EnvironmentFile` protegido e/ou secrets (Vault, Ansible Vault, etc.).

---

## 6) Iniciar e validar o serviço

```bash
sudo systemctl daemon-reload
sudo systemctl restart minio
sudo systemctl status minio --no-pager
```

Acesso:

- **Console:** `http://IP_DO_MINIO:9001`
- **API S3:** `http://IP_DO_MINIO:9000`

---


# 🧰 Instalação do pgBackRest (PostgreSQL 17) — Lab

Esta seção cobre a instalação e configuração do **pgBackRest** no servidor PostgreSQL, usando o **MinIO** como repositório S3.

> **Premissas do exemplo (ajuste para o seu ambiente):**
> - PostgreSQL 17 em Debian 13
> - Datadir: `/var/lib/postgresql/17/main`
> - Porta: `5432`
> - Stanza: `main`
> - MinIO acessível em: `http://IP_DO_MINIO:9000`
> - Bucket: `pgbackrest`

---

## 0) Preparar bucket e credenciais no MinIO

Crie um bucket (ex.: `pgbackrest`) e gere credenciais (Access Key / Secret Key) com permissões no bucket.

Se estiver usando o `mc` (MinIO Client), exemplo:

```bash
mc alias set labminio http://IP_DO_MINIO:9000 MINIO_ROOT_USER MINIO_ROOT_PASSWORD
mc mb labminio/pgbackrest
mc ls labminio
```

> **Produção:** prefira usuário dedicado no MinIO com policy mínima (somente no bucket do backup).

---

## 1) Instalar o pgBackRest

No servidor do PostgreSQL:

```bash
sudo apt update
sudo apt install -y pgbackrest
```

Crie diretórios locais (log/cache) e ajuste permissões:

```bash
sudo mkdir -p /var/log/pgbackrest /var/lib/pgbackrest
sudo chown -R postgres:postgres /var/log/pgbackrest /var/lib/pgbackrest
sudo chmod 750 /var/log/pgbackrest /var/lib/pgbackrest
```

---

## 2) Configurar o pgBackRest (`/etc/pgbackrest/pgbackrest.conf`)

Crie/edite o arquivo:

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

Exemplo de configuração com repositório S3 (MinIO):

```ini
[global]
# Logs
log-level-console=info
log-level-file=info
log-path=/var/log/pgbackrest

# Retenção (exemplo — ajuste conforme sua política)
repo1-retention-full=2
repo1-retention-diff=7

# Repositório S3 (MinIO)
repo1-type=s3
repo1-path=/pgbackrest
repo1-s3-endpoint=IP_DO_MINIO:9000
repo1-s3-region=us-east-1
repo1-s3-bucket=pgbackrest
repo1-s3-key=ACCESS_KEY_AQUI
repo1-s3-key-secret=SECRET_KEY_AQUI
repo1-s3-uri-style=path
repo1-s3-verify-tls=n

# (Opcional) compressão
compress-type=zst
compress-level=3

[main]
pg1-path=/var/lib/postgresql/17/main
pg1-port=5432
```

> **Notas críticas:**
> - `repo1-s3-verify-tls=n` é aceitável em **lab** (HTTP). Em produção use **TLS** e ative a verificação.
> - Evite armazenar `repo1-s3-key-secret` em texto puro; use secrets/ACLs e proteja permissões do arquivo.

Ajuste permissões do arquivo de configuração:

```bash
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
```

---

## 3) Habilitar archive de WAL no PostgreSQL

Edite o `postgresql.conf` do cluster:

```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```

Garanta/adicione (exemplo):

```conf
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_timeout = 60
```

Reinicie o PostgreSQL:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql --no-pager
```

---

## 4) Criar a stanza e validar

Como usuário `postgres`:

```bash
sudo -iu postgres
pgbackrest --stanza=main stanza-create
pgbackrest --stanza=main check
exit
```

Se falhar, valide conectividade com o MinIO e permissões do bucket.

---

## 5) Executar backups (Full / Incremental)

Backup full:

```bash
sudo -iu postgres pgbackrest --stanza=main --type=full backup
```

Backup incremental:

```bash
sudo -iu postgres pgbackrest --stanza=main --type=incr backup
```

Verificar estado do repositório:

```bash
sudo -iu postgres pgbackrest --stanza=main info
```

---

## 6) Restore (visão geral — validação em nova VM)

Em uma VM nova (PostgreSQL parado), o fluxo típico é:

1. Instalar PostgreSQL e pgBackRest
2. Configurar `pgbackrest.conf` com o mesmo repositório S3
3. Parar o serviço do PostgreSQL
4. Limpar o datadir (com cuidado)
5. Executar `restore`
6. Subir o PostgreSQL

Exemplo (ajuste paths e garanta que o PostgreSQL esteja parado):

```bash
sudo systemctl stop postgresql
sudo -iu postgres rm -rf /var/lib/postgresql/17/main/*
sudo -iu postgres pgbackrest --stanza=main restore
sudo systemctl start postgresql
```

> **Atenção:** restore é destrutivo no datadir. Faça isso apenas na VM de validação/DR.

---

## ✅ Verificação rápida (opcional)

Se tiver o cliente `mc` (MinIO Client), você pode validar acesso e criar bucket:

```bash
# Baixe o mc (exemplo)
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure um alias
mc alias set labminio http://IP_DO_MINIO:9000 MINIO_ROOT_USER MINIO_ROOT_PASSWORD

# Crie um bucket (ex.: pgbackrest)
mc mb labminio/pgbackrest
mc ls labminio
```

---

## 📁 Estrutura sugerida do repositório (opcional)

```text
.
├── docs/
│   ├── arquitetura.md
│   └── resultados.md
├── scripts/
│   ├── minio-setup.sh
│   └── pgbackrest-setup.sh
└── README.md
```

---

## 👨‍💻 Autor

**Wellygton Matheus Sena de Macedo**  
Tecnologia em Redes de Computadores – IFRN  
📧 matheussenna631@gmail.com
