# Setup — Projeto Cowork do Triathlon Dashboard

Guia para criar o projeto no Claude Cowork (Claude Desktop) e operar o fluxo
"atualizo os arquivos → o Code commita". Feito para Antonio Diniz.

---

## Pré-requisitos (uma vez)

1. **Claude Desktop** atualizado (Cowork exige a versão mais recente; plano Pro/Max/Team/Enterprise).
2. **Git instalado e autenticado** na máquina:
   - macOS: `git` já vem; autentique com `gh auth login` (instale o gh com `brew install gh`).
   - Windows: instale o **Git for Windows** — é obrigatório para a aba Code funcionar.
3. **Repositório já clonado localmente.** Se ainda não estiver, no terminal:
   ```
   git clone https://github.com/SEU_USUARIO/SEU_REPO.git
   ```
   Anote o caminho da pasta (ex: `~/Documents/triathlon-dashboard`).

A pasta do repositório deve conter: `index.html`, `data.json`, `CLAUDE.md`.

---

## Passo 1 — Criar o projeto no Cowork

1. Abra o Claude Desktop.
2. No painel lateral esquerdo, em **Projects**, clique no **+**.
3. Escolha **"Use an existing folder"** (usar pasta existente).
4. Selecione a pasta do repositório clonado (a que tem o `index.html` e o `data.json`).
5. Dê um nome: **Triathlon Dashboard**.
6. Em **Instructions**, cole o conteúdo do bloco "INSTRUÇÕES DO PROJETO" abaixo.
7. Clique em **Create**.

> Observação: os projetos do Cowork são locais (não há sync na nuvem). Se você usa
> mais de um computador, mantenha a pasta num drive sincronizado (iCloud/Dropbox).

---

## Passo 2 — Conceder acesso à pasta

Em **Settings > Cowork**, confirme que a pasta do repositório está na lista de
pastas com acesso concedido. Conceda acesso só a essa pasta (boa prática de segurança).

---

## Passo 3 — O fluxo de uso (o dia a dia)

### Registrar treino / dado novo
No Cowork, com o projeto aberto, descreva o que aconteceu em linguagem natural. Ex:
> "Registra o treino de hoje: corrida 12km, 6×1km LT2 a 5:05–5:02, FC máx 159,
> 1:15 total. Atualiza o data.json e faz o Code commitar."

O Cowork edita o `data.json` (e o `index.html` se for VO2max/narrativa), depois
**despacha para o Code** fazer `git add/commit/push`. Você recebe notificação quando termina.

### Pedir o commit explicitamente
Se você mesmo editou um arquivo e só quer publicar:
> "Abre uma sessão do Code e faz commit + push das mudanças com uma mensagem descritiva."

### Conferir resultado
Após o push, o GitHub Pages republica em ~1 min. Abra a URL do dashboard com
**refresh forçado** (Cmd/Ctrl+Shift+R) para furar o cache. Confira o selo
"⛁ dados: AAAA-MM-DD" no header — a data deve bater com a atualização.

---

## INSTRUÇÕES DO PROJETO (colar no campo Instructions do Cowork)

```
Este projeto mantém um dashboard de triathlon (Método Norueguês) do atleta Antonio Diniz,
publicado via GitHub Pages a partir desta pasta (que é um repositório Git).

ARQUITETURA
- index.html: o dashboard (Chart.js, layout, lógica). Muda raramente.
- data.json: os dados que mudam toda semana (workouts, peso, quality, inbody, vo2max).
  O index.html carrega o data.json no boot; mudou o data.json, o dashboard reflete.
- CLAUDE.md: contém o schema completo dos dados e as regras. SEMPRE leia o CLAUDE.md
  antes de editar dados, e siga o schema à risca.

REGRAS AO REGISTRAR DADOS
- Treino realizado -> novo objeto no array "workouts" do data.json, id = max+1, nunca reusar id.
- Sessão de qualidade com km planejado -> também um item em "quality_manual".
- Novo exame InBody -> item em "inbody_exams" (com seg completo) E em "weight_data".
- Peso SÓ entra em weight_data se vier de bioimpedância InBody. Balança comum não entra.
- VO2max: agora vem do data.json (array "vo2max"; meta em "vo2_goal"). Adicionar {date,v} lá.
  O index.html mantém só um fallback embutido para abertura via file://.
- Prova: alvo e referências da projeção ficam em "race" no data.json. O card de projeção
  na aba Plano recalcula sozinho ao adicionar novos PRs em "race.references".
- Sempre atualizar o campo "updated_at" no topo do data.json em toda mudança de dados.

PROPAGAÇÃO DE CONTEXTO (importante)
Registrar um treino não é só colar o número. Quando o treino for relevante (sessão de
qualidade, hills, teste, prova), escreva também uma NOTA ANALÍTICA no campo "note" do
workout: compare com o plano, comente FC vs zona, e qualquer sinal relevante. Se a
narrativa da semana (WEEK_NARRATIVES no index.html) ainda for genérica/de planejamento,
atualize-a com o que de fato aconteceu. Treinos burocráticos (Z2, regenerativo) podem
ter nota simples.

VALIDAÇÃO ANTES DE COMMITAR
- Rodar: python3 -m json.tool data.json > /dev/null  (o JSON tem que passar).
- Se mexeu no index.html, confirmar que os arrays fecham corretamente (];).
- Só então despachar para o Code: git add -A && commit + push na branch main.
  Push direto na main é permitido (repositório pessoal de arquivo único).
- Mensagem de commit descritiva: "treino: run AAAA-MM-DD — resumo" / "inbody: ..." / "ui: ...".

CONTEXTO DO ATLETA (para escrever notas com sentido)
- Prova alvo: 10K em 25/jul/2026. Pace alvo 4:50-5:00/km, busca sub-49.
- Limitador atual: muscular (perna), não cardíaco. VO2max já alto (~53,7).
- Histórico: LCA e posterior sensível — assimetria de perna esquerda em observação.
- Faixas LT2 recalibradas (semanas 9-10): segunda 5:05-5:10 (dia de reserva),
  quarta 5:10-5:15 (dia fatigado). Critério duplo: FC <=165 E pernas controladas.
- Força: Musc B (superiores) terça, Musc A (inferiores+plio) quinta, Musc C (plio leve) sábado.
- Princípio guia: nunca forçar pace; deixar FC + sensação das pernas autorizarem ajustes.
```

---

## Notas finais

- **Projetos do Cowork ainda não existem dentro do Code isoladamente** — por isso o
  fluxo é: editar/decidir no Cowork, e o Cowork despacha o commit para o Code. Os dois
  vivem no mesmo Claude Desktop.
- **Análise profunda continua valendo no chat.** Para sessões de qualidade, hills,
  testes e provas — onde a leitura fina importa — traga os dados ao Claude no chat,
  pegue a nota analítica pronta, e leve para o Cowork registrar. Treinos de rotina vão
  direto pelo Cowork.
- **O repositório é a fonte da verdade.** No início de cada sessão de análise no chat,
  suba o data.json atual do repo para o Claude trabalhar em cima da versão correta.
- VO2max e a prova-alvo (`race`) já foram migrados para o data.json (13/06/2026). O
  `index.html` mantém apenas fallbacks embutidos para abertura via file://.
```
