# ÓleoApp: backend

API do **ÓleoApp**, um aplicativo que ajuda moradores de **Maricá (RJ)** a encontrar pontos de coleta de óleo lubrificante usado, traçar a rota até eles, registrar descartes e acompanhar o impacto ambiental.

## Stack

- Python 3.12 e FastAPI (uvicorn)
- MongoDB 8 (driver PyMongo assíncrono) e Redis 7 (rate limit e cache)
- httpx (chamadas ao Google, ViaCEP, Nominatim e Brevo)
- pytest e ruff
- Docker e Docker Compose

## Estrutura de pastas

```
app/
  api/            rotas e dependências (autenticação, rate limit)
  core/           configuração, erros, geografia, cache, fatores de impacto
  db/             conexão, coleções e índices
  schemas/        modelos Pydantic de entrada e saída
  repositories/   acesso ao MongoDB
  services/       regras de negócio e integrações externas
scripts/          importação de pontos e candidatos da curadoria
seed/             collection_points.json (pontos verificados)
tests/            testes automatizados
docs/             deploy.md e curadoria-pontos.md
```

## Como rodar localmente

Pré-requisito: Docker com Docker Compose.

1. Copie o arquivo de exemplo: `cp .env.example .env`
2. Abra o `.env` e preencha as senhas. Para gerar cada uma: `openssl rand -hex 24`
3. Suba tudo: `docker compose up --build`
4. Confira:
   - http://localhost:8000/api/health deve responder 200
   - http://localhost:8000/docs mostra a documentação interativa (só existe fora de produção)

O compose sobe três serviços (`api`, `mongo` e `redis`). Localmente, a API e o Mongo ficam acessíveis só em `127.0.0.1` (o Mongo na porta 27017, para o MongoDB Compass). O Redis não é publicado.

O script de criação do usuário do banco só roda quando o volume do Mongo é criado pela primeira vez. Se mudar as senhas do `.env` depois disso, apague o volume (`docker compose down -v`) e suba de novo.

## Testes e lint

```bash
docker compose run --rm api pytest
docker compose run --rm api ruff check .
docker compose run --rm api ruff format --check .
```

O CI (`.github/workflows/backend-ci.yml`) roda os mesmos comandos em todo PR e em todo push na `main`, e ainda confere se a imagem do Render compila (`docker build`).

## Importar pontos de coleta

Os pontos verificados ficam em `seed/collection_points.json`.

- **Local** (primeiro uma simulação, depois gravando):

  ```bash
  docker compose exec api python -m scripts.import_points seed/collection_points.json
  docker compose exec api python -m scripts.import_points seed/collection_points.json --apply
  ```

- **Atlas (produção):** veja o passo "Importação de pontos" em [docs/deploy.md](docs/deploy.md).

Para montar a lista de pontos reais, veja [docs/curadoria-pontos.md](docs/curadoria-pontos.md).

## Variáveis de ambiente

Os valores nunca vão para o git. Os nomes e exemplos locais estão em `.env.example`. As variáveis do compose local (usuários e senhas do Mongo e do Redis) estão explicadas nos comentários desse arquivo.

| Variável | Exemplo | Finalidade |
| --- | --- | --- |
| `PORT` | definido pelo Render (padrão 8000 local) | Porta do uvicorn |
| `ENV` | `development`, `test` ou `production` | Em produção desliga `/docs` e o envio de e-mail pelo console |
| `MONGO_URL` | `mongodb://...` ou `mongodb+srv://...` | Conexão com o MongoDB |
| `MONGO_DB_NAME` | `oleoapp` | Nome do banco |
| `MONGO_MAX_POOL_SIZE` | `10` | Tamanho máximo do pool de conexões |
| `MONGO_TIMEOUT_MS` | `10000` | Tempo para escolher servidor e conectar |
| `REDIS_URL` | `redis://:<senha>@redis:6379/0` | Rate limit e cache. Vazio desliga os dois |
| `SESSION_COOKIE_NAME` | `oleo_session` | Nome do cookie de sessão |
| `SESSION_TTL_DAYS` | `7` | Validade da sessão |
| `COOKIE_SECURE` | `false` local, `true` em produção | Cookie só via HTTPS |
| `FRONTEND_URL` | `http://localhost:3000` | Base dos links de redefinição de senha |
| `GOOGLE_MAPS_API_KEY` | a chave | Routes API e Geocoding no backend |
| `GEOCODER` | `google` ou `nominatim` | Provedor da busca por endereço |
| `NOMINATIM_USER_AGENT` | `OleoApp/1.0 (contato: <e-mail>)` | Exigido pela política do Nominatim |
| `AREA_CENTER_LAT`, `AREA_CENTER_LNG`, `AREA_RADIUS_KM` | `-22.9189`, `-42.8186`, `40` | Área atendida |
| `EMAIL_PROVIDER` | `console` local, `brevo` em produção | O `console` só é aceito fora de produção |
| `BREVO_API_KEY` | a chave | Envio de e-mail |
| `EMAIL_FROM`, `EMAIL_FROM_NAME` | remetente verificado, `ÓleoApp` | Remetente dos e-mails |
| `PASSWORD_RESET_TTL_MIN` | `30` | Validade do link de redefinição de senha |
| `RATE_LIMIT_GLOBAL_PER_MIN`, `RATE_LIMIT_LOGIN_PER_MIN`, `RATE_LIMIT_SIGNUP_PER_HOUR`, `RATE_LIMIT_PASSWORD_FORGOT_PER_HOUR`, `RATE_LIMIT_DIRECTIONS_PER_HOUR`, `RATE_LIMIT_GEOCODE_PER_MIN` | `120`, `5`, `5`, `5`, `30`, `30` | Limites de requisições |

## Convenções

- **Camadas:** a rota (`app/api`) valida a entrada e chama um serviço; o serviço (`app/services`) aplica as regras de negócio; o repositório (`app/repositories`) é o único que fala com o banco.
- **Formato de erro:** toda resposta de erro tem o formato `{"error": {"code", "message", "fields"}}`, com mensagens em português.
- **Branches:** `tarefa/<ID>-<resumo>`, por exemplo `tarefa/BE-06-rotas`.
- **Commits convencionais:** `feat(BE-06): ...`, `fix: ...`, `docs: ...`, `test: ...`.
- **Um PR por tarefa**, com o `ruff` e os testes passando.

## Produção

Produção: Render (Docker, plano gratuito) + MongoDB Atlas M0, ver [docs/deploy.md](docs/deploy.md).

## Riscos aceitos

- **Chave do Google de demonstração, sem restrições configuráveis.** Quem obtiver a chave pode usá-la até o limite da conta. Mitigação: rate limit por usuário nas rotas que a usam e acompanhamento do consumo no painel do Google.
- **O plano gratuito do Render dorme após 15 minutos sem tráfego.** A primeira requisição depois disso leva cerca de 1 minuto. O frontend mostra "Acordando o servidor…".
- **O cluster M0 do Atlas não tem backup automático** e pausa após 30 dias sem conexões. O backup é feito manualmente, fora do projeto (veja [docs/deploy.md](docs/deploy.md)).
