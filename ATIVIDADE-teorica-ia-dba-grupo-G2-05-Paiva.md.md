# Atividade Teórica: Usuários Especialistas, IA e Distribuição Segura de Dados

**Aluno(s):** Annalice Cezar Leal, Ramon Nascimento, Renan Nunes de Mello Paiva, Luan Vinicius Saraiva de Almeida
**Turma:** Banco de Dados 2026
**Data:** 17/08/2026
**Repositório Git:** https://github.com/RenanPaiva140/ATIVIDADE-teorica-ia-dba-G2-05-Paiva.md

## Resumo Executivo

Este trabalho analisa os riscos e as estratégias de mitigação envolvidos no uso de ferramentas de Inteligência Artificial generativa por usuários especialistas que acessam diretamente um SGBD PostgreSQL, sem a mediação de programadores de aplicação. O cenário evidencia um deslocamento do controle técnico tradicional: consultas que antes passavam por revisão de código agora são geradas e executadas quase instantaneamente por chatbots e assistentes de IA, criando riscos de exposição de dados sensíveis, degradação de performance e vazamento de informações para serviços externos. A posição do grupo é de que a distribuição segura de dados nesse cenário não deve depender da confiança nos prompts ou nas respostas da IA, mas de controles estruturais no próprio banco de dados — aplicados pelo DBA através do princípio do menor privilégio, views restritivas, roles customizadas, limitação de recursos de execução e auditoria contínua. A IA é tratada como um usuário não confiável por padrão, cujo acesso é sempre mediado por uma camada de segurança definida e mantida pelo DBA, e não pelo bom senso de quem escreve o prompt.

## 1. Desenvolvimento Teórico

### 1.1 O que é o DBA e quais suas funções?

O Database Administrator (DBA) é o profissional (ou papel organizacional) responsável pela administração central de um Sistema de Gerenciamento de Banco de Dados (SGBD), garantindo que os dados estejam disponíveis, íntegros, seguros e com performance adequada para todos os usuários e aplicações que dependem deles. Diferente do usuário comum, o DBA possui privilégios amplos (superuser ou equivalente) e responde pela infraestrutura de dados como um todo, não por uma aplicação específica.

Suas funções principais, aplicadas ao cenário proposto, são:

- **Definição do esquema (schema definition):** o DBA projeta e mantém a estrutura lógica do banco — tabelas, tipos de dados, relacionamentos, chaves primárias e estrangeiras. No cenário de IA, isso é crítico porque o esquema determina o que a IA "enxerga" ao gerar SQL: se uma tabela `clientes` expõe diretamente colunas como `cpf` ou `renda_mensal`, qualquer consulta gerada, mesmo sem intenção maliciosa, pode retornar dados sensíveis. Um esquema bem desenhado, com views intermediárias, reduz essa superfície de exposição antes mesmo de qualquer controle de acesso ser aplicado.

- **Definição da estrutura de armazenamento e método de acesso:** envolve decisões sobre índices, particionamento de tabelas e planos de execução. Isso está diretamente ligado ao risco de consultas pesadas geradas por IA: um usuário especialista pedindo a uma IA "todas as vendas cruzadas com todos os clientes dos últimos 5 anos" pode gerar um `JOIN` sem filtros adequados. Índices bem planejados e limites de estrutura ajudam a conter o impacto dessas consultas na performance geral do sistema.

- **Autorização de acesso aos dados (concessão de privilégios):** é a função mais diretamente relacionada ao cenário, pois define quem pode ler, escrever ou modificar quais dados. No contexto de IA, essa autorização precisa ser pensada não em termos de "o usuário X pode acessar a tabela Y", mas em termos mais granulares — quais colunas, quais linhas, com que frequência e sob qual custo computacional máximo. É o DBA quem concede (`GRANT`) e revoga (`REVOKE`) esses privilégios de forma centralizada.

- **Especificação de regras de integridade:** o DBA define restrições (`CHECK`, `NOT NULL`, chaves estrangeiras, triggers) que protegem a consistência dos dados mesmo diante de operações inesperadas. Se uma IA gerar uma consulta de atualização (`UPDATE`) mal formulada — por exemplo, sem cláusula `WHERE` — essas regras de integridade, combinadas a permissões restritas de escrita, funcionam como uma segunda camada de proteção contra corrupção de dados.

