# Dashboard de Triathlon — Antonio Diniz

## Arquitetura

Site estático no GitHub Pages, com **dados separados do código**:

- `index.html` — código (gráficos Chart.js, layout, lógica). Muda raramente.
- `data.json`  — dados (treinos, peso, InBody, overrides de qualidade). Muda toda semana.
- `CLAUDE.md`  — este arquivo.

`index.html` carrega `data.json` via XHR síncrono no boot. Se falhar (ex: aberto via
`file://`), usa o fallback embutido no próprio HTML. Quando os dados externos carregam,
um selo "⛁ dados: YYYY-MM-DD" aparece no header — é como conferir se o site está
servindo a versão certa.

O cache (localStorage) invalida sozinho via hash do conteúdo dos workouts — mudou o
`data.json`, o app recarrega os dados sem precisar de versão manual.

## Schema do data.json (schema_version: 1)

```json
{
 "schema_version": 1,
 "updated_at": "YYYY-MM-DD",          // atualizar a CADA mudança de dados
 "workouts": [                         // fonte única de treinos realizados
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
   "name": "10K — Brasília", "date": "2026-07-25", "distance_km": 10,
   "target_time": "48:59", "target_pace": "4:54",
   "references": [                     // esforços usados na projeção Riegel
     { "date": "2026-05-10", "dist": 5, "time": "24:45", "label": "5K (Wings for Life)" }
   ]
 }
}
```

Regras dos dados:
- `id` de workout: sempre `max(id) + 1`. Nunca reaproveitar.
- `sec` é o pace em segundos (run/swim: por km/100m; bike: usa watts na note + sec equivalente).
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

**Registrar treino** (pedidos tipo "registra a corrida de hoje: ..."):
1. Editar `data.json`: adicionar o objeto em `workouts` (e em `quality_manual` se for
   sessão de qualidade com km planejado), atualizar `updated_at`.
2. Validar JSON: `python3 -m json.tool data.json > /dev/null`.
3. Commit: `treino: <sport> <data> — <resumo curto>` e push.

**Novo exame InBody**: adicionar em `inbody_exams` (com `seg` completo) E em
`weight_data`. Os gráficos e cards do perfil se atualizam sozinhos.

**Mudança de plano/visual**: editar `index.html` (estrutura `mn_*`, WORKOUT_DETAILS,
CAL_OVERRIDES seguem no HTML por enquanto). Commit: `plano: ...` ou `ui: ...`.

**Publicar**: `git add -A && git commit -m "<msg>" && git push origin main`.
Push direto na main é permitido e esperado (repo pessoal). GitHub Pages republica ~1min.

## Regras gerais

- Antes de commitar: validar o JSON e, se mexeu no HTML, abrir sem erro de console.
- Nunca commitar segredos. O site é público por link.
- O repositório é a fonte da verdade — não o projeto do Claude.ai. Ao trabalhar com o
  Claude no chat, subir o data.json atual do repo no início da conversa.
- localStorage (refeições, água) é por dispositivo e NÃO é versionado aqui.
