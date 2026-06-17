# Dashboard de Triathlon — Antonio Diniz

## Arquitetura

Site estático no GitHub Pages, com **dados separados do código**:

- `index.html` — código (gráficos Chart.js, layout, lógica). Muda raramente.
- `data.json`  — dados (treinos, peso, InBody, overrides de qualidade). Muda toda semana.
- `CLAUDE.md`  — este arquivo.

`index.html` carrega `data.json` via XHR síncrono no boot. Quando os dados externos
carregam, um selo "⛁ dados: YYYY-MM-DD" aparece no header — é como conferir se o site
está servindo a versão certa.

**Princípio: fonte única da verdade.** Cada dado mora só no `data.json`; o `index.html`
só renderiza e deriva (entries/sessions/gráficos) em runtime. Migração em andamento para
tirar do HTML tudo que é dado:
- ✅ `config` (FTP, FCmáx, zonas, baselines, metas, cycle_start) — no banco.
- ✅ `hr_rest` (FC repouso) — no banco.
- ✅ `workouts` — **sem fallback embutido**; `data.json` é a única fonte (abrir via
  `file://` não mostra treinos — usar o site publicado ou um servidor local).
- ✅ `plan` — todo o plano no banco: `weekly_plan` (molde por dia-da-semana),
  `plan_phases` (overrides por faixa de data), `cal_overrides` (plano DIA A DIA — a
  fonte de edição preferida), `cycle_phases` (fases 1–14), `planned_weekly` (volume),
  `workout_details` (conteúdo dos modais). Sem fallback embutido (só `{}`/`[]`).
- 🔄 cards da aba Plano: os valores de config/prova (FTP, W/kg, faixa LT2, data e pace
  da prova) já renderizam do banco via `renderPlanoCards()` (IDs `pc-*`). Textos
  descritivos (grade "Semana Tipo", pliometria, volumes S7–8) seguem estáticos como
  referência — mudam raramente.

Config e hr_rest ainda têm um fallback mínimo embutido (segurança); workouts e plan não.

Precedência do plano em runtime: `cal_overrides[data]` → `plan_phases` (faixa) →
`weekly_plan[dow]`. Editar um dia = editar `plan.cal_overrides` no `data.json`.

O cache (localStorage) invalida sozinho via hash do conteúdo dos workouts — mudou o
`data.json`, o app recarrega os dados sem precisar de versão manual.

## Schema do data.json (schema_version: 1)

