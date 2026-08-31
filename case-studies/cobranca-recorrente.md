# Cobrança Recorrente

Automação que transforma uma venda fechada pelo closer em assinatura recorrente ativa, com registro no painel de CS.

**Stack:** n8n · Pagar.me · Notion · front estático + função serverless (Vercel)
**Estado:** em produção

---

## O problema

O modelo comercial tem duas partes: uma **entrada** de fidelidade, parcelável, cobrada hoje; e uma **mensalidade vitalícia** que começa depois. Fazer isso à mão significa gerar link de pagamento, esperar o cliente pagar, criar a assinatura no gateway, anotar no painel e lembrar de atualizar a data da próxima cobrança. Cada passo é um lugar onde alguém esquece.

## Arquitetura

O closer preenche um formulário estático que chama uma função serverless; ela cria o link de pagamento no gateway com os termos da recorrência no `metadata`. Quando o cliente paga, o gateway dispara um webhook e a automação assume: confirma o pagamento, cria a assinatura, grava no painel e agenda a próxima cobrança.

A separação é deliberada — o formulário **só gera o link**. Criar assinatura, confirmar pagamento e gravar é responsabilidade da automação. Assim uma falha no navegador do closer não deixa cobrança pela metade.

## Três decisões que vieram de erro real

### 1. Confirmar o pagamento na fonte, não confiar no webhook

O webhook diz que foi pago. Isso não é prova suficiente: payload pode ser reenviado, forjado ou chegar fora de ordem. Antes de criar a assinatura, o fluxo consulta o pedido direto na API do gateway e só segue se o status confirmar lá.

O webhook também é **idempotente** — o mesmo evento entregue duas vezes não cria duas assinaturas. Gateway reenvia por design, e reenvio é o caso normal, não a exceção.

### 2. Erro engolido é pior que erro barulhento

Durante 15 dias, nada foi gravado no painel. As execuções apareciam como **"Succeeded"**.

A causa era pequena: o código enviava uma propriedade com o nome antigo, e o Notion rejeita a **página inteira** quando recebe propriedade inexistente — não grava parcialmente, devolve 400. Mas o motivo de ninguém ter percebido era estrutural: todos os nós estavam configurados para continuar em caso de erro. A falha era engolida e a execução terminava verde.

A correção nos nós críticos foi passar a **interromper o fluxo**. Duas coisas mudam com isso: a falha aparece vermelha na lista, e — mais importante — sem o `200 OK` o gateway reenvia o evento. O erro deixa de ser perda silenciosa e vira retentativa.

A lição que ficou: `continueOnError` é conveniente no desenvolvimento e perigoso em produção. Ele não elimina a falha, elimina o **sinal** dela.

### 3. Verificar a premissa antes de projetar em cima dela

Na versão para um segundo gateway, o desenho inicial criava a assinatura primeiro e a cancelava se a entrada não fosse paga — assumindo que criar assinatura era passo gratuito e reversível.

Um spike controlado mostrou que **a assinatura cobra na criação, valor cheio, em segundos**. A ordem foi invertida: entrada primeiro, assinatura depois. A chamada de cancelamento deixou de existir.

Custo do spike: algumas horas. Custo de descobrir isso em produção: cobrança indevida no cartão de cliente real.

## Dado morto

Dois campos — o closer responsável e as observações da venda — eram coletados pelo formulário e nunca chegavam ao painel. O bloco que deveria lê-los procurava nomes de campo que o formulário nunca enviou; estava morto desde sempre, sem erro nenhum, porque procurar chave inexistente devolve vazio em silêncio.

Aparece nos dois gateways, com quatro dias de diferença entre as correções. Erro de contrato entre duas pontas escritas em momentos diferentes não dá exceção — dá campo em branco que ninguém associa a bug.

## O que eu faria diferente

Contrato explícito entre o formulário e a automação, validado na borda. Se o payload que chega fosse verificado contra um schema, os campos com nome trocado teriam falhado alto na primeira execução, em vez de sumirem em silêncio por meses.
