# Black Friday Lab

Laboratório de carga real com painel React/TypeScript, CloudFront/S3, integração direta API Gateway → SQS FIFO, processamento Lambda e estoque transacional DynamoDB. Os geradores são Lambdas distribuídas: a página inicia o teste e pode ser fechada durante a execução.

## Executar

Requer Node.js 22.18 ou superior. Siga o roteiro para criar os recursos na sua conta AWS. No console do API Gateway, obtenha a URL do stage e a chave do seu laboratório para configurar o painel.

Para configurar o front localmente, copie [site/.env.example](site/.env.example) para `site/.env` e preencha `VITE_API_URL` com a URL base do API Gateway, incluindo o stage. Esse arquivo fica fora do Git. Informe a chave da API na aba **Conexão & execução**.

```bash
cp site/.env.example site/.env
npm --prefix site ci
npm --prefix site run dev
```

`npm --prefix site run build` lê o mesmo arquivo e gera o site em `site/dist/`.

Para preparar o backend, rode `npm --prefix backend ci`. Para compilar e validar, use `npm --prefix backend run typecheck` e `npm --prefix backend run build`.

**Não executar testes, smoketests ou cargas sem solicitação explícita do usuário.** O comando `npm --prefix backend test` está disponível para uso quando autorizado; compila o backend e executa apenas os testes locais existentes. Os scripts `backend/tests/smoke-live.mjs` e `backend/load/verify-capacity.mjs` acessam a AWS e geram carga real.

## Usar o laboratório

1. Configure quantidade, janela (1–600 segundos) e estoque. O painel dimensiona os geradores automaticamente.
2. Comece com 1.000 pedidos em 10 segundos.
3. Inicie e acompanhe o relatório a cada 2 segundos. Há 10 segundos de preparação dos geradores.
4. Para ampliar a carga, ajuste o limite no backend e no build do painel e use a tabela do roteiro de montagem. Um milhão em 60 segundos equivale a cerca de 16.667 pedidos/s.
5. Baixe o relatório JSON pelo ícone na seção Execução. O ID fica salvo no navegador para retomar a consulta; a chave precisa ser informada novamente após recarregar.

Não existe promessa de alcançar a taxa configurada. Cada gerador limita a 128 requisições simultâneas e encerra novos envios ao fim da janela. Requisições em voo podem levar mais 10 segundos para terminar. A interface contabiliza separadamente as que não chegaram a ser enviadas. Configure mais geradores para reduzir esse limite, respeitando a concorrência da conta. O gerador acessa diretamente o API Gateway: o CloudFront entrega os arquivos do painel, não faz parte do caminho dos pedidos.

## Significado dos números

- **Aceitas**: HTTP 202 após o SQS confirmar o envio. Não significa venda concluída.
- **Erros no envio**: respostas HTTP não 2xx ou falhas de rede. Status HTTP e latências média/máxima por gerador constam no JSON. Falha de rede pode ocorrer depois de a mensagem ter sido aceita, gerando uma diferença entre aceitas e processadas.
- **Processados**: pedidos com estoque abatido por transação.
- **Sem estoque**: rejeição de negócio, gravada sem descontar estoque.
- **Não enviadas**: saldo do alvo que o gerador não conseguiu iniciar dentro da janela; não inclui geradores que falharam antes de finalizar.
- **DLQ**: mensagens que falharam vinte vezes no processamento. Fila e DLQ são aproximadas, compartilhadas entre execuções; os contadores do DynamoDB são por execução.
- **Tempo até a última atividade**: intervalo entre o início programado da carga e a última atualização de processamento ou conclusão de gerador. `elapsedMs` e `lastProcessedAt` constam no JSON. O tempo de processamento pode ultrapassar a janela de envio.
- **Concluído**: todos os geradores finalizaram e processados + sem estoque = aceitas. Em erros ambíguos de rede essa igualdade pode não ocorrer; o painel sinaliza para verificar o processamento após a janela e tolerância.