```json
{
 "schema_version": 1,
 "updated_at": "YYYY-MM-DD",          // atualizar a CADA mudança de dados
 "config": {                           // config do atleta — fonte única (evita FTP repetido)
   "cycle_start": "2026-04-20", "ftp": 243, "ftp_ref": "Ramp Test 06/06/2026",
   "fcmax": 183,
   "baseline":      { "run": 360, "bike": 1.0, "swim": 120 },  // run/swim=sec, bike=W/kg
   "goal_target":   { "run": 300, "bike": 3.0, "swim": 110 },
   "goal_baseline": { "run": 540, "bike": 1.0, "swim": 330 }
 },
 "hr_rest": {                          // FC repouso (migrado do index.html)
   "alert": 60,
   "data": [ { "date": "2026-06-07", "bpm": 54 } ]
 },
 "workouts": [                         // fonte única de treinos realizados (sem fallback no HTML)
   { "id": 129, "date": "2026-06-10", "sport": "run|bike|swim|strength",
     "dist": 12.0, "dur": "1:18:30", "pace": "5:09", "sec": 309,
     "hr": 150, "kcal": 900,           // opcionais
     "blocks": {"done": 5, "total": 5}, // opcional (sessões de qualidade)
     "note": "texto livre · separado por ·" }
 ],
 "weight_data": [                      // SÓ com bioimpedância (regra do atleta)
   { "date": "2026-06-12", "kg": 89.9 }
 ],
 "quality_manual": [                   // overrides de km de qualidade (ajuste humano)
   { "date": "2026-06-10", "sport": "run", "done": 7.5, "planned": 7.5 }
 ],
 "inbody_exams": [                     // exames completos, com segmentar
   { "date": "2026-06-12", "label": "12/06", "hora": "08:38",
     "peso": 89.9, "musculo": 42.9, "gc_pct": 16.6, "gordura_kg": 14.9,
     "score": 89, "tmb": 1989, "visceral": 6,
     "seg": { "braco_e": 4.31, "braco_d": 4.43, "tronco": 32.5,
              "perna_e": 10.97, "perna_d": 11.10 } }
 ],
 "vo2_goal": 55,                       // meta de VO2máx (fallback no HTML = 55)
 "vo2max": [                           // leituras do Apple Watch (migrado do index.html)
   { "date": "2026-06-13", "v": 53.7 }
 ],
 "race": {                             // prova-alvo + referências p/ projeção (card na aba Plano)
   "name": "10K — Corrida POUPEX", "date": "2026-07-25",
   "start_time": "18:00", "location": "Praça dos Cristais — SMU, Brasília/DF",
   "distance_km": 10, "target_time": "48:59", "target_pace": "4:54",
   "course": {                         // resumo + altimetria (mini gráfico no card)
     "summary": "plano, técnico nos 2 km iniciais ...", "ascent_m": 35,
     "elevation": [ { "km": 0, "alt": 1140 }, { "km": 1.0, "alt": 1152 } ]
   },
   "references": [                     // esforços usados na projeção Riegel
     { "date": "2026-05-10", "dist": 5, "time": "24:45", "label": "5K (Wings for Life)" }
   ]
 }
}
```

Regras dos dados:
- `id` de workout: sempre `max(id) + 1`. Nunca reaproveitar.
- `sec` é o pace em segundos (run/swim: por km/100m; bike: usa watts na note + sec equivalente).
  **Sessão de qualidade (tiros): `sec` = média do pace dos TIROS, não da sessão inteira** —
  é `sec` que alimenta o gráfico "Evolução de Pace". `dist`/`dur` seguem o total da sessão.
  (Ex: 6×1km a 4:42 numa corrida de 53min → `sec:282`, `dist`/`dur` da sessão toda.)
- **Cuidado com as PALAVRAS da `note`**: o gráfico "Eficiência Aeróbica" (`buildEFChart`)
  classifica corrida fácil vs. forte pelo texto da nota — inclui se bate
  `z1|z2|regenerativo|longão|longo|recupera|matinal` E NÃO bate
  `lt2|limiar|threshold|prova|teste|vo2`. Numa corrida fácil (Z2), NÃO use termos HARD na
  nota (ex: "depois do limiar" exclui o treino do gráfico por engano). Use o `sec` da
  sessão nas fáceis (sem tiros) — é ele que vira o EF.
- Peso (`weight_data`) só entra com exame de bioimpedância — NUNCA com balança comum.
  Peso de balança pode ir na `note` de um workout ou ser usado em análise, mas não aqui.
- `updated_at` deve ser atualizado em todo commit que mexa em dados.
- Não remover exames/treinos antigos — o histórico é o ativo.
- VO2máx agora vem do `data.json` (`vo2max`/`vo2_goal`), não mais hardcoded no HTML.
  Nova leitura → adicionar `{date,v}` em `vo2max`. O `index.html` ainda guarda um
  fallback embutido (para abrir via `file://`).
- Projeção da prova: ajustar a prova-alvo em `race` (data, meta, `references`). O card
  recalcula sozinho (Riegel + ganho/semana). Adicionar novos PRs em `references` melhora a estimativa.

## Fluxos de atualização

