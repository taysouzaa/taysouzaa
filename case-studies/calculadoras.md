# Calculadoras de Precificação e ROAS

Ferramenta de precificação para sellers de marketplace: margem de contribuição, ROAS de equilíbrio e histórico sincronizado entre dispositivos.

| | |
|---|---|
| **Stack** | React · TypeScript · Vite · Tailwind · AWS (DynamoDB, Lambda) · n8n |
| **Escala** | ~6.000 LOC por versão · 6 marketplaces com regra própria · duas versões em produção |
| **Estado** | Em produção · CI com typecheck e build |

---

## O problema

Seller de marketplace erra preço por não considerar o custo completo: comissão do canal, taxa fixa, imposto, frete, custo de anúncio. O resultado é vender com margem negativa achando que está lucrando — e o erro só aparece no fechamento do mês, quando o estoque já girou.

A pergunta que a ferramenta responde não é "qual meu lucro". É:

> **a partir de qual ROAS este anúncio para de dar prejuízo?**

É a diferença entre uma planilha de conferência e uma ferramenta de decisão: o seller usa o número para definir lance de campanha, não para conferir o passado.

## Decisões

### Duas versões do mesmo motor

`P4Calculator` é a ferramenta pública, ligada ao site institucional como porta de entrada. `ClienteCalculator` é a versão de cliente, acessada por link privado.

Compartilham a lógica de cálculo e divergem na superfície.

**O ganho:** uma correção de fórmula vale para as duas.
**O custo:** elas divergem se alguém corrigir só uma — e foi exatamente o que aconteceu, ver abaixo.

### Histórico em DynamoDB, não em localStorage

O seller calcula no computador e confere no celular. `localStorage` prende o histórico a um navegador; DynamoDB o segue entre dispositivos.

Para um padrão de acesso simples — buscar por usuário, ordenar por recência — DynamoDB custa menos que subir um Postgres e não exige manutenção de instância. É o caso em que a escolha "menos poderosa" é a certa: não há junção, não há consulta ad hoc, não há relatório.

### Cálculo puro, separado da tela

As fórmulas vivem em módulos próprios, fora dos componentes.

Regra de precificação é o tipo de código que precisa ser conferido linha a linha por quem entende do negócio — e isso é inviável se ela estiver espalhada em `onChange` de input. Separar também permite que a mesma fórmula alimente a versão pública e a de cliente sem duplicação.

## Um bug que só o typecheck encontrou

Os dois repositórios rodavam apenas `vite build`, sem verificação de tipos. Ao executar `tsc --noEmit` pela primeira vez, apareceu uma chamada a uma função **que nunca havia sido importada** no arquivo.

Em produção isso é `ReferenceError` em dois caminhos:

1. o usuário preenche **"Investimento em Ads"** e sai do campo (`onBlur`);
2. o usuário carrega um item do **histórico**.

O `vite build` nunca reclamou porque o **esbuild só remove os tipos** — ele não resolve identificadores, assume que pode ser um global e deixa passar. É exatamente a classe de erro que um verificador de tipos existe para pegar, e que nenhuma quantidade de build bem-sucedido revela.

Evidência de que era real: o bundle cresceu 61 bytes com a correção — a função não estava lá antes.

O bug existia nos **dois** repositórios, porque um derivou do outro. É o custo previsto da decisão de manter duas versões, cobrado na prática.

A correção veio junto com o CI que faltava: typecheck e build a cada push e PR. O primeiro `tsc` acusou 44 erros num repositório e 6 no outro — a maioria lacuna de configuração (`vite-env.d.ts` e `@types/node` ausentes), que é justamente por que ninguém rodava: a saída era ruído demais para ser útil.

## O que eu faria diferente

**Três componentes concentram 65% do código** — o maior tem quase 1.500 linhas. Arquivo desse tamanho é difícil de revisar e praticamente impossível de testar em unidade.

Extrair a lógica restante para módulos puros resolveria testabilidade e tamanho ao mesmo tempo, e abriria caminho para testar as fórmulas de precificação, que hoje não têm cobertura. Numa ferramenta cujo único produto é o número que ela devolve, essa é a cobertura que mais falta.

**As duas versões deveriam compartilhar o motor como pacote**, não como código copiado. Enquanto forem dois repositórios independentes, toda correção precisa ser aplicada duas vezes — e a próxima vai divergir do mesmo jeito.
