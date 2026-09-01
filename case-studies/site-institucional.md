# Site Institucional e Blog

Site do Método P4 com blog, páginas legais e ferramentas gratuitas para sellers.

| | |
|---|---|
| **Stack** | HTML5 · CSS3 · JavaScript ES6+ · PHP · Apache · GitHub Actions |
| **Escala** | 5 páginas comerciais · blog com 14 artigos · 4 páginas legais · 4 ferramentas |
| **Estado** | Em produção, deploy automático |

---

## Decisões

### Sem framework e sem build

Site multipágina estático, HTML/CSS/JS puros. Não há bundler nem etapa de build.

A escolha é deliberada e vem de quem mantém: o conteúdo muda com frequência — artigos, páginas de campanha — e quem edita nem sempre é quem programa. Sem build step, publicar um artigo é editar um arquivo e dar push. Não é instalar dependência, rodar build e descobrir que a versão do Node mudou.

**O custo é ausência de componentização:** cabeçalho e rodapé repetidos em cada página. Para um site desse tamanho, esse custo é menor que a complexidade que um framework traria — e a comparação honesta é essa, não "framework é melhor".

### Proxy same-origin para manter a credencial fora do navegador

Os formulários não falam direto com o serviço de destino. Postam para uma rota do próprio domínio, e é o servidor que encaminha.

```
navegador ──▶ /api/leads (mesmo domínio) ──▶ serviço de destino
                     │
              credencial vive aqui,
              como variável de ambiente
```

Sem isso, a credencial de envio precisaria estar no JavaScript — visível para qualquer visitante que abrisse o DevTools. Não é uma questão de obscuridade: chave em código de cliente **é** chave pública.

Como o site roda em dois ambientes, existem duas implementações do mesmo proxy: uma função serverless e um script PHP no Apache. O contrato é idêntico dos dois lados, o que mantém o front sem saber onde está rodando.

### Publicação automática

Push na `main` dispara o deploy em produção via GitHub Actions, por FTPS. Não há passo manual.

O ambiente de preview atualiza no mesmo push, então dá para conferir o resultado antes de olhar o domínio real — e, mais importante, o deploy deixa de depender de alguém lembrar do procedimento.

### URLs limpas mantendo os arquivos-fonte

As rotas não expõem `.html`, mas os arquivos continuam sendo `.html` no repositório — a reescrita fica na configuração do servidor, espelhada nos dois ambientes.

Isso mantém o repositório navegável e o site com URL apresentável, sem gerador de site estático no meio.

## Escopo

Páginas comerciais, blog com listagem e artigos, quatro páginas legais (privacidade, cookies, termos de uso e proteção de dados), e ferramentas gratuitas que funcionam como porta de entrada — as calculadoras e o SPI, ambos com case study próprio aqui.

O blog tem `sitemap.xml`, `robots.txt` e metadados por artigo. Num site cujo objetivo é captar seller orgânico, indexação não é detalhe: é o produto.

## O que eu faria diferente

**O cabeçalho e o rodapé repetidos** são o ponto que mais dói conforme o número de páginas cresce. Hoje é sustentável; passando de algumas dezenas de páginas, deixa de ser.

Um gerador estático simples paga o próprio custo nesse ponto — mantendo o "sem build" para quem escreve artigo, mas eliminando a repetição estrutural. A decisão certa não é migrar agora, é saber qual é o sinal para migrar.

**As duas implementações do proxy podem divergir.** São o mesmo contrato escrito duas vezes, em linguagens diferentes, e nada verifica que continuam iguais. Um teste que dispare o mesmo payload contra os dois e compare a resposta custaria pouco e fecharia essa lacuna.
