# API DevOps de Tarefas: Pipeline CI/CD com GitHub Actions, Docker, Amazon ECR e EC2

Trabalho da disciplina **Integração DevOps**. O repositório implementa uma esteira completa de integração e entrega contínua para uma API Python: a cada alteração aprovada na branch `main`, o GitHub Actions testa o código, constrói a imagem Docker, publica no Amazon ECR e implanta automaticamente em uma instância Amazon EC2.

## 1. Descrição da aplicação e objetivo

A aplicação é uma API REST de tarefas escrita com **FastAPI**. Os dados ficam em memória, pois o foco do trabalho é o fluxo DevOps e não a persistência.

| Método | Caminho | Objetivo |
|---|---|---|
| GET | `/` | Apresentação da API |
| GET | `/health` | Estado da API e **versão implantada** |
| GET | `/tasks` | Lista as tarefas |
| POST | `/tasks` | Cria uma tarefa |
| PATCH | `/tasks/{id}/done` | Conclui uma tarefa |

**Objetivo do trabalho:** substituir a publicação manual (atualizar código no servidor, construir imagem, executar contêiner) por um processo automatizado, rastreável e reproduzível:

```
Alteração no código → Branch → Pull Request → Testes automatizados → Merge na main
→ Build Docker → Publicação no ECR → Deploy na EC2 → Aplicação atualizada
```

## 2. Tecnologias utilizadas

- **Python 3.13** e **FastAPI** (API) com **Uvicorn**
- **pytest** e **pytest-cov** (testes e cobertura) e **Ruff** (lint e formatação)
- **Docker** (contêiner da aplicação)
- **GitHub Actions** (CI e CD)
- **Amazon ECR** (registro privado de imagens)
- **Amazon EC2** com Amazon Linux 2023 (hospedagem)
- **IAM Role** (permissão da EC2 para ler o ECR) e **GitHub Secrets** (credenciais protegidas)

## 3. Estrutura do repositório

```text
.
├── .github/workflows/
│   ├── ci.yml          # integração contínua (Pull Requests)
│   └── deploy.yml      # build, publicação no ECR e deploy na EC2 (main)
├── app/
│   ├── __init__.py
│   └── main.py         # API FastAPI
├── docs/
│   ├── API.md
│   ├── DOCKER.md
│   ├── GITHUB-ACTIONS.md
│   ├── ROTEIRO-AULA.md
│   └── evidencias/     # capturas de tela das pipelines e do deploy
├── tests/
│   └── test_api.py     # testes automatizados
├── .dockerignore
├── .gitignore
├── Dockerfile
├── compose.yaml
├── pyproject.toml      # configuração de pytest, cobertura e Ruff
├── requirements.txt    # dependências de execução
├── requirements-dev.txt# dependências de desenvolvimento e CI
└── README.md
```

## 4. Execução local

Pré-requisitos: Git e Python 3.13.

**Windows (PowerShell):**

```powershell
py -3.13 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

**Linux e macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

Acesse `http://localhost:8000/health` ou a documentação interativa em `http://localhost:8000/docs`.

**Testes e qualidade** (os mesmos comandos executados pelo CI):

```bash
ruff check .
ruff format --check .
pytest --cov=app --cov-report=term-missing
```

## 5. Construção e execução da imagem Docker

O `Dockerfile` usa `python:3.13-slim`, instala as dependências de `requirements.txt` durante o build, executa a aplicação com um usuário sem privilégios e inicia o Uvicorn na porta **8000**.

```bash
docker build -t devops-python-api:local .
docker run --rm -p 8000:8000 --name api-local devops-python-api:local
```

Em outro terminal:

```bash
curl http://localhost:8000/health
docker ps
```

Resposta esperada: `{"status":"ok","version":"<versão>"}`.

Com Docker Compose: `docker compose up --build`.

## 6. Pipeline de integração contínua (CI)

Arquivo: `.github/workflows/ci.yml`

**Gatilhos:** abertura ou atualização de um **Pull Request** com destino à `main` (e execução manual).

| Job | O que faz |
|---|---|
| `quality-and-tests` | Baixa o código (`actions/checkout`), configura o Python (`actions/setup-python`, versão lida de `.python-version`), instala `requirements-dev.txt`, executa `ruff check`, `ruff format --check` e **`pytest`** com cobertura mínima de 90%. Publica relatórios como artefatos e registra no resumo da execução se o resultado foi sucesso ou falha. |
| `container-smoke-test` | Executado somente se o job anterior passar (`needs`). Constrói a imagem Docker, inicia o contêiner e consulta `/health`. |

Se os testes falham, o job fica vermelho e o Pull Request mostra o bloqueio no painel de *Checks*. Nenhuma etapa de entrega é executada em Pull Requests.

## 7. Pipeline de entrega contínua (CD)

Arquivo: `.github/workflows/deploy.yml`

**Gatilho:** `push` na branch `main`, ou seja, após o merge de um Pull Request aprovado (e execução manual).

Os jobs são encadeados com `needs`, então uma etapa só roda se a anterior foi aprovada:

1. **`test`:** instala as dependências e executa o `pytest`. Se falhar, nada é publicado.
2. **`build-and-push`:**
   - autentica na AWS usando GitHub Secrets;
   - faz login no Amazon ECR (`aws-actions/amazon-ecr-login`);
   - constrói a imagem Docker;
   - identifica a imagem com a tag do commit (`${{ github.sha }}`);
   - envia a imagem ao ECR.
3. **`deploy`:** conecta à EC2 por SSH com chave privada guardada em Secret e executa:
   - autenticação do Docker da instância no ECR (usando a IAM Role);
   - `docker pull` da imagem do commit;
   - remoção do contêiner anterior (`docker rm -f api`);
   - início da nova versão (`docker run -d --restart unless-stopped -p 80:8000`);
   - verificação (`docker ps` e `curl /health`). Se a aplicação não responder, o job falha.