**Registrar treino** (pedidos tipo "registra a corrida de hoje: ..." / "puxa o treino"):
1. Puxar a sessão do Strava (MCP) e extrair: distância, tempo, pace/W, FC média/máx,
   elevação, kcal, relative effort, melhores esforços, segmentos/PRs e, em sessões de
   qualidade, os laps/blocos (pace + FC por bloco).
2. **Entregar a análise profunda** (ver seção "Análise profunda de treino" abaixo)
   ANTES de fechar — é o entregável principal, não o registro.
3. Editar `data.json`: adicionar o objeto em `workouts` (e em `quality_manual` se for
   sessão de qualidade com km planejado), atualizar `updated_at`.
4. Validar JSON: `python3 -m json.tool data.json > /dev/null`.
5. Commit: `treino: <sport> <data> — <resumo curto>` e push.

## Análise profunda de treino

Sempre que o Antonio mandar puxar/registrar um treino, entregar uma análise completa —
não só os números crus. A análise deve cobrir:

- **Vs. plano da semana**: qual slot do Método Norueguês a sessão corresponde (LT2 AM/PM,
  longão Z1–Z2, regenerativo, bike longo etc.) e se bateu volume, pace e zona-alvo
  prescritos. Apontar desvios (mais rápido/lento, mais/menos km) e se foram positivos.
- **Eficiência aeróbica (EF)**: velocidade (m/min) ÷ FC média. Comparar com sessões
  semelhantes anteriores — subindo = mesmo esforço, mais rápido (a métrica do volume fácil).
- **Desacoplamento (Pa:HR / Pw:HR drift)**: comparar 1ª vs 2ª metade. <5% = base sólida;
  >5% sob calor/déficit = atenção à recuperação.
- **Distribuição de zonas e FC**: tempo em cada zona, FC média/máx vs zonas (FCmáx 183).
  Sinalizar se um treino "fácil" subiu para gray zone.
- **Sessão de qualidade (LT2/VO2)**: bloco a bloco — pace e FC de cada tiro vs alvo,
  quantos completou (done/total), tendência ao longo dos blocos (estável vs derretendo),
  e se houve PR de pace de limiar.
- **Carga e contexto**: relative effort, volume acumulado da semana, encadeamento com os
  treinos vizinhos (recuperação suficiente?), e o contexto de déficit/Mounjaro quando
  relevante (fueling, recuperação rebaixada).
- **Progressão vs prova**: o que o treino sinaliza para a meta da `race` (projeção Riegel,
  pace de prova) e para os objetivos do ciclo.
- **Recado acionável**: 1–3 conclusões práticas — o que manter, o que ajustar no próximo
  treino semelhante, e qualquer bandeira de alerta (FC derivando, sinais de overreaching).

Tom: técnico, direto, honesto. Elogiar o que foi bem, mas apontar riscos sem suavizar.

**Novo exame InBody**: adicionar em `inbody_exams` (com `seg` completo) E em
`weight_data`. Os gráficos e cards do perfil se atualizam sozinhos.

**Mudança de plano**: editar `plan` no `data.json` — dia a dia em `plan.cal_overrides`
(`"YYYY-MM-DD": [ {sport,key,label,detail,zone,color}, ... ]`); detalhe do modal em
`plan.workout_details[key]`. Validar JSON. Commit `plano: ...`. (Só mudança visual/lógica
de fato mexe no `index.html` → commit `ui: ...`.)

**Publicar**: `git add -A && git commit -m "<msg>" && git push origin main`.
Push direto na main é permitido e esperado (repo pessoal). GitHub Pages republica ~1min.

## Regras gerais

- Antes de commitar: validar o JSON e, se mexeu no HTML, abrir sem erro de console.
- Nunca commitar segredos. O site é público por link.
- O repositório é a fonte da verdade — não o projeto do Claude.ai. Ao trabalhar com o
  Claude no chat, subir o data.json atual do repo no início da conversa.
- localStorage (refeições, água) é por dispositivo e NÃO é versionado aqui.
