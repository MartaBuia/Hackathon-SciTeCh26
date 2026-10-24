# FIXING CP — Apoio à decisão na via única da Linha do Douro

Protótipo académico (hackathon) que ajuda os chefes de estação da **via única Marco de Canaveses – Pocinho** a decidir **que comboio avança** quando dois comboios querem o mesmo troço em sentidos opostos, e que **dá o alerta** quando um comboio fica retido num troço tempo demais.

Construído sobre o **Coreflux** (LoT + Python + connector de email).

> ⚠️ **Aviso:** protótipo de apoio à decisão. Não substitui os procedimentos de segurança da CP / IP nem qualquer sistema de sinalização.

---

## O problema

Na via única, cada troço entre duas estações seguidas só pode ter um comboio de cada vez. Quando dois comboios se querem cruzar, um tem de esperar. Na prática avança quem foi despachado primeiro, mas isso nem sempre minimiza o atraso total na linha. Além disso, em grande parte da linha não há rede nem Wi-Fi, por isso um comboio parado num troço pode passar despercebido.

## Os dois subsistemas

| Subsistema | Notebook | O que faz |
|---|---|---|
| **1 — Decisão de cruzamentos** | `Sistema1.lotnb` | Deteta conflitos, calcula (MILP) qual dos dois comboios provoca menos atraso total, e envia a ordem *avançar* / *reter* por email. |
| **2 — Alertas de via única** | `Sistema2.lotnb` | Vigia o tempo de ocupação de cada troço, avisa o chefe de estação e escala para a central da CP. |

### Divisão de trabalho

- **LoT**: estado dos troços, deteção de conflitos, mensagens, emails, painel.
- **Python**: horário dos comboios e otimização (MILP com SciPy); preparação do conteúdo dos emails e histórico.
- **Connector EMAIL**: envio via SMTP (Mailpit no ambiente de desenvolvimento).

---

## Subsistema 1 — Decisão de cruzamentos (`FCP11*`)

### Como decide

1. O chefe de estação envia eventos `pronto`, `partida` e `chegada`.
2. Se o troço está livre e ninguém espera do lado oposto → recomenda **AVANÇAR**.
3. Se há um comboio à espera no sentido oposto → pede uma decisão ao motor Python.
4. O motor resolve o horário **duas vezes** (*"se avançar A"* e *"se avançar B"*) e compara o **atraso total na linha**.
5. Avança o que provoca menos atraso. Em caso de empate, avança o que já estava à espera.
6. O LoT envia 3 emails: chefe da estação que avança, chefe da que aguarda, e registo na CP.

### Modelo de horário

- Horas de partida/chegada vêm do PDF de horários do grupo (`HORARIOS_TXT`).
- Passagens intermédias são **estimadas** proporcionalmente ao perfil do IR 861 (o único com todas as horas).
- Como a estimativa gera cruzamentos impossíveis, calcula-se um **horário de referência ajustado sem conflitos**; os atrasos são medidos face a ele.
- Margem de segurança entre comboios no mesmo troço: `MARGEM_MIN = 2` min.
- Nunca se antecipam partidas.
- Pressuposto: todas as estações da via única têm linha de cruzamento.

### Tópicos

| Tópico | Payload | Significado |
|---|---|---|
| `LinhaDouro/v11/estacoes/<Estacao>/pronto` | `{"comboio","proxima","hora"}` | Comboio pronto para entrar no troço |
| `LinhaDouro/v11/estacoes/<Estacao>/partida` | `{"comboio","proxima","hora"}` | Comboio entrou no troço |
| `LinhaDouro/v11/estacoes/<Estacao>/chegada` | `{"comboio","anterior","hora"}` | Comboio chegou e libertou o troço |
| `LinhaDouro/v11/config/padrao` | qualquer | Define emails dos 16 chefes e da CP (correr uma vez) |
| `LinhaDouro/v11/config/estacao/<Estacao>` | `{"email"}` | Altera o email de uma estação |
| `LinhaDouro/v11/sistema/carregar` | `{}` | Calcula o horário ajustado |
| `LinhaDouro/v11/consulta/horario` | `{"comboio"}` | Horário estimado/ajustado de um comboio |
| `LinhaDouro/v11/recomendacoes/<Estacao>` | texto | Recomendação para essa estação |
| `LinhaDouro/v11/erros`, `.../alertas`, `.../eventos` | texto | Erros, perigos e log de eventos |

Estado guardado: `LinhaDouro/v11/trocos/<De>_<Para>/espera` e `.../ocupado`.

### Painel

