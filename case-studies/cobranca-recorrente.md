# Cobrança Recorrente

Automação que transforma uma venda fechada pelo closer em assinatura recorrente ativa, com registro no painel de CS.

| | |
|---|---|
| **Stack** | n8n · Pagar.me · Notion API · front estático + função serverless (Vercel) |
| **Escala** | 40 nós em produção · workflow de alerta dedicado · segundo gateway em construção |
| **Estado** | Em produção |

---

## O problema

O modelo comercial tem duas partes: uma **entrada de fidelidade**, parcelável, cobrada no ato; e uma **mensalidade vitalícia** que começa depois.

Fazer isso à mão significa gerar link de pagamento, esperar o cliente pagar, criar a assinatura no gateway, anotar no painel e lembrar de atualizar a data da próxima cobrança. Cada passo é um lugar onde alguém esquece — e o que se esquece é receita recorrente que nunca começa.

## Arquitetura

```
Closer ─▶ formulário estático ─▶ função serverless ─▶ cria link (gateway)
                                                            │
                                          cliente paga ◀────┘
                                                │
                                          webhook ─▶ confirma na fonte
                                                     │
                                                     ├─ cria assinatura
                                                     ├─ grava no painel
                                                     └─ agenda próxima cobrança
```

A separação é deliberada: **o formulário só gera o link**. Criar assinatura, confirmar pagamento e gravar é responsabilidade da automação, disparada pelo gateway. Uma falha no navegador do closer não deixa cobrança pela metade, e a credencial do gateway vive na função serverless — nunca no cliente.

## Três decisões que vieram de erro real

### 1. Confirmar o pagamento na fonte, não confiar no webhook

O webhook diz que foi pago. Isso não é prova suficiente: payload pode ser reenviado, forjado ou chegar fora de ordem.

Antes de criar a assinatura, o fluxo consulta o pedido direto na API do gateway e só segue se o status confirmar lá. O webhook virou **gatilho**, não fonte de verdade.

O fluxo também é **idempotente**: o mesmo evento entregue duas vezes não cria duas assinaturas. Gateway reenvia por design — reenvio é o caso normal, não a exceção.

### 2. Erro engolido é pior que erro barulhento

Durante 15 dias, nada foi gravado no painel. As execuções apareciam como **"Succeeded"**.

A causa imediata era pequena: o código enviava uma propriedade com o nome antigo, e o Notion rejeita a **página inteira** quando recebe propriedade inexistente — não grava parcialmente, devolve `400`.

A causa real era estrutural: **todos os nós estavam configurados para continuar em caso de erro**. A falha era engolida e a execução terminava verde.

A correção nos nós críticos foi passar a **interromper o fluxo**. Duas coisas mudam com isso:

- a falha aparece vermelha na lista de execuções;
- sem o `200 OK`, o gateway **reenvia o evento** — a política de retry dele passa a trabalhar a favor.

O erro deixou de ser perda silenciosa e virou retentativa automática.

> `continueOnError` é conveniente no desenvolvimento e perigoso em produção. Ele não elimina a falha, elimina o **sinal** dela.

### 3. Verificar a premissa antes de projetar em cima dela

Na versão para um segundo gateway, o desenho inicial criava a assinatura primeiro e a cancelava se a entrada não fosse paga — assumindo que criar assinatura era passo gratuito e reversível.

Um spike controlado mostrou que **a assinatura cobra na criação, valor cheio, em segundos**. A ordem foi invertida — entrada primeiro, assinatura depois — e a chamada de cancelamento deixou de existir.

Custo do spike: algumas horas. Custo de descobrir em produção: cobrança indevida no cartão de um cliente real, e a conversa que vem depois.

O mesmo trabalho revelou outra restrição do gateway: o token de cartão é **de uso único**, e o modelo precisa de duas cobranças no mesmo cartão. Isso não era contornável com esforço — mudava o desenho.

## Dado morto: o defeito que não dá erro

Dois campos — o closer responsável e as observações da venda — eram coletados pelo formulário e nunca chegavam ao painel.

O bloco que deveria lê-los procurava nomes de campo que o formulário nunca enviou. Estava morto desde sempre, sem erro nenhum, porque **procurar chave inexistente devolve vazio em silêncio**.

Apareceu nos dois gateways, com quatro dias entre as correções. Erro de contrato entre duas pontas escritas em momentos diferentes não dá exceção — dá campo em branco que ninguém associa a bug.

## Como o workflow é versionado

O fluxo do segundo gateway não é editado na interface: é **gerado por código**. Um script Python monta o JSON dos nós e conexões, e há uma suíte que valida o grafo resultante — que todo nó é alcançável, que as conexões existem, que não há ramo órfão.

Editar o JSON à mão é desfeito na próxima geração, e isso está documentado no README.

O ganho é revisão: diff de workflow gerado é legível, diff de JSON arrastado na tela não é.

## O que eu faria diferente

**Contrato explícito entre o formulário e a automação, validado na borda.** Se o payload que chega fosse verificado contra um schema, os campos com nome trocado teriam falhado alto na primeira execução, em vez de sumirem em silêncio por meses.

É a mesma lição da seção 2, aplicada uma camada antes: o problema não é o erro acontecer, é o sistema não ter onde falhar.