Nenhum acesso manual à EC2 é necessário para publicar uma nova versão.

### Credenciais e segurança

Nenhuma credencial está no código. Elas ficam em **Settings → Secrets and variables → Actions**:

| Secret | Uso |
|---|---|
| `AWS_ACCESS_KEY_ID` | Autenticação na AWS (build e push) |
| `AWS_SECRET_ACCESS_KEY` | Autenticação na AWS |
| `AWS_SESSION_TOKEN` | Token temporário (ambiente AWS Academy) |
| `EC2_HOST` | Endereço da instância |
| `EC2_USER` | Usuário SSH (`ec2-user`) |
| `EC2_SSH_KEY` | Chave privada SSH |

A EC2 usa uma **IAM Role** (`LabRole`) para baixar imagens do ECR, sem credenciais gravadas na instância.

## 8. Arquitetura de implantação

```text
 Desenvolvedor
      │  git push (branch) + Pull Request
      ▼
 ┌──────────────┐   PR: ci.yml (testes)    ┌────────────────────┐
 │   GitHub     │─────────────────────────▶│  GitHub Actions    │
 │ (repositório)│   merge na main:         │  runner (Ubuntu)   │
 └──────────────┘   deploy.yml             └─────────┬──────────┘
                                                     │ 1. testes (pytest)
                                                     │ 2. docker build
                                                     │ 3. docker push (tag = SHA do commit)
                                                     ▼
                                            ┌────────────────────┐
                                            │   Amazon ECR       │
                                            │ (repositório       │
                                            │  privado)          │
                                            └─────────┬──────────┘
                                                      │ docker pull (IAM Role)
        SSH (chave em Secret)                         ▼
 GitHub Actions ─────────────────────────▶ ┌────────────────────────┐
                                           │  Amazon EC2            │
   Usuário ──── HTTP :80 ────────────────▶ │  Docker: contêiner API │
                                           │  (porta 80 → 8000)     │
                                           └────────────────────────┘
```

**Configuração da instância EC2:**
- Amazon Linux 2023, tipo `t3.micro`, região `us-east-1`
- Docker instalado e ativo (`systemctl enable --now docker`)
- IAM Role com permissão de leitura do ECR
- Security Group: SSH (22) para o deploy e HTTP (80) para acesso à API
- Aplicação executada diretamente como contêiner Docker

## 9. URL da API publicada

- Verificação de saúde e versão: **http://174.129.109.2/health**
- Documentação interativa: http://174.129.109.2/docs

> A instância é temporária e será encerrada após a avaliação, conforme as regras da atividade.

## 10. Como verificar a versão implantada

```bash
curl http://174.129.109.2/health
```

Exemplo de resposta: `{"status":"ok","version":"1.1.0"}`.

A versão vem de `app.version` em `app/main.py`. Também é possível confirmar na EC2 (somente consulta) com `docker ps`, que mostra a imagem em execução com a tag do commit, e no ECR, que lista as imagens publicadas e suas tags.

## 11. Fluxo de desenvolvimento e Pull Requests

Todo o desenvolvimento seguiu o fluxo de branches e Pull Requests, sem commits diretos na `main`:

| PR | Branch | Conteúdo |
|---|---|---|
| nº 1 | `feature/ci-cd-ec2` | CI restrito a Pull Requests, resumo de resultado e `.gitignore` |
| nº 2 | `feature/deploy-ecr-ec2` | Pipeline de deploy (ECR + EC2) |
| nº 3 | `feature/atualiza-versao` | Atualização da versão `1.0.0` → `1.1.0` (demonstração do deploy automático) |
| nº 4 | `docs/readme-final` | Documentação e evidências |

Links dos Pull Requests:

- PR nº 1: <COLE_AQUI_O_LINK>
- PR nº 2: <COLE_AQUI_O_LINK>
- PR nº 3: <COLE_AQUI_O_LINK>
- PR nº 4: <COLE_AQUI_O_LINK>

## 12. Evidências da execução das pipelines

### Integração contínua (CI)

Execução do CI com os testes passando em um Pull Request:

![CI verde no PR nº 1](docs/evidencias/01-ci-verde.png)

### Docker local

![Docker local](docs/evidencias/02-docker-local.png)

### Configuração da EC2

![Configuração da EC2](docs/evidencias/03-ec2-config.png)

### Deploy automatizado (versão 1.0.0)

Pipeline de deploy concluída com sucesso:

![Deploy verde](docs/evidencias/04-deploy-verde.png)

Imagem publicada no Amazon ECR com a tag do commit:

![Imagem no ECR](docs/evidencias/05-ecr-tag.png)

Contêiner em execução na instância EC2:

![Contêiner na EC2](docs/evidencias/06-ec2-docker-ps.png)

Aplicação acessível pelo navegador (versão 1.0.0):

![API versão 1.0.0](docs/evidencias/07-api-v1.png)

### Atualização automática (versão 1.1.0)

CI do Pull Request que alterou a versão:

![CI do PR nº 3](docs/evidencias/08-ci-pr3.png)

Deploy disparado automaticamente pelo merge na `main`:

![Deploy da versão 1.1.0](docs/evidencias/09-deploy-v2.png)

Aplicação na EC2 já com a nova versão (1.1.0), sem nenhum comando manual de implantação:

![API versão 1.1.0](docs/evidencias/10-api-v2.png)

## 13. Encerramento dos recursos AWS

Após a avaliação, os recursos foram/serão removidos para evitar cobranças: instância EC2, repositório ECR, Security Group e key pair.