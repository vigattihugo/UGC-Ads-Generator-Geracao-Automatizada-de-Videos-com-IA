# Schema do Google Sheet de Controle

A planilha funciona como fila de tarefas. Cada linha representa um vídeo a ser gerado.

| Coluna | Tipo | Descrição |
|---|---|---|
| `No` | string/number | Identificador único da linha — usado para localizar e atualizar o registro após a geração |
| `Status` | string | `Ready` (pronto para processar) → `Finished` (vídeo gerado) |
| `Model` | string | Estratégia de geração: `Veo 3.1`, `Nano + Veo 3.1` ou `Sora 2` |
| `Product` | string | Nome do produto a ser destacado no vídeo |
| `ICP` | string | Perfil do cliente ideal (idade, estilo, contexto) usado para definir a persona do "influenciador" |
| `Product Features` | string | Principais características do produto a serem mencionadas no diálogo |
| `Video Setting` | string | Cenário/ambiente desejado para o vídeo (ex: cozinha, academia, carro) |
| `Product Photo` | url | URL pública da foto do produto, usada como imagem de referência |
| `Finished Video` | url | Preenchido automaticamente pelo fluxo com a URL do vídeo gerado |

---

## Fluxo de status

```
Ready  →  (processamento pelo n8n)  →  Finished
```

Linhas com `Status: Ready` são capturadas pelo Schedule Trigger. Após a geração ser concluída com sucesso, o fluxo atualiza a linha para `Status: Finished` e preenche `Finished Video` com a URL do resultado.

## Dica de uso

Para testar as três estratégias com o mesmo produto, duplique a linha três vezes alterando apenas a coluna `Model` — isso permite comparar custo, tempo de geração e qualidade visual lado a lado.