Em resumo, cada uma dessas funções tradicionais do DBA ganha uma camada extra de importância no cenário de IA generativa: elas deixam de ser apenas boas práticas de modelagem e passam a ser a principal barreira técnica entre uma consulta gerada automaticamente e um incidente de segurança ou de integridade.

### 1.2 Perfis de usuários de banco de dados

Os SGBDs relacionais costumam categorizar os usuários finais em quatro perfis principais, cada um com um nível diferente de conhecimento técnico e forma de interação com o banco:

- **Programadores de aplicações (application programmers):** desenvolvedores que escrevem código de aplicação (em linguagens como Python, Java, etc.) que embute comandos SQL, geralmente por meio de APIs, ORMs ou drivers de acesso. Sua interação com o banco é indireta e mediada por camadas de aplicação, testes e revisão de código.

- **Usuários sofisticados (sophisticated users):** interagem diretamente com o SGBD, sem escrever programas de aplicação, mas utilizando linguagens de consulta (SQL) ou ferramentas de BI para submeter suas próprias consultas, geralmente ad hoc, para análise de dados.

- **Usuários especialistas (specialized users):** também escrevem aplicações de banco de dados especializadas, mas voltadas a necessidades específicas — por exemplo, sistemas de apoio à decisão, ferramentas analíticas avançadas ou, no cenário proposto, assistentes de IA que traduzem linguagem natural em SQL. É esse perfil que combina acesso direto e privilegiado ao banco com o uso de ferramentas automatizadas que fogem do fluxo tradicional de revisão.

- **Usuários navegantes (casual/naïve users):** interagem com o banco por meio de interfaces já prontas (formulários, painéis, aplicações finais), sem qualquer conhecimento de SQL ou da estrutura interna dos dados. Seu acesso é totalmente mediado pela aplicação.

O usuário especialista é o foco de atenção deste cenário porque ele reúne duas características que, isoladamente, já demandam cuidado, mas que juntas multiplicam o risco: (1) possui privilégios de acesso direto ao banco, superiores aos de um usuário navegante, e (2) delega a geração da consulta a uma ferramenta de IA que ele não controla totalmente e cujo comportamento pode ser imprevisível. Diferente do programador de aplicação, cujo código passa por versionamento, revisão e testes antes de ir a produção, o usuário especialista com IA pode gerar e executar uma consulta em segundos, sem qualquer revisão humana adicional — e sem que a IA "saiba" quais dados são sensíveis ou qual o custo computacional real daquela consulta no ambiente de produção.

### 1.3 Riscos do uso de IA por usuários especialistas

**1. Consultas com erros de lógica ou sintaxe.** Modelos generativos podem produzir SQL sintaticamente válido, mas semanticamente incorreto — por exemplo, um `JOIN` que duplica linhas por não considerar a cardinalidade correta entre tabelas, inflando artificialmente valores de vendas em um relatório. O impacto é a tomada de decisões de negócio baseadas em números errados, sem que ninguém perceba, já que a consulta "roda sem erro".

**2. Exposição de dados sensíveis em respostas.** Uma IA instruída a "listar os clientes com maior inadimplência" pode, sem qualquer intenção maliciosa, incluir CPF, endereço completo e telefone no resultado, simplesmente porque essas colunas estavam disponíveis na tabela de origem. O impacto direto é a violação de princípios de minimização de dados exigidos pela LGPD, além do risco de esses dados pessoais serem posteriormente compartilhados, copiados ou colados em outros sistemas sem controle.

**3. Consultas pesadas que degradam a performance do banco.** Um usuário especialista sem conhecimento aprofundado de otimização pode pedir à IA uma consulta analítica complexa (agregações sobre milhões de linhas, subconsultas aninhadas, ausência de índices adequados) que consome grande parte dos recursos do servidor. O impacto é a degradação do banco para todos os outros usuários e sistemas em produção — um problema de disponibilidade que afeta toda a organização, não apenas quem executou a consulta.

