# Grafana FLEX NOC

Dashboards Grafana da FLEX Internet para o NOC. Pasta no Grafana: `REDE - FLEX`.

Dois grupos de dashboard, com fontes de dados distintas:

- **MAPA FLEX** — mapa geográfico de CTOs e prédios, cruzando cadastro do IXC (MySQL/MariaDB) com
  sessões RADIUS, para detectar queda e atenuação óptica em massa.
- **NOC — Visão Geral da Rede** — mapa esquemático da topologia física (OLTs, switches e núcleo),
  alimentado pela API JSON-RPC do Zabbix via plugin Infinity.

## Dashboards

| Arquivo | Título | O que mostra |
|---|---|---|
| [`dashboards/mapa-flex.json`](dashboards/mapa-flex.json) | **MAPA FLEX** | Mapa geográfico com marcador por CTO/prédio: verde (normal), cinza (sem cliente considerado), vermelho grande com rótulo (queda confirmada: 2+ clientes desconectados nas últimas 48h e 50%+ do ponto afetado). Tabela lateral "🔴 QUEDAS EM ANDAMENTO". |
| [`dashboards/mapa-flex-status-ctos.json`](dashboards/mapa-flex-status-ctos.json) | **MAPA FLEX - STATUS CTO's** | Clone do MAPA FLEX que soma detecção de **atenuação óptica em massa**: CTO/prédio com 2+ logins ativos onde 100% têm último sinal RX ≤ -27 dBm ganha marcador laranja sobreposto no mapa (sem ícone/rótulo grande, reservado à queda). Tabela unificada "🔴🟠 QUEDAS E CTOs ATENUADAS" com coluna `Status`. |
| [`dashboards/noc-visao-geral-da-rede.json`](dashboards/noc-visao-geral-da-rede.json) | **NOC — Visão Geral da Rede** | Topologia física em Node Graph: 31 equipamentos em posição fixa por POP e 33 enlaces PtP. Cor da linha = status (verde UP, laranja LAG degradado, vermelho down), espessura = tráfego. Anel de cada nó dividido em tipo de equipamento + status ao vivo do Zabbix. Tabela de banda por enlace (download/upload) e alertas críticos. Feito para TV do NOC (1920x1080, sem rolagem). |

## MAPA FLEX — lógica de negócio (resumo)

- **Queda**: `offline >= 2` clientes **e** `>= 50%` do total de clientes ativos/recentes (últimas
  48h) do ponto. Cliente sem sessão em 48h é ignorado do cálculo (não é evidência de queda nem de
  normalidade).
