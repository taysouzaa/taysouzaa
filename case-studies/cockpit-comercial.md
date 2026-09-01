# Cockpit Comercial

CRM que substituiu a operação comercial do Método P4, que rodava em Google Sheets.

| | |
|---|---|
| **Stack** | React 18 · TypeScript · Vite · TanStack Query · Supabase (PostgreSQL + RLS + Edge Functions) · n8n |
| **Escala** | 60 migrations · 19 tabelas · 82 policies · 63 funções `SECURITY DEFINER` · 10 Edge Functions · ~14.800 LOC |
| **Estado** | Em produção |

---

## O problema

A operação comercial vivia em planilhas: captação, distribuição de leads entre closers, agenda, comissões.

Planilha não tem dono de escrita. Qualquer pessoa com o link altera qualquer célula, não há registro de quem mudou o quê, e não existe forma de expressar "este closer vê estes leads". Com o time crescendo, dois problemas ficaram caros: lead aparecendo para o closer errado, e comissão calculada sobre um número que alguém editou depois.

## A decisão que define o projeto

**A autorização mora no banco, não no front-end.**

O caminho comum é checar permissão no React e confiar que o cliente se comporta. O problema é que essa checagem vale para exatamente um cliente: a tela. Um script de manutenção, uma integração futura ou um token vazado falam direto com a API e não passam por ela.

Aqui a regra é do PostgreSQL:

```
Cliente ──▶ PostgREST ──┬──▶ RLS decide quais linhas existem
                        └──▶ RPC decide o que pode ser escrito
```

- **19 tabelas, todas com Row Level Security ativo.** Sem exceção.
- **82 policies** definem quem enxerga qual linha. Um closer não consulta o lead de outro porque **o banco não devolve a linha** — não porque a tela esconde o botão.
- **63 funções `SECURITY DEFINER`**, todas com `set search_path` fixo. Sem isso, uma função que roda com privilégio elevado pode ser induzida a executar código de um schema controlado por quem chama — o vetor clássico de escalada de privilégio em Postgres, e o mais fácil de esquecer.
- Escritas sensíveis passam por **RPC**, não por `INSERT` direto do cliente. A tabela de segredos de webhook e a de deduplicação **não têm policy nenhuma**: só funções acessam. Ausência de policy com RLS ligado é negação total — mais seguro que uma policy permissiva escrita com pressa.

**O custo:** regra de negócio em SQL é mais trabalhosa de escrever e testar que em TypeScript, e exige disciplina de migration. Não é a escolha certa para todo projeto.

**O ganho:** a regra vale para qualquer cliente que se conecte, hoje e daqui a dois anos.

## O que a autorização precisa garantir

Cada linha abaixo é um cenário que roda a cada push:

| Regra | Por quê |
|---|---|
| Colaborador não lê lead de outro | Isolamento entre closers é a premissa do modelo comercial |
| Colaborador não exclui lead | Exclusão é irreversível e não há caso de uso legítimo |
| Observação é append-only | O log de tratativas só tem valor probatório se ninguém reescreve |
| `autor_id` não é forjável, **nem pelo master** | O campo diz quem atendeu; falsificável, não serve para apurar nada |
| Qualificação o dono **edita** | Contraste proposital: o questionário é corrigido durante a call |
| Colaborador não vira faixa preta | Guarda anti-escalonamento de privilégio |
| Colaborador não escreve em `crm_config` | O de-para de funil redireciona todo lead que entra no sistema |
| Desativado perde acesso na hora | Ver abaixo |
| Diretório continua visível | Contraprova — ver abaixo |

## Um bug que virou método

"Desativar" no cadastro marcava `active = false` — e a pessoa continuava lendo e editando os leads dela.

`profiles.active` é coluna da aplicação. O serviço de autenticação não a conhece, então o refresh token seguia válido indefinidamente. Desativar parecia funcionar e não fazia nada.

A correção foi levar `active` para dentro das policies. O que ficou como método é mais interessante que a correção: **para cada regra que nega, existe uma contraprova que exige**. O mesmo conjunto que verifica "desativado não lê os próprios leads" verifica "o diretório continua visível para todos" — porque a forma mais fácil de fazer o primeiro teste passar é quebrar a tela inteira, e um teste que só sabe negar não percebe isso.

## Migração sem parar a operação

A planilha não foi desligada no dia da virada. Durante o cutover ela seguiu como **espelho e rede de segurança**: o sistema virou a fonte de verdade e um fluxo de sincronização mantinha a planilha atualizada.

Se algo desse errado, a operação voltava para o que já conhecia sem perder um dia de vendas. Existe runbook documentado para essa janela.

## Estado da verificação

A suíte de autorização roda em CI a cada push e pull request: sobe um PostgreSQL limpo, aplica as 60 migrations e o seed, e executa **37 asserções em pgTAP** contra as policies reais — não contra mocks.

Antes disso os cenários já existiam, mas como scripts que imprimiam `PASS`/`FAIL` numa tabela para leitura humana. Um `FAIL` não mudava o código de saída do psql, então não travava nada: o teste protegia quem lembrasse de rodar *e* de olhar o resultado. A conversão trocou isso por "o merge não passa".

Duas coisas apareceram quando as migrations rodaram contra um banco vazio pela primeira vez:

- **A cadeia não aplicava do zero.** Quatro migrations validavam uma configuração contra uma lista que só era populada nove dias depois na ordem de aplicação. Em produção nunca apareceu — a lista já existia quando elas rodaram. O defeito só atingia ambiente novo, o que significa que **não dava para reconstruir o banco a partir do repositório**: nem CI, nem staging, nem restauro de backup.
- **O seed não era idempotente**, apesar de se declarar idempotente no cabeçalho: inseria comissões que um trigger já havia criado, violando o índice único.

Nenhum dos dois afetava produção. Ambos afetavam a capacidade de *recriar* produção — que é o que se descobre no pior momento possível.

## O que eu faria diferente

**A cobertura ainda está desequilibrada.** Os testes de front cobrem cálculo de funil, seleção e senha; a suíte pgTAP cobre as policies. As RPCs de ingestão e distribuição — que decidem para quem cada lead vai — ainda dependem de verificação manual. É onde um erro é silencioso: o lead entra, ninguém vê erro, e o rodízio fica torto por semanas.

**Componentes grandes.** A ficha do lead tem 644 linhas e concentra formulário, histórico e ações. Extrair a lógica para hooks tornaria testável o que hoje só é verificável clicando.