**4. Vazamento de dados por meio de prompts enviados a ferramentas de IA externas.** Ao pedir para uma ferramenta de IA generativa "analisar" um conjunto de dados, é comum que o usuário cole diretamente resultados de consultas (contendo dados pessoais ou comerciais sensíveis) no prompt. Se a ferramenta for um serviço externo, sem contrato de proteção de dados adequado, esses dados podem ser armazenados, usados para treinar modelos de terceiros ou expostos em incidentes de segurança fora do controle da empresa. O impacto é uma violação de confidencialidade que ocorre fora do perímetro do banco de dados, tornando-se muito mais difícil de detectar e de remediar.

**5. Escalada de privilégios e uso indevido de informações (risco adicional citado no enunciado).** Um usuário especialista pode, deliberadamente ou por engano, pedir à IA para gerar consultas que combinam dados de diferentes domínios (financeiro, RH, comercial) aos quais ele tem acesso técnico, mas não autorização de negócio para cruzar — criando relatórios que revelam informações que, isoladamente, seriam inofensivas, mas que juntas configuram uso indevido (por exemplo, cruzar dados salariais com desempenho individual sem finalidade legítima).

### 1.4 Distribuição segura de dados

- **Princípio do menor privilégio (least privilege):** cada usuário ou role deve ter apenas os privilégios estritamente necessários para sua função, nunca acesso amplo "por conveniência". No PostgreSQL, isso significa conceder `SELECT` apenas nas colunas e tabelas necessárias, e nunca conceder `ALL PRIVILEGES` a usuários especialistas. Isso é importante porque reduz a superfície de dano: mesmo que a IA gere uma consulta abrangente demais, ela simplesmente falhará por falta de permissão, em vez de retornar dados que não deveriam ser acessados.

- **Uso de views para limitar colunas e linhas:** em vez de conceder acesso direto às tabelas-base, o DBA cria views que já filtram colunas sensíveis (como CPF ou dados salariais) e podem restringir linhas por meio de `WHERE` (ex.: apenas dados da região ou período sob responsabilidade do usuário). A IA e o usuário especialista consultam apenas a view, nunca a tabela original — isso é essencial porque desloca a decisão sobre "o que é sensível" do prompt (não confiável) para o esquema do banco (confiável e auditável).

- **Criação de roles customizadas por perfil:** em vez de gerenciar permissões usuário a usuário, o DBA cria roles (ex.: `role_analista_vendas`, `role_especialista_ia`) com um conjunto predefinido de privilégios, e associa os usuários a essas roles. Isso é importante porque padroniza o controle de acesso, facilita auditorias e evita o acúmulo desordenado de permissões individuais ao longo do tempo (o chamado "privilege creep").

- **Controle de tempo de execução e concorrência de consultas:** o PostgreSQL permite configurar parâmetros como `statement_timeout` (tempo máximo de execução de uma consulta) e limites de conexões simultâneas por role, além do uso de filas de execução (ex.: via `pg_hba.conf`, connection pooling ou extensões como `pg_cron`/`pgbouncer`). Isso é importante porque impede que uma única consulta mal gerada por IA monopolize os recursos do servidor e afete a disponibilidade para os demais usuários.

- **Auditoria e rastreamento de operações por usuário (logs):** todo acesso, especialmente os realizados por meio de ferramentas de IA, deve ser registrado — quem executou, qual consulta, quando e quais dados foram retornados. No PostgreSQL, isso pode ser feito com extensões como `pgAudit` ou por meio de logging nativo configurado (`log_statement`, `log_min_duration_statement`). Isso é essencial porque, mesmo com controles preventivos, é preciso ter capacidade de investigação posterior (forense) caso um incidente ocorra.

- **Conformidade com a LGPD:** a Lei Geral de Proteção de Dados exige minimização de dados (coletar e expor apenas o necessário), finalidade específica de uso, e a possibilidade de rastrear quem acessou dados pessoais e por quê. As estratégias acima (views, roles, auditoria) não são apenas boas práticas técnicas — são, na prática, os mecanismos que permitem à organização demonstrar conformidade legal em caso de fiscalização ou incidente, evidenciando controles técnicos e administrativos adequados (art. 46 da LGPD).