- **Atenuação** (só no MAPA FLEX - STATUS CTO's): `total_logins >= 2` **e** 100% deles com último
  `sinal_rx <= -27` dBm (RX óptico — quanto mais negativo, pior). Queda tem prioridade sobre
  atenuação no mesmo ponto.
- CTOs (`rad_caixa_ftth`) e prédios com rede interna (`predio`) são fontes distintas, unificadas
  via `UNION ALL` — um condomínio com rede própria não aparece em `rad_caixa_ftth`.

## NOC — Visão Geral da Rede

Mapa **esquemático** (não geográfico) da rede de transporte. Responde de relance:

| Pergunta | Resposta no mapa |
|---|---|
| Que equipamento está inacessível? | anel do nó vermelho |
| Que enlace caiu? | linha vermelha |
| Que LAG está degradado? | linha laranja (1 de 2 membros down) |
| Por onde passa mais tráfego? | espessura da linha |
| Quanto exatamente? | hover na linha + tabela abaixo |

### Topologia modelada

```
                    Internet
                       │
                     BR10 (Borda / BGP)
                       │  Eth-Trunk (LAG 4x10G)
                    SWC01 (Core)
                       │  Eth-Trunk (LAG 4x10G)
                    CR110 (BNG)
                       │  Eth-Trunk AGG (LAG 4x10G)
                    SWA01 (agregação principal)
                    ╱   │   ╲
              SWA02   OLTs   SWA03
```

### Convenções visuais

O anel de cada nó soma 1.0: **0.25 indica o tipo** de equipamento e **0.75 indica o status**.

| Tipo | Cor da fatia |
|---|---|
| OLT | branco `#E0E0E0` |
| Switch de agregação | azul |
| Core / Borda (BNG, Core, Borda) | roxo |

| Status | Cor dos 75% |
|---|---|
| Sem alerta ≥ Média nos últimos 7 dias | verde |
| Alerta severidade Média | amarelo |
| Alerta severidade Alta/Crítica | vermelho |

Alertas de severidade Warning/Info são **ignorados de propósito** — senão o mapa fica todo amarelo
por alarme de ONU, que não é problema de rede.

**Só alarme dos últimos 7 dias colore o nó.** Alarme crônico (óptica degradada há meses, sessão BGP
desativada) deixava metade dos switches vermelhos permanentemente, e a cor perdia o sentido.
Implementado com `"lastChangeSince": "${__from:date:seconds}"` nas queries de status do nó e
`"timeFrom": "7d"` no painel do mapa, que sobrepõe o seletor de tempo do dashboard só ali. O alarme
antigo continua na tabela de alertas, com a coluna "Desde" mostrando há quanto tempo está aberto.

### Detalhes de implementação

- Todas as queries usam o plugin **Infinity** batendo direto na API JSON-RPC do Zabbix. O
  datasource nativo do Zabbix não expõe os métodos necessários (`trigger.get`, `summarize`,
  filtros compostos).
- O token do Zabbix vai **no corpo** da requisição (campo `auth` do JSON-RPC), não em header.
- As posições dos nós são **fixas** (`fixedX`/`fixedY`), calculadas fora do Grafana por um
  otimizador (recozimento simulado em pixels de tela) que garante zero cruzamento de linha, zero
  linha passando por cima de equipamento e zero rótulo sobreposto. O Node Graph não faz auto-fit:
  as coordenadas são pixels do viewBox, centrado em (0,0).
- O painel de alertas filtra "Link down" pelos **`itemids` dos enlaces desenhados no mapa** — e não
  por host ou por texto. Assim aparecem exatamente as linhas vermelhas do mapa, sem trazer
  interface de cliente ou de gerência do mesmo switch.

A lógica completa (armadilhas do plugin Node Graph, do Infinity/Zabbix e das transformações do
Grafana, o otimizador de layout, procedimento para adicionar equipamento) está documentada em
repositório interno separado de skills operacionais — este repositório guarda apenas os **exports
JSON dos dashboards** (para restore/versionamento/diff), não duplica a documentação de lógica.

## Publicar/restaurar um dashboard via API do Grafana

```bash
# Publicar/atualizar (uid fixo do payload = sobrescreve o dashboard existente)
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" -X POST "$GRAFANA_URL/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -d '{"dashboard": <conteúdo do json>, "folderUid": "<uid da pasta REDE - FLEX>", "overwrite": true}'

# Clonar (remover "id" e "uid" do JSON antes de enviar → gera um dashboard novo)
```

### Placeholders no `noc-visao-geral-da-rede.json`

Esse dashboard chama a API do Zabbix direto das queries, então URL e token aparecem no JSON. Ambos
foram substituídos por placeholders — troque antes de importar:

| Placeholder | Substituir por |
|---|---|
| `__ZABBIX_URL_MAIN__` | URL do `api_jsonrpc.php` do Zabbix principal |
| `__ZABBIX_URL_ALT__` | URL do `api_jsonrpc.php` do Zabbix secundário |
| `__ZABBIX_TOKEN_MAIN__` | token de API do Zabbix principal |
| `__ZABBIX_TOKEN_ALT__` | token de API do Zabbix secundário |

```bash
sed -e "s|__ZABBIX_URL_MAIN__|$ZBX_URL|g"       -e "s|__ZABBIX_URL_ALT__|$ZBX_URL_ALT|g" \
    -e "s|__ZABBIX_TOKEN_MAIN__|$ZBX_TOKEN|g"   -e "s|__ZABBIX_TOKEN_ALT__|$ZBX_TOKEN_ALT|g" \
    dashboards/noc-visao-geral-da-rede.json > /tmp/noc-restore.json
```

Também é preciso que o Grafana de destino tenha o **plugin Infinity** instalado e os datasources
com os UIDs referenciados no JSON (2 apontando para os Zabbix, 1 para CSV inline).

Nenhum segredo/token/IP/domínio está versionado — os JSONs trazem só definição de painéis, queries
SQL contra tabelas do IXC e placeholders.
