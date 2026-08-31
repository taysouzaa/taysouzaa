# Cockpit Comercial

CRM que substituiu a operação comercial inteira do Método P4, que rodava em Google Sheets.

**Stack:** React 18 · TypeScript · Supabase (PostgreSQL + RLS + Edge Functions) · n8n
**Estado:** em produção

---

## O problema

A operação comercial vivia em planilhas: captação, distribuição de leads entre closers, agenda, comissões. Planilha não tem dono de escrita — qualquer pessoa com o link altera qualquer célula, e não existe registro de quem mudou o quê. Com o time crescendo, dois problemas ficaram caros: lead aparecendo para o closer errado e comissão calculada sobre dado que alguém editou depois.

## A decisão que define o projeto

**O banco é dono das escritas e das regras de negócio — não o front-end.**

O caminho comum seria checar permissão no React e confiar que o cliente se comporta. Aqui a autorização mora no PostgreSQL:

- **19 tabelas, todas com Row Level Security ativo.** Nenhuma exceção.
- **82 policies** definem quem enxerga qual linha. Um closer não consulta o lead de outro porque o banco não devolve a linha — não porque a tela esconde.
- **63 funções `SECURITY DEFINER`**, todas com `set search_path` fixo. Sem isso, uma função que roda com privilégio elevado pode ser induzida a executar código de um schema controlado pelo chamador — é o vetor clássico de escalada de privilégio em Postgres.
- Escritas sensíveis passam por **RPC**, não por `INSERT` direto do cliente. A tabela de segredos de webhook e a de deduplicação não têm policy nenhuma: só funções acessam.

O custo dessa escolha é real: regra de negócio em SQL é mais trabalhosa de escrever e testar que em TypeScript, e exige disciplina de migration. O ganho é que a regra vale para qualquer cliente que se conecte — app novo, script de manutenção, integração futura.

## Migração sem parar a operação

A planilha não foi desligada no dia da virada. Durante o cutover ela seguiu como **espelho e rede de segurança**: o sistema passou a ser a fonte de verdade, e um fluxo de sincronização mantinha a planilha atualizada. Se algo desse errado, a operação voltava para o que já conhecia sem perder um dia de vendas.

Existe um runbook de cutover documentado para essa janela.

## O que o sistema faz

Captação (via n8n, vindo das landing pages), distribuição e qualificação de leads, agenda de reuniões integrada ao Google Agenda, telefonia, cálculo de comissões e dashboards.

## Números

| | |
|---|---|
| Migrations | 59 |
| Tabelas | 19 — todas com RLS |
| Policies | 82 |
| Funções `SECURITY DEFINER` | 63 — todas com `search_path` fixo |
| Edge Functions | 10 |
| Linhas de código | ~14.800 |

## O que eu faria diferente

A cobertura de teste está no lugar errado. Os testes existentes cobrem cálculo de funil, seleção e senha — lógica de front. A camada que mais importa (policies e funções com privilégio) não tem teste automatizado, e é justamente onde um erro vaza dado entre closers. `pgTAP` rodando no CI resolveria, e é a próxima dívida que eu pagaria.
