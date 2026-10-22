# Curadoria de pontos de coleta

Este roteiro descreve como transformar uma lista de **candidatos** em **pontos verificados** no `seed/collection_points.json`. Não é preciso programar: o trabalho é ligar, visitar e registrar.

> Regra de ouro: **só entra no ÓleoApp quem confirmou que recebe o óleo e autorizou a divulgação.**

## Visão geral

1. Gerar a lista de candidatos (script).
2. Contatar cada candidato e perguntar o que está no roteiro abaixo.
3. Registrar só os que confirmaram e autorizaram.
4. Preencher o `seed/collection_points.json`.
5. Abrir o PR, revisar e importar no Atlas.
6. Apagar o CSV de candidatos.
7. Reverificar a cada 6 meses.

**Meta inicial:** pelo menos 10 pontos verificados em Maricá, importados no Atlas.

## 1. Gerar a lista de candidatos

O script roda só na máquina de quem faz a curadoria, nunca no Render, e precisa da `GOOGLE_MAPS_API_KEY` no `.env`:

```bash
python -m scripts.places_candidates
```

Por padrão, grava `data/candidates/candidatos-AAAAMMDD.csv`. A pasta `data/` está no `.gitignore`, então o CSV **nunca vai para o git**. Se a chave estiver vazia, o script avisa e termina.

O CSV tem as colunas `google_place_id`, `name`, `formatted_address`, `lat`, `lng`, `types`, `queries` e `google_maps_uri`.

Se a chave do Google não estiver disponível, faça a lista de candidatos à mão (pesquisa e indicações).

**Quem priorizar:** a Resolução CONAMA 362/2005 obriga os revendedores de óleo lubrificante (postos, autopeças, centros de troca) a receber o óleo usado dos consumidores [confirmar o texto da resolução]. São os melhores candidatos.

## 2. Roteiro de contato (ligação ou visita)

Para cada candidato, pergunte:

1. Vocês recebem óleo lubrificante usado de pessoas físicas, sem custo?
2. Quais tipos (carro, moto, diesel, outros)?
3. Em que dias e horários?
4. Existe limite de volume ou exigência de recipiente?
5. Podemos divulgar o ponto no ÓleoApp?

Anote as respostas na hora, com a data e o nome de quem atendeu.

## 3. Quem entra no seed

- **Entra:** quem confirmou que recebe e autorizou a divulgação.
- **Não entra:** quem não respondeu, recusou, cobra pelo serviço ou não autorizou.

Regras para os dados:

- Os dados digitados são **os confirmados com o estabelecimento**, e não os que aparecem no Google.
- **Não copie fotos, avaliações nem textos do Google.**
- Guarde só o `google_place_id` como referência [confirmar nos termos de uso do Google Maps Platform].

## 4. Preencher o `seed/collection_points.json`

Use o formato definido na tarefa BE-04. Em cada ponto confirmado:

| Campo | Valor |
| --- | --- |
| `verified` | `true` |
| `verified_at` | data da confirmação com o estabelecimento |
| `source` | `"curadoria_equipe"` |
| `address.bairro` | escrito **exatamente** como na lista oficial de `lib/bairros-marica.ts` (frontend) |
| `accepted_oils` | os tipos confirmados no contato |
| `google_place_id` | o do CSV, só como referência |

## 5. PR e importação

1. Abra um PR só com o `seed/collection_points.json`.
2. Quem revisa roda a importação **em modo de simulação** e confere 2 ou 3 pontos:

   ```bash
   docker compose exec api python -m scripts.import_points seed/collection_points.json
   ```

3. Depois do merge, a importação no Atlas segue o procedimento de [docs/deploy.md](deploy.md) (passo "Importação de pontos").

## 6. Limpeza

Ao final de cada rodada de curadoria, **apague o CSV de candidatos** da sua máquina.

## 7. Reverificação

Reverifique cada ponto a cada **6 meses**, usando o campo `verified_at`. Se o ponto deixou de receber óleo, marque-o como inativo e abra um PR com a alteração.