### 1.5 Atuação do DBA no cenário de IA

O DBA, nesse cenário, deixa de atuar apenas como um administrador de infraestrutura e passa a ser um curador ativo da relação entre usuários especialistas e IA generativa. Suas responsabilidades incluem:

- **Definir e manter esquemas seguros por padrão**, projetando desde o início views e camadas de abstração que já ocultam dados sensíveis das tabelas-base, para que nenhuma consulta gerada por IA tenha acesso direto a colunas críticas.
- **Redesenhar políticas de acesso continuamente**, revisando periodicamente roles e permissões à medida que novos usuários especialistas ou novos casos de uso de IA surgem, evitando o acúmulo de privilégios desnecessários.
- **Monitorar consultas geradas por IA**, utilizando ferramentas de observabilidade do PostgreSQL (`pg_stat_statements`, logs de auditoria) para identificar padrões de consultas abusivas, lentas ou fora do comportamento esperado, e agir proativamente antes que causem incidentes.
- **Orientar o uso responsável da IA**, atuando como referência técnica para os usuários especialistas — definindo boas práticas de prompt (ex.: nunca colar dados reais de clientes em ferramentas externas) e educando sobre os riscos de exposição de dados.
- **Manter backups, índices e performance geral**, garantindo que, mesmo diante de um crescimento no volume de consultas ad hoc geradas por IA, o sistema continue íntegro, recuperável e responsivo.

Em suma, o DBA atua como a "auditoria humana" (ou de processo) que falta ao fluxo de geração automática de SQL: assim como o enunciado da atividade pede que o próprio grupo revise e corrija as respostas de IA usadas no trabalho, o DBA é quem audita, corrige e limita, no nível estrutural do banco, o que a IA pode fazer em nome dos usuários especialistas.

### 1.6 Análise crítica: qual a melhor abordagem?

A posição do grupo é a de que a distribuição segura de dados em ambientes com uso de IA generativa deve partir de uma premissa simples: **a IA deve ser tratada como um usuário não confiável, mesmo quando quem a opera é um especialista de confiança.** Isso significa que a segurança não pode depender de instruções dadas à IA ("não exponha dados sensíveis") nem do bom senso do usuário ao formular o prompt, pois ambos são falhos e não auditáveis de forma consistente. A segurança precisa estar embutida na estrutura do banco de dados — em views, roles, limites de execução e auditoria — de modo que, independentemente do que a IA gere como SQL, o próprio SGBD imponha os limites do que pode ser lido, alterado ou retornado.

Essa abordagem é justificada por analogia com um princípio já consolidado em segurança da informação: não se confia na entrada do usuário (input) em uma aplicação web, e sim se validam e restringem as ações no lado do servidor. Da mesma forma, não se deve confiar no prompt ou na saída da IA, mas sim restringir, no lado do banco de dados, o que é fisicamente possível de ser consultado ou modificado. Um exemplo concreto sustenta essa conclusão: mesmo que um usuário peça à IA "mostre todos os dados de todos os clientes, incluindo CPF", se o acesso dele estiver restrito a uma view sem essa coluna, a consulta gerada simplesmente não terá como retornar aquele dado — o controle é estrutural, não comportamental.

## 2. Exemplos e Casos

### 2.1 Exemplo de view restritiva no PostgreSQL

Suponha uma tabela `clientes` com colunas sensíveis. O DBA cria uma view que expõe apenas o necessário para análises comerciais, sem CPF nem endereço completo:

```sql
-- Tabela original (acesso restrito, apenas ao DBA e a processos internos)
CREATE TABLE clientes (
    id_cliente     SERIAL PRIMARY KEY,
    nome           VARCHAR(120),
    cpf            VARCHAR(14),
    endereco       VARCHAR(200),
    cidade         VARCHAR(80),
    estado         CHAR(2),
    valor_total_compras NUMERIC(12,2)
);

-- View segura para usuários especialistas e ferramentas de IA
CREATE VIEW clientes_visiveis AS
SELECT
    id_cliente,
    cidade,
    estado,
    valor_total_compras
FROM clientes;

-- Apenas a view é acessível pelo perfil de análise
GRANT SELECT ON clientes_visiveis TO role_especialista_ia;
REVOKE ALL ON clientes TO role_especialista_ia;
```

