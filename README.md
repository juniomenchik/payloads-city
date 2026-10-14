# 🏙 Payloads City

Mapa vivo dos **endpoints, payloads, filas, webhooks e canais** de todos os seus projetos — numa página web interativa, com um JSON hierárquico que serve de “raiz” para integrar projetos entre si.

O Claude Code lê o código de cada projeto (`/mapayloads-update`), publica o mapa aqui, e, ao trocar de projeto, consulta os contratos reais dos outros (via MCP) em vez de adivinhar.

```
 projeto A ──/mapayloads-update──┐
 projeto B ──/mapayloads-update──┼──►  Payloads City  (http://localhost:6500)
 projeto C ──/mapayloads-update──┘        │  página web · JSON root · API · MCP
                                          ▼
              Claude Code consulta os payloads corretos de A, B e C
```

## Subir (1 comando)

```bash
docker compose up -d          # http://localhost:6500
DEMO=1 docker compose up -d   # com 5 projetos de exemplo, para ver o mapa funcionando
```

Os dados ficam no volume `payloads-city-data` e sobrevivem a reinicializações.

## Instalar o plugin no Claude Code

```text
/plugin marketplace add /caminho/para/payloads-city
/plugin install payloads-city@payloads-city
```

(ou, para testar sem instalar: `claude --plugin-dir /caminho/para/payloads-city/plugin`)

Você ganha:

| O quê | Para quê |
|---|---|
| `/mapayloads-update` | Dentro de um projeto: o Claude mapeia endpoints, payloads de entrada/saída, filas, webhooks, canais, crons e chamadas a outros projetos, e publica aqui. |
| `/mapayloads-get [projeto \| busca]` | Consulta rápida do catálogo. |
| MCP `payloads-city` | Ferramentas `list_projects`, `get_project`, `get_endpoint`, `search`, `get_graph`, `get_format`. |
| Skill `consult-payloads-city` | Faz o Claude consultar o catálogo sozinho ao integrar com outro projeto. |

Se o servidor não estiver em `localhost:6500`, defina `PAYLOADS_CITY_URL`.

## O que a página mostra

- **🏙 Mapa** — cada projeto é um prédio (as janelas acesas crescem com o nº de entradas); filas/tópicos/canais são “praças”; os fluxos são animados e mudam de cor por tipo (HTTP, publica, consome, webhook). Arraste, clique para detalhes, duplo clique abre o projeto. **Vermelho = chamada quebrada** (aponta para endpoint que não existe); **⚠ sem par** = fila publicada que ninguém consome (ou o contrário).
- **📚 Projetos** — endpoints por grupo/método, com parâmetros, query, headers, body, respostas, exemplo de JSON, `curl`, quem consome cada endpoint, e a aba de filas/webhooks/canais.
- **{ } JSON root** — o catálogo hierárquico completo, para copiar ou baixar.

## Formato (resumo)

Cada projeto é um JSON (`GET /api/format` mostra a documentação completa):

```jsonc
{
  "name": "comanda-api", "role": "provider", "stack": "laravel", "baseUrl": "http://localhost:8000/api",
  "endpoints": [{ "method": "POST", "path": "/orders", "auth": "bearer",
                  "body": { "fields": [{ "name": "items", "type": "array", "required": true, "children": [] }] },
                  "responses": [{ "status": 201, "fields": [] }] }],
  "events":  [{ "kind": "rabbitmq", "direction": "out", "name": "order.paid", "payload": [] }],
  "calls":   [{ "project": "outro-projeto", "method": "GET", "path": "/menu" }]
}
```

`events.kind`: `rabbitmq kafka sqs sns redis nats websocket sse webhook cron job other`.
Eventos de mesma `kind`+`name` em projetos diferentes (um `out`, outro `in`) são ligados automaticamente, e `calls` são conferidas contra os endpoints do projeto de destino.

## API

| Rota | |
|---|---|
| `GET /api/catalog` | JSON root: projetos, estatísticas e grafo de conexões |
| `GET /api/projects` · `GET /api/projects/:name` | resumo / mapa completo |
| `PUT /api/projects/:name` | publica/atualiza (valida; 422 com `issues` se o formato estiver errado) |
| `DELETE /api/projects/:name` | remove |
| `GET /api/search?q=&method=&project=&scope=` | busca em endpoints e eventos |
| `GET /api/format` | documentação do formato |
| `GET /api/health` | healthcheck |

## Desenvolvimento

```bash
cd server && npm install && npm run dev     # API em :6500 (DATA_DIR=./data, DEMO=1 para exemplos)
cd web    && npm install && npm run dev     # UI em :5173 com proxy para :6500
```

Estrutura: `server/` (Fastify + zod, arquivos JSON), `web/` (Vue 3 + Vite), `plugin/` (comandos, skill e MCP sem dependências), `examples/` (dados de demonstração).

## Segurança

Feito para uso **local**: não há autenticação, e o `docker-compose.yml` publica a porta apenas em `127.0.0.1`. Para expor em rede, coloque atrás de um proxy com autenticação. O mapa guarda só o *formato* dos dados — o comando `/mapayloads-update` orienta o Claude a nunca incluir segredos.

## Limitações conhecidas

- Quem interpreta o código é o Claude: o mapa pode ter pequenos erros. Cada projeto mostra a data do último mapeamento, branch e commit para você saber quando atualizar.
- Não há geração automática de OpenAPI/AsyncAPI (ainda): é uma boa evolução exportar o catálogo nesses formatos.

## Licença

MIT
# payloads-city
