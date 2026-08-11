# Grafana FLEX NOC — MAPA FLEX

Dashboards Grafana da FLEX Internet para o NOC: mapa geográfico em tempo real de CTOs (caixas
FTTH) e prédios com rede interna, cruzando cadastro de infraestrutura do IXC com sessões RADIUS
ativas, para detecção visual imediata de queda e atenuação óptica em massa.

Datasource: MySQL/MariaDB (banco de produção do IXC, acesso read-only via datasource Grafana —
sem ETL). Pasta no Grafana: `REDE - FLEX`.

## Dashboards

| Arquivo | Título | O que mostra |
|---|---|---|
| [`dashboards/mapa-flex.json`](dashboards/mapa-flex.json) | **MAPA FLEX** | Mapa geográfico com marcador por CTO/prédio: verde (normal), cinza (sem cliente considerado), vermelho grande com rótulo (queda confirmada: 2+ clientes desconectados nas últimas 48h e 50%+ do ponto afetado). Tabela lateral "🔴 QUEDAS EM ANDAMENTO". |
| [`dashboards/mapa-flex-status-ctos.json`](dashboards/mapa-flex-status-ctos.json) | **MAPA FLEX - STATUS CTO's** | Clone do MAPA FLEX que soma detecção de **atenuação óptica em massa**: CTO/prédio com 2+ logins ativos onde 100% têm último sinal RX ≤ -27 dBm ganha marcador laranja sobreposto no mapa (sem ícone/rótulo grande, reservado à queda). Tabela unificada "🔴🟠 QUEDAS E CTOs ATENUADAS" com coluna `Status`. |

## Lógica de negócio (resumo)

- **Queda**: `offline >= 2` clientes **e** `>= 50%` do total de clientes ativos/recentes (últimas
  48h) do ponto. Cliente sem sessão em 48h é ignorado do cálculo (não é evidência de queda nem de
  normalidade).
- **Atenuação** (só no MAPA FLEX - STATUS CTO's): `total_logins >= 2` **e** 100% deles com último
  `sinal_rx <= -27` dBm (RX óptico — quanto mais negativo, pior). Queda tem prioridade sobre
  atenuação no mesmo ponto.
- CTOs (`rad_caixa_ftth`) e prédios com rede interna (`predio`) são fontes distintas, unificadas
  via `UNION ALL` — um condomínio com rede própria não aparece em `rad_caixa_ftth`.

A lógica completa (armadilhas de dado do IXC, queries SQL comentadas, decisões de UX do painel
Geomap, cores/thresholds) está documentada em repositório interno separado de skills operacionais
— este repositório guarda apenas os **exports JSON dos dashboards** (para restore/versionamento/
diff), não duplica a documentação de lógica.

## Publicar/restaurar um dashboard via API do Grafana

```bash
# Publicar/atualizar (uid fixo do payload = sobrescreve o dashboard existente)
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" -X POST "$GRAFANA_URL/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -d '{"dashboard": <conteúdo do json>, "folderUid": "<uid da pasta REDE - FLEX>", "overwrite": true}'

# Clonar (remover "id" e "uid" do JSON antes de enviar → gera um dashboard novo)
```

Nenhum segredo/token/IP está versionado nos JSONs — são apenas definição de painéis e queries SQL
contra tabelas do IXC.
