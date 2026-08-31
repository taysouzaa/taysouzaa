# Captação e Qualificação de Leads

Landing pages de campanha e o site institucional alimentando um ponto único de entrada, com deduplicação contra o CRM e qualificação antes do envio.

**Stack:** HTML · CSS · JS · n8n · Google Sheets · Supabase
**Estado:** em produção

---

## O problema

Campanhas pagas e orgânicas para produtos diferentes — tributação, Product Ads, Shopee Ads, diagnóstico, imersão presencial, consultoria. Cada uma com sua página. Se cada página fala com o CRM do seu jeito, aparecem três problemas: o mesmo lead entra várias vezes por campanhas diferentes, ninguém sabe de qual anúncio ele veio, e o time comercial recebe uma fila sem prioridade.

## Decisões

### Ponto único de entrada

Todas as páginas postam no mesmo endpoint. Regra de negócio existe em **um** lugar, não em dez — mudar a definição de lead qualificado é uma alteração, não dez.

O endpoint responde `ok` em cerca de 0,2s e processa depois. A página não fica esperando o CRM, o Sheets e o banco responderem para liberar a tela.

### Qualificação calculada antes do envio

O lead é classificado como MQL ainda no navegador, a partir das respostas de qualificação — no caso do produto de tributação, medalha no marketplace e regime tributário. Quem está fora do perfil entra marcado.

O comercial recebe fila priorizada em vez de lista crua. É a diferença entre "chegaram 40 leads" e "chegaram 40, 12 valem ligação hoje".

### Rastreamento de origem com first-touch

As UTMs são capturadas na primeira visita e persistidas no navegador, não lidas na hora do submit. Lead que chega pelo anúncio, sai para pesquisar e volta direto pelo domínio continua atribuído ao anúncio — que foi o que realmente pagou por ele.

### Fila de reenvio no cliente

Conexão móvel cai no meio do submit. Em vez de perder o lead, o envio fica numa fila no navegador e tenta de novo.

### Deduplicação contra o CRM

Antes de criar, o fluxo verifica se o lead já existe e trata reengajamento como reengajamento — não como lead novo. Isso preserva histórico e evita que a mesma pessoa seja distribuída duas vezes para closers diferentes.

## Uma refatoração que valeu

Dois nós do fluxo carregavam a mesma biblioteca de normalização: **337 das 437 linhas eram cópia literal**. Toda mudança de regra precisava ser feita nos dois lugares, e a primeira vez que alguém esquecesse metade, os dois passariam a discordar em silêncio.

A separação ficou por responsabilidade: um nó virou dono das regras de negócio e passou a emitir a linha pronta; o outro ficou só com o que depende de consultar o CRM. Sobrou uma função em comum, marcada e verificada por script.

O que tornou a refatoração segura foi a verificação: **126 casos num arnês local mais 5 execuções reais de produção, com zero diferença de comportamento**. Refatorar sem esse passo é reescrever e torcer.

## Peculiaridades que valeram a pena

**Fontes e ícones auto-hospedados.** A tipografia variável vai em WOFF2 no repositório (47 KB) e os ícones são um subset gerado (5,5 KB), em vez de CDN. Landing page de tráfego pago vive de tempo de carregamento, e cada requisição a terceiro é uma chance de atraso. O custo é que adicionar um ícone novo exige regerar o subset — está documentado, porque senão vira armadilha para quem mexer depois.

## Números

| | |
|---|---|
| Landing pages | 9, mais 3 páginas de confirmação |
| Fontes integradas ao endpoint | 8 landing pages + site institucional |
| Destinos por lead | 3 |
| Casos no arnês de refatoração | 126, zero diferença |
