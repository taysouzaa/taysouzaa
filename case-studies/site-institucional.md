# Site Institucional e Blog

Site do Método P4 com blog, páginas legais e ferramentas gratuitas para sellers.

**Stack:** HTML · CSS · JS · PHP · Apache · GitHub Actions
**Estado:** em produção

---

## Decisões

### Sem framework e sem build

Site multipágina estático, HTML/CSS/JS puros. Não há bundler nem etapa de build.

A escolha é deliberada: o conteúdo muda com frequência (artigos, páginas de campanha) e quem edita nem sempre é quem programa. Sem build step, publicar um artigo é editar um arquivo e dar push — não é instalar dependência, rodar build e torcer para a versão do Node bater.

O custo é ausência de componentização: cabeçalho repetido em cada página. Para um site desse tamanho, o custo é menor que o da complexidade que um framework traria.

### Proxy same-origin para manter a credencial fora do cliente

Os formulários não falam direto com o serviço de destino. Postam para uma rota do próprio domínio, e é o servidor que encaminha.

Sem isso, a credencial de envio precisaria estar no JavaScript — visível para qualquer visitante que abrir o DevTools. Com o proxy, ela vive como variável de ambiente no servidor e nunca chega ao navegador.

Como o site roda em dois ambientes, existem duas implementações do mesmo proxy: uma função serverless na Vercel e um script PHP no Apache. O contrato é o mesmo dos dois lados.

### Publicação automática

Push na `main` dispara o deploy em produção via GitHub Actions. Não há passo manual de FTP.

O preview em outra plataforma atualiza no mesmo push, então dá para conferir o resultado antes de olhar o domínio real.

### URLs limpas mantendo os arquivos-fonte

As rotas não expõem `.html`, mas os arquivos continuam sendo `.html` no repositório — a reescrita fica na configuração do servidor. Isso mantém o repositório navegável e o site com URL apresentável, sem gerador de site estático no meio.

## Escopo

Páginas comerciais, blog com listagem e artigos, quatro páginas legais (privacidade, cookies, termos, LGPD), e ferramentas gratuitas que servem de porta de entrada — as calculadoras e o SPI.

## O que eu faria diferente

O cabeçalho e o rodapé repetidos em cada página são o ponto que mais dói conforme o número de páginas cresce. Se o site passar de algumas dezenas de páginas, um gerador estático simples paga o próprio custo — mantendo o "sem build" para quem escreve artigo, mas eliminando a repetição estrutural.
