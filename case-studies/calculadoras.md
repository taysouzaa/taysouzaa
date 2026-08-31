# Calculadoras de Precificação e ROAS

Ferramenta de precificação para sellers de marketplace: margem de contribuição, ROAS de equilíbrio e histórico sincronizado entre dispositivos.

**Stack:** React · TypeScript · Vite · Tailwind · AWS (DynamoDB, Lambda) · n8n
**Estado:** em produção, em duas versões

---

## O problema

Seller de marketplace erra preço por não considerar o custo completo: comissão do canal, taxa fixa, imposto, frete, custo de anúncio. O resultado é vender com margem negativa achando que está lucrando — e o erro só aparece no fechamento do mês.

A pergunta que a ferramenta responde não é "qual meu lucro", é **"a partir de qual ROAS eu paro de perder dinheiro neste anúncio"**.

## Decisões

### Duas versões do mesmo motor

`P4Calculator` é a ferramenta pública, ligada ao site institucional como porta de entrada. `ClienteCalculator` é a versão de cliente, acessada por link privado.

Compartilham a lógica de cálculo e divergem na superfície. O ganho é que uma correção de fórmula vale para as duas; o custo é que **elas divergem se alguém corrigir só uma** — e foi exatamente o que aconteceu (ver abaixo).

### Histórico em DynamoDB, não em localStorage

O seller calcula no computador e confere no celular. `localStorage` prende o histórico a um navegador; DynamoDB o segue entre dispositivos.

Para um padrão de acesso simples — buscar por usuário, ordenar por recência — DynamoDB custa menos que subir um Postgres e não exige manutenção de instância.

### Cálculo puro, separado da tela

As fórmulas vivem em módulos próprios, fora dos componentes. Regra de precificação é o tipo de código que precisa ser conferido linha a linha por quem entende do negócio, e isso é inviável se ela estiver espalhada em `onChange` de input.

## Um bug que o typecheck pegou

Os dois repositórios rodavam só `vite build`, sem verificação de tipos. Ao rodar `tsc --noEmit` pela primeira vez, apareceu uma chamada a uma função **que nunca havia sido importada** no arquivo.

Em produção isso é `ReferenceError` em dois caminhos: quando o usuário sai do campo de investimento em Ads, e quando carrega um item do histórico.

O build nunca reclamou porque o esbuild só remove os tipos — ele não resolve identificadores, assume que pode ser um global e deixa passar. O erro estava nos **dois** repositórios, porque um derivou do outro.

A correção veio junto com o CI que faltava: typecheck e build a cada push e PR. É a classe de erro que só um verificador de tipos encontra, e o único jeito de garantir que ele rode é a máquina rodar por você.

## Números

| | |
|---|---|
| Linhas de código | ~6.000 por versão |
| Marketplaces com regra própria | 6 |
| CI | typecheck + build, verde |

## O que eu faria diferente

Três componentes concentram 65% do código — o maior tem quase 1.500 linhas. Arquivo desse tamanho é difícil de revisar e praticamente impossível de testar em unidade. Extrair a lógica restante para módulos puros resolveria testabilidade e tamanho ao mesmo tempo, e abriria caminho para testar as fórmulas de precificação, que hoje não têm cobertura.
