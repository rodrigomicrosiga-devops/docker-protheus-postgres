# 🐳 TOTVS Protheus - Cluster PostgreSQL Automatizado

Este repositório é um componente isolado da arquitetura TOTVS Protheus Modern DevOps [https://github.com/rodrigomicrosiga-devops/totvs-protheus-modern-devops], e isola a camada de persistência de dados do ecossistema Protheus dentro da organização **`rodrigomicrosiga-devops`**. Ele provisiona um cluster **PostgreSQL 16** totalmente automatizado, configurado nativamente com as regras de *locale* e *encoding* restritas exigidas pelo dicionário de dados da TOTVS.

---

## 🏗️ Arquitetura de Inicialização (Bootstrap)

A stack utiliza o ciclo de vida nativo do container do PostgreSQL para realizar o provisionamento da base de dados e do usuário proprietário no primeiro subset de inicialização, sem a necessidade de intervenção humana ou scripts manuais pós-deploy.

```mermaid
graph TD
    %% Fluxo de Inicialização
    A[docker compose up] -->|1. Carrega Variáveis| B(.env.postgres)
    B -->|2. Instancia Container| C[Engine: postgres:16]
    C -->|3. Mapeia Volume Local| D[(Volume: postgres_data)]
    
    %% Decisão de Primeira Execução
    C -->|4. Verifica Diretório de Dados| E{Banco já Inicializado?}
    E -->|Sim| F[Apenas Inicia o Serviço]
    E -->|Não| G[Executa Scripts em /docker-entrypoint-initdb.d/]
    
    %% Bootstrap do Script
    G -->|5. Roda Script| H[init-protheus.sh]
    H -->|Criação Primária| I[User: totvs]
    H -->|Encoding: WIN1252 / LC_COLLATE: C| J[Database: protheus_dev]
    J -->|Privilégios Totais| K[GRANT ALL PRIVILEGES]
    
    %% Conclusão
    F --> L[Cluster Pronto para Conexões do DbAccess]
    K --> L

    %% Estilização
    style C fill:#bbf,stroke:#333,stroke-width:2px
    style H fill:#f9f,stroke:#333,stroke-width:2px
    style J fill:#bfb,stroke:#333,stroke-width:2px
```

### ⚙️ Especificações Técnicas e Governança de Dados

Para garantir a integridade relacional do ERP, a inicialização força parâmetros específicos que evitam quebras de índices e erros de agrupamento de caracteres (collation) no DbAccess:

* **Encoding da Base**: `WIN1252`
* **LC_COLLATE**: `C`
* **LC_CTYPE**: `pt_BR.CP1252` (Tratamento nativo de caracteres e acentuações)
* **Persistência Volumétrica**: Nomeada via volume externo (`protheus_postgres_volume`) para blindar os dados contra deleções acidentais de containers.

### 🚀 Como Executar o Cluster

Como estamos utilizando arquivos de ambiente específicos para segregação de escopos, o deploy deve apontar explicitamente para o arquivo de credenciais do Postgres:

```bash
# Inicialização da stack isolada de banco de dados
docker compose --env-file .env.postgres up -d
```