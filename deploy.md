# Deploy: Render + MongoDB Atlas M0

Este documento descreve como colocar a API do ÓleoApp no ar e como operá-la. Os passos marcados com **[PAINEL]** são ações manuais nos painéis do Atlas e do Render.

> Nunca coloque senhas, chaves ou URLs com credenciais neste arquivo, em issues, em commits ou em chats. Guarde-as no cofre de senhas do time. Neste documento, `<servico>`, `<cluster>`, `<regiao>` e `<senha>` são marcadores.

## 1. Visão geral

| Peça | O que é | Pontos de atenção |
| --- | --- | --- |
| Vercel | Frontend. Um rewrite de `/api/*` aponta para o Render | A Vercel espera até 120 s pela resposta da API |
| Render Web Service (gratuito, Docker) | A API | Dorme após 15 min sem tráfego e acorda em cerca de 1 min; disco efêmero; sem shell |
| Render Key Value (gratuito) | Redis para rate limit e cache | Guarda os dados só em memória |
| MongoDB Atlas M0 | Banco de dados | 0,5 GB; até 500 conexões; sem backup automático; pausa após 30 dias sem conexões |

## 2. [PAINEL Atlas] Projeto e cluster

1. Crie um **projeto novo**. Não reutilize o do MVP, que provavelmente está liberado para `0.0.0.0/0`.
2. Crie o cluster **M0** no provedor e na região mais próximos da região escolhida no Render (`<regiao>`).

**Verificação:** o cluster aparece como `M0` e a lista de acesso (IP Access List) do projeto está vazia.

## 3. [PAINEL Atlas] Usuários do banco

Caminho: **Database Access** (ou "Database & Network Access" nas versões novas) → **Database Users** → **Add New Database User** → método **Password** → **Specific Privileges**.

| Usuário | Privilégio | Uso |
| --- | --- | --- |
| `oleoapp_api` | `readWrite` em `oleoapp`, com **Restrict Access to Specific Clusters** marcando só este cluster | A API |
| `oleoapp_maint` | `readWrite` em `oleoapp` | Só importação de pontos, por 1 ou 2 pessoas |
| `oleoapp_backup` | `read` em `oleoapp` | Só backup |

- Gere cada senha com `openssl rand -hex 24` (24 ou mais caracteres) e guarde no cofre do time.
- **Proibido** usar `readWriteAnyDatabase` ou `atlasAdmin`.

**Verificação:** são 3 usuários, e nenhum tem privilégio fora de `oleoapp`.

## 4. [PAINEL Render] Key Value

1. Crie uma instância **Free** do Key Value, na **mesma região** da API.
2. Copie a **Internal URL**. Ela será o `REDIS_URL` da API.

## 5. [PAINEL Render] Web Service da API

Configuração:

- Repositório `OleoAppBackEnd`, branch `main`
- Runtime **Docker**, instância **Free**, mesma região do Key Value
- **Health Check Path:** `/api/health`
- **Auto-deploy** ligado

Variáveis de ambiente (só nomes e exemplos, sem segredos):

| Variável | Valor / exemplo |
| --- | --- |
| `PORT` | Definido pelo Render. **Não configure.** O Dockerfile lê essa variável |
| `ENV` | `production` (desliga `/docs`) |
| `MONGO_URL` | `mongodb+srv://oleoapp_api:<senha>@<cluster>.mongodb.net/?retryWrites=true&w=majority&appName=oleoapp-api` |
| `MONGO_DB_NAME` | `oleoapp` |
| `MONGO_MAX_POOL_SIZE` | `10` |
| `MONGO_TIMEOUT_MS` | `10000` |
| `REDIS_URL` | a Internal URL do Key Value |
| `SESSION_COOKIE_NAME` | `oleo_session` |
| `SESSION_TTL_DAYS` | `7` |
| `COOKIE_SECURE` | `true` |
| `FRONTEND_URL` | `https://<projeto>.vercel.app` |
| `GOOGLE_MAPS_API_KEY` | a chave |
| `GEOCODER` | `google` ou `nominatim`, conforme o resultado da G-03 |
| `NOMINATIM_USER_AGENT` | `OleoApp/1.0 (contato: <e-mail>)` |
| `AREA_CENTER_LAT`, `AREA_CENTER_LNG`, `AREA_RADIUS_KM` | `-22.9189`, `-42.8186`, `40` |
| `EMAIL_PROVIDER` | `brevo` |
| `BREVO_API_KEY` | a chave do Brevo |
| `EMAIL_FROM`, `EMAIL_FROM_NAME` | remetente verificado no Brevo, `ÓleoApp` |
| `PASSWORD_RESET_TTL_MIN` | `30` |
| `RATE_LIMIT_GLOBAL_PER_MIN`, `RATE_LIMIT_LOGIN_PER_MIN`, `RATE_LIMIT_SIGNUP_PER_HOUR`, `RATE_LIMIT_PASSWORD_FORGOT_PER_HOUR`, `RATE_LIMIT_DIRECTIONS_PER_HOUR`, `RATE_LIMIT_GEOCODE_PER_MIN` | `120`, `5`, `5`, `5`, `30`, `30` |