`FixingCPDecisoes` mostra atraso se avançar A, atraso se avançar B, atraso evitado, troço, recomendação, comboio a aguardar, justificação e último evento.

---

## Subsistema 2 — Alertas de via única (`FCP10*`)

Cada troço vigiado guarda quem o ocupa, a hora de entrada e um **limite** (`entrada + duração prevista + tolerância`).

| Nível | Quando | Destinatários |
|---|---|---|
| `AVISO` | Passou o limite | Chefes das 2 estações + registo CP |
| `CRITICO` | Passou limite + tempo de escalada (possível acidente) | Chefes + registo CP + **emergência CP** |
| `CONFLITO` | 2.º comboio entra num troço ocupado | Chefes + registo CP + **emergência CP** |
| `RESOLVIDO` | Comboio sai depois de ter havido alerta | Chefes + registo CP |

O Python gera o JSON do email, instala o template HTML e guarda o histórico em `/tmp/fcp_ocorrencias_via_unica.jsonl`.

### Tópicos principais

- `LinhaDouro/v7/trocos/<Troco>/vigilancia/config` — `{"tolerancia_min": ..., "escalada_min": ...}`
- `LinhaDouro/v7/trocos/<Troco>/entrada` — `{"servico","hora_min","duracao_prevista_min"}`
- `LinhaDouro/v7/trocos/<Troco>/saida` — `{"servico","hora_min"}`
- `LinhaDouro/v7/simulacao/relogio` — `{"agora_min": ...}` (relógio simulado, verifica os troços D01–D05)
- `LinhaDouro/v7/cp/ocorrencias/consultar` — `{"ultimas": 20}` devolve o histórico

---

## Instalação

**Requisitos:** broker Coreflux com suporte a Python, `numpy` e `scipy`, e um servidor SMTP (Mailpit em `mailpit:1025`; interface em http://localhost:8025).

1. Importar `Sistema1.lotnb` e `Sistema2.lotnb` no Coreflux.
2. Executar as células pela ordem (modelos → script Python → ações → connector → painel).
3. Garantir que o template `/templates/fcp_recomendacao.html` existe no broker (usado pelo Subsistema 1).
4. Configurar cada troço vigiado do Subsistema 2 (ver acima), incluindo `estado/a` e `estado/b` com os IDs das estações.

## Teste rápido (Subsistema 1)

No Data Viewer, publicar por esta ordem:

1. `LinhaDouro/v11/sistema/carregar` → `{}`
2. `LinhaDouro/v11/config/padrao` → `ok`
3. `LinhaDouro/v11/estacoes/Regua/pronto` → `{"comboio":"R 4100","proxima":"Rede","hora":"09:20"}` → recomenda AVANÇAR
4. `LinhaDouro/v11/estacoes/Rede/pronto` → `{"comboio":"IR 861","proxima":"Regua","hora":"09:20"}` → conflito, decisão e 3 emails no Mailpit
5. `LinhaDouro/v11/estacoes/Regua/partida` → `{"comboio":"R 4100","proxima":"Rede","hora":"09:21"}`
6. `LinhaDouro/v11/estacoes/Rede/chegada` → `{"comboio":"R 4100","anterior":"Regua","hora":"09:35"}` → recomenda AVANÇAR o IR 861

Teste do Subsistema 2: publicar o JSON de exemplo em `LinhaDouro/v7/email/cp_registo`.

---

## Limitações conhecidas

- Passagens intermédias **estimadas**; só as horas terminais vêm do PDF.
- Assume-se que os restantes comboios circulam **a horas**; atrasos reais em curso não entram no cálculo.
- Voltas e rotações das composições não são consideradas.
- Os dois subsistemas ainda **não estão integrados** (prefixos `v11` / `v7`, estações com IDs diferentes, horas em `HH:MM` vs. minutos).
- O Subsistema 2 usa um relógio simulado e 5 troços de exemplo (D01–D05).
- Histórico e template ficam em `/tmp`, perdendo-se se o broker reiniciar.
- Os emails são fictícios (`@exemplo.pt`).

## Possíveis evoluções

- Unificar os dois subsistemas (mesmos IDs, tópicos e formato de hora).
- Confirmação de receção pelo chefe de estação antes de escalar para a CP.
- Alimentar o motor com atrasos reais.
- Persistência fora de `/tmp`.

## Estrutura

```
.
├── Sistema1.lotnb   # Decisão de cruzamentos (FCP11, script Python FixingCPDecisor)
├── Sistema2.lotnb   # Alertas de via única  (FCP10, script Python FixingCPAlertas)
└── README.md
```
