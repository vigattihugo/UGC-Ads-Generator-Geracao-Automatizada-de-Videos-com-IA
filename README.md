# UGC Ads Generator — Geração Automatizada de Vídeos Publicitários com IA

> Pipeline de automação que transforma uma linha de planilha (produto + persona + contexto) em vídeos de anúncio estilo UGC (user-generated content) prontos para publicação, usando múltiplos modelos generativos de vídeo (Veo 3.1, Sora 2) orquestrados por agentes de IA.

<img width="1487" height="1020" alt="image" src="https://github.com/user-attachments/assets/2865e6af-c042-47c3-a669-29b70bc9e03a" />

<img width="1817" height="743" alt="image" src="https://github.com/user-attachments/assets/b484d4e7-515c-4953-a306-a9f28803f664" />


---

## Problema resolvido

Produção de UGC ads tradicionalmente exige contratar criadores de conteúdo, agendar gravações e revisar entregas — um ciclo que leva dias e escala mal quando se precisa testar múltiplas variações de criativo por produto. Este pipeline gera vídeos estilo selfie, com um "influenciador" interagindo com o produto, inteiramente via IA generativa — a partir de uma simples linha preenchida em uma planilha de controle.

---

## Como funciona — 3 estratégias de geração em paralelo

O fluxo lê uma fila de tarefas no Google Sheets e roteia para uma de três estratégias, dependendo do modelo selecionado na planilha:

```
Schedule Trigger → Google Sheets (status = "Ready") → Switch (Model)
                                                            │
        ┌───────────────────────────┬───────────────────────────┬─────────────────────┐
        ▼                           ▼                           ▼
   ESTRATÉGIA A                ESTRATÉGIA B                ESTRATÉGIA C
   Veo direto                 Nano Banana + Veo            Sora 2
        │                           │                           │
Agente de Prompt            Agente de Prompt            Agente de Prompt
(GPT-5-mini)                 de Imagem (GPT-5-mini)       (GPT-5-mini)
        │                           │                           │
        ▼                           ▼                           ▼
  KIE.ai — Veo 3.1          KIE.ai — Nano Banana Edit     KIE.ai — Sora 2
  (imagem → vídeo)           (gera imagem de referência)   (image-to-video)
        │                           │                           │
        │                    GPT-4o Vision analisa            │
        │                    a imagem gerada                   │
        │                           │                           │
        │                    Agente de Prompt de Vídeo          │
        │                    (usa a descrição da imagem)        │
        │                           │                           │
        │                    KIE.ai — Veo 3.1                   │
        │                           │                           │
        └───────────────────────────┴───────────────────────────┘
                                     │
                         Polling assíncrono (taskId)
                                     │
                                     ▼
                    Google Sheets — Status: Finished + URL do vídeo
```

---

## Decisões técnicas

### Três estratégias de geração testáveis em paralelo
O Switch por coluna `Model` permite comparar diretamente o custo, qualidade e consistência visual de três abordagens diferentes para o mesmo produto, sem duplicar a infraestrutura de orquestração — apenas a planilha de controle decide qual caminho seguir.

### KIE.ai como camada de abstração sobre múltiplos modelos generativos
Em vez de integrar SDKs separados para Google Veo, OpenAI Sora e edição de imagem via Gemini, o fluxo usa a API unificada da KIE.ai com um padrão assíncrono consistente: `POST /createTask` retorna um `taskId`, seguido de polling em `/recordInfo` até o status mudar para `success`. Isso centraliza autenticação em uma única credencial e simplifica a manutenção — com o trade-off de depender de um agregador terceiro sobre os modelos reais.