Os contadores dos geradores atualizam a cada 2 segundos. Consultas entre partições não são um snapshot atômico; durante o teste pequenas diferenças são esperadas. Se um gerador morrer abruptamente, seu último snapshot fica visível e ele não é considerado finalizado. Consultar logs é necessário nesses casos. Não se faz retry HTTP automático no gerador para esconder erros ou alterar o volume do teste.

## FIFO, idempotência e estoque

1.024 grupos FIFO nos novos testes permitem concorrência entre grupos. Cada pedido tem ID determinístico e um grupo derivado do índice módulo 1.024. O API Gateway faz validação do corpo e usa runId + orderId na deduplicação SQS. A Lambda lê os marcadores e o estoque, filtra reentregas e grava os novos marcadores com uma atualização do estoque por lote do mesmo grupo na mesma transação. Uma condição de saldo protege contra concorrência; conflitos exigem releitura. Mensagens repetidas não descontam de novo. Em falha de lote FIFO, retorna a mensagem com falha e todas as seguintes para retry.

O estoque está dividido em 1.024 chaves de partição distintas para evitar um contador único quente no DynamoDB. A soma inicial é exata. **Uma partição esgotada rejeita pedidos mesmo se outra tiver saldo**: esse é um inventário particionado, adequado ao experimento, não uma reserva global de estoque. Nenhuma partição pode ficar negativa. O desenho não promete ordenação global dos pedidos.

Dados e marcadores expiram após 7 dias via TTL (remoção assíncrona). A idempotência é limitada a esse período. A DLQ retém 14 dias. CloudWatch recebe logs das Lambdas. Não há reprocessamento automático da DLQ.

## Quotas e acesso

Confira os limites reais antes do cenário de 1 milhão:

```bash
aws apigateway get-account --profile seu-perfil --region us-east-1 --query throttleSettings
aws lambda get-account-settings --profile seu-perfil --region us-east-1
```

No ambiente de referência, a quota foi ampliada para 20.000 req/s. A cota padrão regional do API Gateway em us-east-1 é 10.000 req/s; confira o valor real ao usar outra conta. O usage plan deste laboratório permite 20.000 req/s e 10 milhões/dia, mas **não aumenta a cota da conta**. SQS FIFO, DynamoDB e Lambda também têm limites de vazão/concorrrência e podem precisar de aquecimento ou ajuste. Nenhuma cota é alterada automaticamente.

A API key é um segredo compartilhado para controle do laboratório, não identidade de usuário. API keys e quotas de usage plan não são mecanismos de autenticação nem limites rígidos de gastos. Para exposição em produção, adicione autenticação com IAM/Cognito ou authorizer e controles de acesso por usuário. Referência: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html

O endpoint de início usa POST. Não repita automaticamente em caso de timeout: pode ter criado uma execução e iniciado geradores. O laboratório não tem botão de cancelamento; a janela máxima é 600 segundos. Os testes consomem AWS e usam a concorrência compartilhada da conta.

## Estrutura

- `site/`: painel React e guias. O build gera `site/dist/`.
- `backend/`: Lambdas em TypeScript, módulos compartilhados e testes locais.
- `k6-load-test/`: geradores de chamadas HTTP em k6 e Go.
- `infraestrutura/terraform/`: configuração Terraform.
- [Roteiro no Console AWS](infraestrutura/docs/roteiro-console.md): montagem e configuração manual.

## Ajustes de capacidade (v2)

Novas execuções gravam `schemaVersion: 2` e `shards: 1024`. Os relatórios antigos usam 64 partições quando não há esse campo; o estoque existente não é recriado. A inicialização usa BatchWrite em lotes de 25, e snapshots usam BatchGet em lotes de 100. Há no máximo quatro chamadas desses lotes em paralelo.

O ambiente de referência usou cobrança on-demand e warm throughput de 12.000 unidades de leitura e 16.000 de escrita/s. O aumento inicial de escrita de 4.000 para 16.000 tem custo único estimado de US$ 7,80 em us-east-1, separado das requisições. Implantar em outra tabela/conta pode ter custo diferente. Isso prepara a capacidade inicial, sem garantir que todos os pedidos sejam processados dentro dos 60 segundos de envio.

