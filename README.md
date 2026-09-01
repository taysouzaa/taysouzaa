## Taynara Souza

`Desenvolvedora Full Stack · Sites, ferramentas, sistemas e automações — da concepção ao deploy em produção.`

Construo sistemas que substituem operação manual: CRM comercial, automação de cobrança recorrente, SaaS de processamento de imagem e páginas de captação de leads.

Trabalho o ciclo inteiro — front-end, API, banco, integração e infraestrutura. Quase tudo o que está abaixo roda em produção hoje, atendendo operação real.

---

### Projetos

A maior parte do que construo vive em repositórios privados de cliente. Abaixo está o trabalho que não aparece na lista de repos — com exceção do SPI, cujo código publiquei em [spi-showcase](https://github.com/taysouzaa/spi-showcase).

Cada projeto tem um **[estudo de caso](case-studies/)**: o problema, as decisões de arquitetura, os trade-offs e o que eu faria diferente. Sem código proprietário nem dado de cliente.

**Método P4** — consultoria especializada em marketplaces:

| Projeto | O que resolve | Stack |
|---|---|---|
| **[Cockpit Comercial](case-studies/cockpit-comercial.md)** | CRM de captação, distribuição e qualificação de leads, com agenda, comissões e dashboards. Substituiu a operação comercial inteira, que rodava em planilhas. | React 18 · TypeScript · Supabase (Postgres + RLS + Edge Functions) · n8n |
| **[SPI](case-studies/spi.md)** — [código público](https://github.com/taysouzaa/spi-showcase) | SaaS que padroniza imagens de produto para 6 marketplaces. O processamento é no navegador via Canvas API — a imagem original nunca sai do cliente. | React · Vite · Node + Express · Prisma · PostgreSQL · AWS (S3, EC2, Amplify) |
| **[Cobrança Recorrente](case-studies/cobranca-recorrente.md)** | Automação de cobrança com entrada e mensalidade vitalícia. Webhook idempotente que confirma o pagamento na fonte antes de criar a assinatura. | n8n · Pagar.me API · Notion API |
| **[Captação de Leads](case-studies/captacao-de-leads.md)** | Nove landing pages num ponto único de entrada: tracking de UTM first-touch, fila de reenvio no navegador, cálculo de MQL antes do envio e deduplicação contra o CRM. | HTML · CSS · JS · PHP · n8n |
| **[Site institucional + blog](case-studies/site-institucional.md)** | Site com blog, ferramentas gratuitas para sellers e captação de leads via proxy same-origin, para manter credencial fora do cliente. | HTML · CSS · JS · PHP · Apache |
| **[Calculadoras de precificação](case-studies/calculadoras.md)** | Simulação de margem de contribuição e ROAS de equilíbrio por marketplace, com histórico sincronizado entre dispositivos. | React · TypeScript · AWS DynamoDB |

### Verificação

Os sistemas acima têm CI: o **Cockpit** roda 121 asserções em pgTAP contra as policies reais a cada push; **SPI** e **calculadoras** rodam typecheck, testes e build. As fórmulas de precificação são testadas sobre um exemplo conferível na mão.

---

### Stacks

[![My Skills](https://skillicons.dev/icons?i=html,css,javascript,typescript,react,nodejs,express,vite,tailwind,python,postgres,prisma,supabase,aws,gcp,docker,git,github,figma,vscode)](https://skillicons.dev)

Também trabalho com **n8n** (automação de fluxos), **Pagar.me** e **Notion API** — sem ícone na régua acima.

---

### Idiomas

Português (nativo) · Espanhol (avançado)

---

### Contato

Aberta a projetos de desenvolvimento, automação e integrações.
Se quiser conversar sobre uma oportunidade ou parceria:

- **Email:** [souza.codes@gmail.com](mailto:souza.codes@gmail.com)
- **LinkedIn:** [linkedin.com/in/taynara-correia-souza](https://www.linkedin.com/in/taynara-correia-souza/)
- **Portfólio:** [taynarasouza.vercel.app](https://taynarasouza.vercel.app)
