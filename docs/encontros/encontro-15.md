# Encontro 15

## Tema

NoSQL e persistência poliglota.

## Objetivos

- Comparar modelos relacional e documental.
- Identificar diferenças entre estrutura, relacionamento e consulta.
- Evitar a adoção de NoSQL apenas por tendência.
- Escolher o armazenamento de acordo com os padrões de acesso.
- Reconhecer custos e benefícios da persistência poliglota.

## Questão orientadora

Quando um modelo não relacional resolve melhor o problema e quando ele apenas
transfere a complexidade para a aplicação?

## Organização sugerida (90 minutos)

1. Retomada e contextualização: 10 min
2. Modelos relacionais e documentais: 20 min
3. Padrões de acesso e estudo de caso: 20 min
4. Modelagem e comparação de alternativas: 30 min
5. Discussão e fechamento: 10 min

## Atividade e saída esperada

Comparar duas formas de persistir uma parte do sistema de solicitações: uma
modelagem relacional no PostgreSQL e uma representação documental. A decisão
deve considerar consultas, consistência, relacionamentos, evolução do schema e
necessidade de transações.

## Conexão com o percurso

O encontro amplia as alternativas de persistência sem abandonar os critérios
de integridade estudados anteriormente. A escolha da tecnologia deve partir do
problema e dos padrões de acesso, deixando explícitos seus impactos sobre
consistência, manutenção e operação.