O SQS continua FIFO high throughput, agora agrupando por execução + partição. O mapeamento permite até 128 processadoras simultâneas e a DLQ recebe mensagens após 20 recebimentos sem sucesso. As amostras de erro do gerador são limitadas a três por worker e aparecem no relatório e painel; a integração preserva a mensagem do SQS e mapeia RequestThrottled para 429. O gerador continua sem retries de envio para não ocultar falhas.

### Carga HTTP direta com k6

Com Node.js 22.18 ou mais recente, k6 instalado e credenciais AWS com acesso à tabela do laboratório, copie o arquivo de configuração uma vez e preencha o perfil, a tabela e a URL base do stage. Se a chave da API estiver ausente, o script pede que você a cole no terminal sem exibi-la.

```bash
cp k6-load-test/.env.example k6-load-test/.env
${EDITOR:-vi} k6-load-test/.env
./k6-load-test/run.sh 1000000 60
```

O primeiro argumento é o total de requisições. Use `1000000 60` para programar um milhão de chamadas em 60 segundos. O script cria o estoque no DynamoDB e faz chamadas `OPTIONS` espaçadas para preparar as conexões. Na janela medida, o cenário reparte as iterações entre os usuários virtuais e agenda um `POST /orders` por pedido. A carga passa pelo API Gateway, pelo SQS FIFO e pela Processor. Leia `orders_sent` para o total de POSTs e `orders_http_duration` para a latência HTTP dos pedidos. As métricas gerais `http_reqs` e `http_req_duration` incluem o aquecimento. Se o total enviado ficar abaixo da meta, ajuste `ORDER_TEST_VUS` ou distribua a carga. [Executor de iterações compartilhadas do k6](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/shared-iterations/)

O k6 imprime o resumo no terminal e salva `summary.json` e `dashboard.html` em `k6-load-test/reports/<runId>/`. Abra o HTML no navegador para ver o relatório.

O comando abaixo usa o `runId` impresso antes da carga e consulta no DynamoDB quantos pedidos foram processados ou rejeitados e quanto estoque restou. Se houver amostras, ele inclui P95 e P99 do envio à baixa no estoque. Esses percentis cobrem o processamento assíncrono, enquanto os do k6 medem a resposta HTTP.

```bash
./k6-load-test/run.sh status ID_DA_EXECUCAO
```

#### Um milhão de chamadas individuais a partir do Mac

Com Go instalado, use o mesmo `k6-load-test/.env` e rode na raiz do projeto:

```bash
./k6-load-test/run.sh local 1000000 60
```

O script compila o gerador Go, prepara o estoque no DynamoDB uma vez e agenda cerca de 16.667 chamadas/s. Cada chamada é um `POST /orders` individual. O gerador usa goroutines e reaproveita conexões HTTP. O padrão para este cenário é de até 40.000 requisições em trânsito, ajustável por `ORDER_LOCAL_CONCURRENCY` no `.env`.

O terminal e o arquivo `k6-load-test/reports/<runId>/summary.json` mostram tentativas, pedidos aceitos, falhas, pedidos não enviados na janela e P95/P99 do HTTP. Use `./k6-load-test/run.sh status <runId>` para consultar o processamento no DynamoDB. O comando sem `local` continua disponível para usar k6.

Antes de aumentar a carga, confira no Console AWS se o Warm Throughput do DynamoDB está ativo e se há um teto de escrita configurado.

## Lambdas em TypeScript

Cada Lambda tem um `.env.example` com as variáveis que o código usa, comentários e valores de referência: [Controller](backend/lambda-controller/.env.example), [Generator](backend/lambda-generator/.env.example) e [Processor](backend/lambda-processor/.env.example). Para deploy manual, configure os valores em **Lambda → Configuration → Environment variables**. Os exemplos servem como referência e não são carregados pela função.

O código de autoria do backend fica em TypeScript com modo `strict`, tipos de eventos de `aws-lambda` e tipos do SDK da AWS:

- `backend/shared/types.ts`: configuração da carga, pedido, estoque, marcador de deduplicação e métricas.
- `backend/lambda-controller/control.ts`: valida a configuração, inicializa o estoque, solicita os geradores e consulta o relatório.
- `backend/lambda-generator/generator.ts`: controla o ritmo dos envios, limita concorrência HTTP e persiste métricas.
- `backend/lambda-processor/processor.ts`: separa grupos FIFO e confirma estoque/marcadores em transação.
- `backend/shared/domain.ts`: validações, distribuição de estoque, chaves e agregação.
- `backend/shared/db.ts`: leitura/escrita do DynamoDB com lotes e retry de itens não processados.
- `backend/shared/errors.ts`: tratamento de erros recebidos como `unknown`.

`npm --prefix backend run build` gera módulos ESM não empacotados em `backend/dist/`, usados pelos testes locais quando autorizados. A pasta é ignorada pelo Git. Os testes foram ajustados para importar esses módulos compilados; a migração não executa testes automaticamente.

## Histórico no painel

A página inicial possui uma tabela por execução: data/hora de Brasília, cenário configurado, requisições enfileiradas (HTTP 202), falhas no envio, pedidos processados, duração do processamento e situação. Rejeições por falta de estoque e pedidos não enviados aparecem separadamente. O botão Abrir carrega o relatório individual.

A duração mede o intervalo entre `startAt` e a última gravação de processamento no estoque, incluindo espera na fila. Enquanto a execução não termina, o tempo é parcial. Testes antigos sem marcação de início mostram tempo indisponível em vez de uma estimativa inventada.

O endpoint autenticado `GET /runs` retorna dez resumos por página, dos mais recentes aos mais antigos, com cursor para as páginas seguintes. Os resumos ficam na partição `HISTORY` da tabela existente, sem TTL; não há Scan na consulta normal do painel. Somente execuções ainda não concluídas precisam reler suas partições de estoque, com até duas execuções em paralelo. O painel consulta a cada 15 segundos enquanto houver execuções em andamento; o relatório selecionado mantém seu acompanhamento de dois segundos.

Execuções novas são registradas pela Lambda de controle antes de iniciar os geradores. Consultar o histórico ou o relatório atualiza o resumo; resultados finais já salvos são reutilizados. Os registros de estoque, pedidos e metadados detalhados continuam expirando após sete dias. Se uma execução não tiver seu resultado final observado antes dessa expiração, a linha conserva o último resumo e informa que os detalhes expiraram, sem afirmar totais finais que não foram registrados.

## Guia visual

A página pública [`/guia-workshop.html`](site/public/guia-workshop.html) explica cada caixa da arquitetura, separa decisões de código e infraestrutura, traz um roteiro de mudanças graduais e inclui uma simulação local interativa. O simulador não chama a AWS nem inicia pedidos. O link também aparece na navegação do painel.

O roteiro sugere uma stack de demonstração separada para limites baixos de API Gateway, SQS FIFO, concorrência da processadora e teto de escrita do DynamoDB. Os passos são instruções para o workshop, não alterações automáticas na stack atual. Use o histórico e as métricas CloudWatch para comparar execuções com a mesma carga. A meta de 1 milhão em 60 segundos ainda requer validação real; o pedido de pausa dos testes continua em vigor.

## Envio em lote no laboratório

Selecione **SendMessageBatch** em **Modo de envio** para medir pedidos por segundo. Cada chamada HTTP envia até dez pedidos do mesmo grupo FIFO para `POST /orders/batch`. O API Gateway chama o SQS diretamente e devolve o resultado por entrada. O gerador separa pedidos aceitos, pedidos que falharam e pedidos que não chegaram a ser enviados. O painel e o histórico também mostram o número de chamadas HTTP iniciadas.

Para medir 200.000 chamadas HTTP individuais, mantenha o modo **Individual**. O modo em lote com 200.000 pedidos em 12 segundos mantém a taxa alvo de 16.667 pedidos/s com menos chamadas HTTP.