Com essa estrutura, uma consulta gerada por IA como "liste os clientes com maior valor de compras" retorna apenas cidade, estado e valor total — nunca CPF ou endereço, independentemente do prompt utilizado.

### 2.2 Exemplo de role customizada com limites de execução

```sql
-- Criação de role dedicada a usuários especialistas que usam IA
CREATE ROLE role_especialista_ia NOLOGIN;

-- Limite de tempo de execução de consultas (evita consultas pesadas travarem o servidor)
ALTER ROLE role_especialista_ia SET statement_timeout = '30s';

-- Associação de um usuário concreto à role
CREATE USER analista_maria LOGIN PASSWORD 'senha_segura';
GRANT role_especialista_ia TO analista_maria;
```

Esse conjunto garante que qualquer consulta submetida por `analista_maria` — manualmente ou via ferramenta de IA — seja automaticamente interrompida após 30 segundos, prevenindo a degradação de performance descrita na Seção 1.3.

### 2.3 Caso real ilustrativo: sistema de vendas

Em um sistema de vendas, um usuário especialista da área comercial usa um assistente de IA para gerar relatórios de faturamento por região. Sem controles adequados, um prompt como "compare o faturamento de todos os vendedores este ano" poderia gerar uma consulta que expõe comissões individuais e dados de desempenho de colegas, configurando uso indevido de informações (risco 5, Seção 1.3). Com a aplicação das estratégias descritas — view `vendas_por_regiao` sem dados individualizados de comissão, role com acesso restrito ao próprio time, e auditoria via `pgAudit` registrando cada consulta —, o mesmo prompt resulta em um relatório agregado e seguro, e qualquer tentativa de acesso fora do escopo é registrada e pode ser investigada pelo DBA.

## 3. Referências

- ELMASRI, R.; NAVATHE, S. B. *Fundamentals of Database Systems*. 7. ed. Pearson, 2015 — capítulos sobre perfis de usuários de banco de dados e funções do DBA.
- PostgreSQL Global Development Group. *PostgreSQL Documentation — Chapter on Database Roles and Privileges*. Disponível em: https://www.postgresql.org/docs/current/user-manag.html
- PostgreSQL Global Development Group. *PostgreSQL Documentation — pgAudit Extension and Logging*. Disponível em: https://www.postgresql.org/docs/current/runtime-config-logging.html
- BRASIL. *Lei nº 13.709, de 14 de agosto de 2018 (Lei Geral de Proteção de Dados Pessoais — LGPD)*. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm
- Cafegeek. *Perfis de Usuários de Banco de Dados*. Material de apoio do curso.
- Cafegeek. *Regras das Atividades Teóricas*. Material de apoio do curso.

## 4. Conclusões

O estudo deste cenário evidencia que a chegada da IA generativa ao ambiente de banco de dados não cria riscos totalmente novos, mas amplifica riscos já conhecidos — acesso excessivo, consultas ineficientes, vazamento de dados — ao remover a etapa de revisão humana que tradicionalmente existia no ciclo de desenvolvimento de consultas. O principal aprendizado do grupo é que a resposta a esse problema não é impedir o uso de IA, mas redesenhar a governança de dados para pressupor que qualquer consulta pode ter sido gerada automaticamente e, portanto, precisa ser contida por controles estruturais (views, roles, limites de execução) e não apenas por políticas de uso. O papel do DBA, nesse contexto, torna-se ainda mais estratégico: ele deixa de ser apenas o guardião da performance e da integridade técnica do banco e passa a ser também o principal responsável por traduzir exigências legais (como a LGPD) em controles técnicos concretos, sustentando a confiança organizacional no uso de ferramentas de IA para análise de dados.

## Link do Repositório Git

https://github.com/RenanPaiva140/ATIVIDADE-teorica-ia-dba-G2-05-Paiva.md