**Verificação (primeiro deploy):**

```bash
curl -i https://<servico>.onrender.com/api/health          # 200
curl -i https://<servico>.onrender.com/api/health/ready    # 503, esperado até liberar o Atlas
```

## 6. [PAINEL Render → Atlas] Lista de acesso (IPs de saída)

1. No Render, abra o serviço → **Connect** (canto superior direito) → aba **Outbound** e copie **cada faixa CIDR**.
2. No Atlas, vá em **IP Access List** → **Add IP Address**, cole **uma faixa por vez** e use o comentário `Render <regiao>`.

**Verificação:**

```bash
curl -i https://<servico>.onrender.com/api/health/ready    # 200, com mongo "ok"
```

Atenção:

- **Não adicione `0.0.0.0/0`.** Se for inevitável, siga a seção 13 (Compensações).
- As faixas são **compartilhadas por todos os serviços do Render na mesma região**. A lista de acesso reduz muito a exposição, mas a **senha forte continua sendo a proteção principal**.
- IPs dedicados existem, mas são pagos.

## 7. Se o Render mudar as faixas de saída

- **Sintomas:** `/api/health/ready` responde 503 e os logs do Render mostram `ServerSelectionTimeoutError`.
- **O que fazer:** acompanhe o changelog do Render, repita o passo 6 com as faixas novas e **remova as faixas antigas**.

## 8. Importação de pontos

O Render gratuito não tem shell, então a importação é feita da sua máquina:

1. **[PAINEL Atlas]** Adicione uma entrada **TEMPORÁRIA** com o IP da sua máquina (com prazo de validade).
2. Rode primeiro a simulação, depois grave:

   ```bash
   MONGO_URL="<URI do oleoapp_maint>" MONGO_DB_NAME=oleoapp python -m scripts.import_points seed/collection_points.json
   MONGO_URL="<URI do oleoapp_maint>" MONGO_DB_NAME=oleoapp python -m scripts.import_points seed/collection_points.json --apply
   ```

3. **Confira o host impresso** pelo script: deve ser o do Atlas de produção.
4. **[PAINEL Atlas]** Remova a entrada temporária.

## 9. Rotação de senha do `oleoapp_api`

1. **[PAINEL Atlas]** Database Users → editar `oleoapp_api` → **Edit Password** → nova senha (`openssl rand -hex 24`).
2. **[PAINEL Render]** Atualize o `MONGO_URL` (o Render faz um novo deploy).
3. Confira: `curl -i https://<servico>.onrender.com/api/health/ready` deve responder 200.

Rotacione quando alguém sair da equipe e **a cada 6 meses**.

## 10. Cold start

O serviço dorme após 15 minutos sem tráfego. A primeira requisição demora cerca de 1 minuto. Nesse intervalo o frontend mostra "Acordando o servidor…", e a Vercel espera até 120 s pela resposta.

- **Não configure monitores externos** que façam ping periódico. Isso impediria o serviço de dormir e gastaria as horas gratuitas.
- **Teste de cold start:** deixe o serviço 20 minutos sem tráfego, chame `/api/health` e meça o tempo.

  | Data | Região | Tempo até responder 200 |
  | --- | --- | --- |
  | _a preencher_ | `<regiao>` | _a preencher_ |