### Prompt engineering com constraints negativos explícitos
Os system prompts dos agentes de geração de prompt de vídeo (Veo e Sora) seguem um padrão de especificação dupla: uma lista positiva de requisitos visuais (enquadramento, iluminação, tom de voz) e uma lista negativa explícita do que jamais deve aparecer (telefone visível, overlays, marcas d'água, subtítulos). Essa técnica reduz artefatos visuais indesejados nos modelos de geração de vídeo, que tendem a "alucinar" elementos não pedidos quando a instrução é apenas positiva.

### Few-shot prompting via exemplo de saída completo
Cada system prompt termina com um "Example Output Prompt" — uma instância completa do formato esperado de saída. Isso ancora o estilo e reduz a variância entre execuções, o que importa diretamente no custo: cada chamada às APIs de vídeo consome créditos, então um prompt mal formado gerado pelo agente se traduz em desperdício financeiro.

### Loop de consistência via Vision (Estratégia B)
Na estratégia "Nano Banana + Veo", após gerar a imagem de referência, o fluxo usa GPT-4o Vision para descrever textualmente o resultado e reinjeta essa descrição no prompt do agente de vídeo. É uma tentativa de manter consistência visual entre a imagem gerada e o vídeo final usando apenas texto como ponte — funcional, mas com teto de precisão.

---

## Stack técnica

| Categoria | Tecnologias |
|---|---|
| Automação / RPA | n8n |
| LLM — Geração de prompt | GPT-5-mini via OpenRouter |
| Visão computacional | GPT-4o Vision (análise de imagem gerada) |
| Orquestração de agentes | LangChain Agent |
| Prompt Engineering | Constraints negativos explícitos, few-shot via exemplo de saída |
| Geração de vídeo | Google Veo 3.1, OpenAI Sora 2 (via KIE.ai) |
| Geração/edição de imagem | Nano Banana — Gemini image edit (via KIE.ai) |
| Agregador de modelos generativos | KIE.ai API |
| Padrão de integração assíncrona | Async polling (createTask + recordInfo) |
| Orquestração de fila | Google Sheets |

---

## Resultados esperados (a validar em produção)

| Métrica | Resultado |
|---|---|
| Estratégias de geração comparáveis lado a lado | 3 (Veo direto, Nano Banana + Veo, Sora 2) |
| Modelos generativos integrados via uma única API | 3 (Veo, Sora, Nano Banana) |
| Intervenção manual entre briefing e vídeo finalizado | 0 — pipeline totalmente automatizado |
| Pontos de controle de status na fila | 1 coluna (`Status`) — rastreável via planilha |
| Custo por vídeo — estratégia mais barata (Sora 2) | ~\$0.15 |
| Custo por vídeo — estratégia premium (Nano Banana + Veo) | ~\$0.32 |

---

## Comparação de custos por estratégia

Uma das vantagens de orquestrar múltiplos modelos no mesmo pipeline é poder comparar custo e qualidade lado a lado, antes de decidir qual caminho usar em escala.

| Estratégia | Composição de custo | Custo por peça de conteúdo | Saída |
|---|---|---|---|
| **Nano Banana + Veo 3 Fast** | Imagem (Nano Banana): \$0.02 · Vídeo 8s (Veo 3 Fast): \$0.30 | **\$0.32** | Imagem still em qualidade de estúdio → transformada em vídeo UGC curto |
| **Sora 2** | Vídeo 10s, geração direta | **\$0.15** | Vídeo gerado diretamente, sem etapa intermediária de imagem |

**Leitura do trade-off:** a estratégia Nano Banana + Veo é aproximadamente **2× mais cara** que o Sora 2 — mas a etapa extra de imagem é exatamente o que viabiliza o loop de consistência via Vision (ver seção de decisões técnicas acima). Você está pagando a diferença por controle adicional sobre a aparência do produto antes de gerar o vídeo, não apenas por "mais qualidade" de forma abstrata.

Isso se traduz em uma recomendação de uso prática, não apenas teórica:

- **Nano Banana + Veo** — preferível para anúncios premium ou campanhas onde a fidelidade visual do produto é crítica (ex: cosméticos, eletrônicos com design distintivo), já que a etapa de imagem permite revisar a composição antes de gastar no vídeo, mais caro.
- **Sora 2** — preferível para iteração rápida, testes A/B em volume, ou campanhas de orçamento mais restrito, onde gerar 5 variações por \$0.75 é mais vantajoso que gerar 2 variações por \$0.64 com a outra estratégia.

Esse tipo de decisão — escolher a estratégia certa por contexto de negócio, não só por capacidade técnica — é o que justifica manter as três rotas no mesmo Switch em vez de comprometer o pipeline com um único modelo.

> Custos de referência (USD), sujeitos a alteração pelos provedores. Validar valores atuais na documentação da KIE.ai antes de projetar custo em escala.

---

## Estrutura do repositório

```
ugc-ads-generator/
├── README.md
├── flow/
│   └── ugc_ads_generator.json     # Fluxo completo importável no n8n
└── docs/
    └── google-sheet-schema.md     # Estrutura de colunas esperada na planilha de controle
```

### Como importar e configurar

1. Importe `flow/ugc_ads_generator.json` no n8n via **Workflows → Import from file**
2. Crie uma cópia do Google Sheet de controle seguindo o schema em `docs/google-sheet-schema.md`
3. Conecte as credenciais necessárias:
   - **Google Sheets** — leitura e atualização da fila de tarefas
   - **OpenRouter** — modelo GPT-5-mini para os agentes de prompt
   - **OpenAI** — GPT-4o Vision para análise de imagem (Estratégia B)
   - **KIE.ai** — chave de API para Veo 3.1, Sora 2 e Nano Banana
4. Popule a planilha com linhas de teste (`Status: Ready`) e ative o Schedule Trigger

---

## Limites conhecidos e próximos passos

**Trade-offs conscientes nesta versão:**

- **Sem consistência de marca via fine-tuning** — a identidade visual do produto/persona depende inteiramente de prompt e imagem de referência. Para campanhas com alto volume do mesmo produto, um **LoRA (Low-Rank Adaptation)** treinado sobre as imagens de referência fixaria a aparência diretamente no modelo gerador, eliminando a dependência de descrição textual via Vision para manter consistência entre múltiplos vídeos. Isso exige infraestrutura de fine-tuning (ex: Replicate, fal.ai) e versionamento de modelo — avaliado como próximo passo quando o volume de campanhas justificar o investimento.
- **Polling com intervalo fixo de 10 segundos** — gera chamadas desnecessárias à API em gerações que demoram minutos. Backoff exponencial (10s → 20s → 40s, com teto em 60s) reduz o número de requisições.
- **Prompts de vídeo Veo parcialmente duplicados** — as duas variantes (com e sem imagem de referência) compartilham ~80% do conteúdo. Candidato a consolidação em um único template parametrizado.
- **Dependência de agregador terceiro (KIE.ai)** — simplifica a integração, mas introduz um ponto único de falha e risco de disponibilidade/pricing fora do controle direto do pipeline.

---
