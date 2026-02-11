# 🗄️ Backup Distribuído PostgreSQL com pgBackRest e MinIO

Este repositório documenta a instalação, configuração e validação de uma arquitetura de backup distribuído para bancos de dados PostgreSQL, utilizando o **pgBackRest** integrado a armazenamento em objetos compatível com S3 por meio do **MinIO**.

O projeto foi desenvolvido como estudo de caso em ambiente controlado, representativo de cenário real, com o objetivo de servir como base técnica para futura implantação em produção.

---

## 📌 Visão Geral da Arquitetura

Fluxo geral da solução:

PostgreSQL (Dados + WAL)
|
| pgBackRest
| (Full / Incremental / WAL)
v
MinIO (S3 Object Storage)
|
| Restore
v
Nova VM PostgreSQL

A arquitetura permite:

- Backups completos e incrementais
- Arquivamento contínuo de WAL
- Armazenamento centralizado
- Restauração em ambiente isolado (simulação de desastre)

---

## 🧱 Tecnologias Utilizadas

- PostgreSQL 17
- pgBackRest
- MinIO
- Debian GNU/Linux 13
- pgbench (geração de dados fictícios)

---

# 🛠️ Instalação do MinIO

## Pré-requisitos

- Debian 13
- Acesso root ou sudo
- Porta 9000 (API) e 9001 (console) liberadas

## Download e Instalação

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

sudo useradd -r minio-user -s /sbin/nologin
sudo mkdir -p /data/minio
sudo chown -R minio-user:minio-user /data/minio

export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin123

sudo -u minio-user minio server /data/minio --console-address ":9001"

👨‍💻 Autor

Wellygton Matheus Sena de Macedo
Tecnologia em Redes de Computadores – IFRN
matheussenna631@gmail.com