## 11. Backups (fora do Render)

O M0 **não tem backup automático**. O que importa guardar são `users`, `disposals` e `favorites` (os pontos também estão no git, em `seed/`).

**Procedimento semanal (opção A)**, feito por uma pessoa responsável, na própria máquina e fora dos repositórios, com as MongoDB Database Tools instaladas:

1. **[PAINEL Atlas]** Adicione uma entrada temporária com o seu IP.
2. Rode:

   ```bash
   mongodump --uri "<URI do oleoapp_backup>" --db oleoapp --gzip --archive=oleoapp-AAAAMMDD.archive.gz
   ```

3. Guarde o arquivo numa pasta restrita (por exemplo, o Drive da conta do projeto, com 2FA). Ele contém e-mails e hashes de senha: **trate como dado pessoal**.
4. **[PAINEL Atlas]** Remova a entrada temporária.
5. **Retenção:** 4 cópias semanais e 3 mensais. **Frequência:** semanal, com lembrete na agenda.

**Teste de restauração (mensal)**, no Mongo local (Docker):

1. Restaure:

   ```bash
   mongorestore --gzip --archive=<arquivo> --nsFrom 'oleoapp.*' --nsTo 'oleoapp_restore.*'
   ```

2. Compare o `countDocuments()` de cada coleção com o Atlas.
3. Registre a data do teste.

> No M0 não é possível fazer dump do banco `admin` nem usar `--dumpDbUsersAndRoles`. Por isso, os usuários do Atlas são **recriados manualmente**, seguindo o passo 3 deste documento.

A opção B (automática, com GitHub Actions num repositório privado separado e a API administrativa do Atlas) é mais trabalhosa e fica como decisão futura (DA5).

## 12. Operação

- **Logs:** Render → serviço → **Logs**. A API registra método, caminho, status e duração, e exceções sem dados pessoais.
- **Falha de deploy:** **[PAINEL Render]** ative as notificações por e-mail.
- **Alertas do Atlas:** **[PAINEL Atlas]** ative os alertas por e-mail disponíveis no M0 (conexões e armazenamento) [confirmar quais estão disponíveis no M0].
- **Armazenamento:** confira todo mês o card do cluster no Atlas.
- **Estado sob demanda:** `GET /api/health/ready`.

Limites e o que acontece:

| Situação | O que acontece | O que fazer |
| --- | --- | --- |
| Banco perto de 0,5 GB | O Atlas reduz a velocidade das operações (throttling) | Revisar dados antigos ou migrar de plano |
| 750 horas gratuitas por mês | O limite vale por workspace, então **use um serviço só**. Acima disso o Render suspende os serviços gratuitos até o mês seguinte | Não criar serviços gratuitos extras |
| 30 dias sem conexões | O M0 **pausa automaticamente** | **[PAINEL Atlas]** abra o cluster e clique em **Resume**; depois confira `/api/health/ready` |

## 13. Compensações, se um dia for preciso liberar `0.0.0.0/0`

Só como último recurso e de forma temporária:

- senha aleatória longa (`openssl rand -hex 24`);
- usuário só com `readWrite` em `oleoapp`;
- nenhum outro usuário amplo no projeto;
- rotação imediata da senha;
- entrada com comentário e data;
- voltar para as faixas do Render assim que possível;
- acompanhar o **Activity Feed** do Atlas.

## Checklist de aceite

- [ ] `https://<servico>.onrender.com/api/health` responde 200 e `/api/health/ready` responde 200 com `mongo: ok` e `redis: ok`.
- [ ] A lista de acesso do Atlas não contém `0.0.0.0/0`, só as faixas do Render (e entradas temporárias que estejam vencendo).
- [ ] O usuário da API tem só `readWrite` em `oleoapp`.
- [ ] Um merge na `main` dispara deploy automático.
- [ ] `/docs` responde 404 em produção.
- [ ] Cadastro e login funcionam no Render, com o cabeçalho `X-OleoApp-Client: web`, e o cookie tem `Secure`, `HttpOnly` e `SameSite=Lax`.
- [ ] O teste de cold start foi feito e anotado na seção 10.
