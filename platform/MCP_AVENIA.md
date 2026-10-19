# MCP público da Avenia

## Propósito

Este documento descreve o endpoint MCP público da Avenia e registra o discovery realizado para a Spike do V-Stable. O servidor fornece documentação de integração, schemas e roteiros consultáveis por um cliente MCP. Ele não substitui a integração HTTP do backend com a API Avenia: o código do V-Stable continua responsável por autenticar e executar chamadas de negócio.

As observações de protocolo e inventário do endpoint são uma fotografia consultada em **18/10/2026 (America/Sao_Paulo; 19/10/2026 UTC)**. Elas podem mudar; repita o discovery antes de usar este inventário como contrato atual.

## Referências oficiais e versão da documentação

A documentação de negócio consultada veio dos recursos MCP da Avenia. O recurso `avenia-guide://operations-overview` referencia o [Avenia Integration Guide — Operations](https://integration-guide.avenia.io/docs/Operations/), e o recurso `avenia-guide://operations-quotes-and-tickets` referencia [Quotes and Tickets](https://integration-guide.avenia.io/docs/Operations/quotesAndTickets). O MCP também identifica `https://api-reference.avenia.io` como referência OpenAPI para os endpoints.

A resposta `initialize` do endpoint afirma que ele serve a documentação Avenia atualizada, mas o servidor anunciou apenas sua própria versão (`avenia-mcp` `0.4.1`). As respostas de `resources/list` não trouxeram data de atualização, versão do Integration Guide nem identificador de canal **LTS**. A consulta de 19/10/2026 ao [repositório oficial mantido pela Avenia](https://github.com/Avenia-io/avenia-mcp) também não encontrou uma política ou release marcada como LTS. Assim, a referência oficial utilizada é o [Avenia Integration Guide](https://integration-guide.avenia.io/), servido pelo MCP e referenciado pelo README dos mantenedores; não afirmamos que seja uma release LTS. Se “LTS” designar um canal separado, a Avenia precisa informar seu identificador e política de suporte.

A [especificação oficial MCP 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) descreve JSON-RPC e Streamable HTTP. Os contratos das superfícies usadas nesta validação estão nas páginas oficiais de [tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools), [resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) e [prompts](https://modelcontextprotocol.io/specification/2025-11-25/server/prompts).

## Endpoint e transporte

| Propriedade | Valor observado |
|---|---|
| URL | `https://mcp.avenia.io/mcp` |
| Transporte | Streamable HTTP sobre HTTPS |
| Protocolo | JSON-RPC 2.0 |
| Operações de cliente | `POST` com `Content-Type: application/json` e `Accept: application/json, text/event-stream` |
| Identidade MCP | `name=avenia-mcp`, `version=0.4.1` |
| Versões negociadas | `2025-06-18` e `2025-11-25` aceitas em `initialize` |
| Autenticação na descoberta | Nenhuma foi solicitada; nenhum token ou chave foi enviado |
| Resposta inicial | HTTP 200, `Content-Type: application/json` |
| Sessão | A resposta inicial não retornou `Mcp-Session-Id` |

O manual Avenia MCP anexo informa que o servidor usa Streamable HTTP e que o cliente deve enviar POSTs JSON-RPC. O teste confirmou o handshake e a descoberta por POST. Não testei um GET simples; a afirmação do manual de que GET retorna corpo vazio não foi verificada nesta Spike.

## Configuração do cliente

Use a URL pública exata. O manual da Avenia informa que não se deve preencher OAuth Client ID nem OAuth Client Secret para este servidor público de documentação.

Configuração genérica apresentada no manual:

```json
{
  "mcpServers": {
    "avenia": {
      "type": "http",
      "url": "https://mcp.avenia.io/mcp"
    }
  }
}
```

No VS Code/Copilot, o manual indica a chave `servers`:

```json
{
  "servers": {
    "avenia": {
      "type": "http",
      "url": "https://mcp.avenia.io/mcp"
    }
  }
}
```

Para Claude Code, o comando documentado no manual é:

```sh
claude mcp add --transport http avenia https://mcp.avenia.io/mcp
```

Os caminhos dos arquivos de configuração variam conforme o cliente. Consulte o manual anexo para os passos específicos de Claude Desktop, ChatGPT, Cursor e Windsurf. Não use a configuração de um pacote local como se fosse este endpoint remoto.

## Procedimento de discovery reproduzível

O teste foi realizado diretamente por JSON-RPC, sem depender de um chat ou de interpretação do modelo. Os exemplos abaixo não usam credenciais nem chamam a API de negócio.

### 1. Inicializar e confirmar a identidade

```sh
curl -i --max-time 20 \
  -X POST 'https://mcp.avenia.io/mcp' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-11-25' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"vstable-mcp-spike","version":"0.1.0"}}}'
```

Confirme HTTP 200 e confira `result.protocolVersion`, `result.serverInfo` e `result.capabilities`. Nesta consulta, `protocolVersion` foi `2025-11-25`, `serverInfo` foi `avenia-mcp` `0.4.1`, e foram anunciadas as superfícies `tools`, `resources` e `prompts`. Cada capacidade veio como objeto vazio; não foram anunciadas opções como `listChanged`.

Após inicializar, o cliente pode enviar a notificação `notifications/initialized`. Em seguida, faça uma requisição POST JSON-RPC separada por método de discovery. Exemplo para ferramentas:

```sh
curl -i --max-time 20 \
  -X POST 'https://mcp.avenia.io/mcp' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-11-25' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

Repita trocando o método por `resources/list` e `prompts/list`. Para consultar o conteúdo de um guia listado, use `resources/read` com o URI retornado, por exemplo `avenia-guide://operations-quotes-and-tickets`. As ferramentas de documentação são chamadas por `tools/call`; por exemplo, `avenia_list_endpoints` lista especificações e `avenia_describe_endpoint` descreve uma especificação pelo nome.

### 2. Teste de leitura sem operação de negócio

Foi lido o recurso `avenia-guide://operations-quotes-and-tickets` e foram descritos os endpoints abaixo por `avenia_describe_endpoint`:

| Nome de catálogo | Endpoint da API descrito | Autenticação segundo o schema MCP |
|---|---|---|
| `avenia_get_fixed_rate_quote` | `GET /v2/account/quote/fixed-rate` | Requerida |
| `avenia_create_ticket` | `POST /v2/account/tickets` | Requerida |

A leitura e a descrição retornam material documental. Nenhuma cotação, ticket, consulta de saldo, KYC, cadastro ou outra operação de negócio foi enviada à API Avenia.

## Capacidades e inventário observados

### Ferramentas MCP

`tools/list` retornou sete ferramentas. Todas tinham schema de entrada e a anotação `readOnlyHint: true`; nenhuma trouxe `outputSchema` na resposta observada.

| Nome | Argumentos observados | Função anunciada |
|---|---|---|
| `avenia_list_guides` | Nenhum | Lista guias por ID, título, categoria e resumo |
| `avenia_read_guide` | `id` string obrigatório | Lê o Markdown de um guia pelo ID |
| `avenia_list_flows` | Nenhum | Lista fluxos e seus argumentos |
| `avenia_get_flow` | `name` string obrigatório | Retorna o passo a passo de um fluxo por nome |
| `avenia_list_endpoints` | Nenhum | Lista catálogo de APIs com método, path e requisito de autenticação; declara que é somente leitura |
| `avenia_describe_endpoint` | `name` string obrigatório | Retorna método, path, parâmetros e schema de request; declara que não chama a API |
| `avenia_get_public_key` | Nenhum | Obtém chave pública; catálogo associa a `GET /v2/public-key` (`authRequired: false`) |

As seis primeiras ferramentas tinham `inputSchema` e `readOnlyHint: true`; `avenia_get_public_key` também tinha schema vazio e `readOnlyHint: true`. Nenhuma retornou `outputSchema`. O hint é metadado declarativo, não uma garantia de segurança ou controle de autorização.

O MCP hospedado apresentou **7 ferramentas** nesta fotografia: seis para documentação/catálogo e `avenia_get_public_key`, que representa a leitura de `GET /v2/public-key`. O catálogo `avenia_list_endpoints` contém **66 especificações de API**; estas não devem ser confundidas com as ferramentas anunciadas pelo endpoint hospedado. `avenia_get_public_key` foi apenas catalogada, não invocada. O [README dos mantenedores](https://github.com/Avenia-io/avenia-mcp) descreve o conector hospedado como documentação somente leitura e separa dele o pacote local, que pode executar operações da API quando configurado com uma chave.

### Recursos

`resources/list` retornou 36 recursos `text/markdown` com URI `avenia-guide://`. A lista completa observada:

```text
avenia-guide://account-about-login
avenia-guide://account-login
avenia-guide://account-recovery
avenia-guide://account-statement
avenia-guide://account-address-api
avenia-guide://subaccounts
avenia-guide://bank-accounts-own
avenia-guide://bank-accounts-static-brcode
avenia-guide://beneficiary-bank-brl
avenia-guide://beneficiary-bank-usd
avenia-guide://beneficiary-bank-eur
avenia-guide://beneficiary-wallets
avenia-guide://email-notification
avenia-guide://export-infinia-migration
avenia-guide://export-template
avenia-guide://kyb-level-1
avenia-guide://kyb-level-1-api
avenia-guide://kyb-usd
avenia-guide://kyb-eur
avenia-guide://kyc-level-1
avenia-guide://kyc-usd
avenia-guide://kyc-eur
avenia-guide://operations-overview
avenia-guide://operations-combinations
avenia-guide://operations-quotes-and-tickets
avenia-guide://operations-ticket-receipt
avenia-guide://sandbox-receive-mock-funds
avenia-guide://security-api-keys-guide
avenia-guide://security-api-keys-management
avenia-guide://security-mfa
avenia-guide://usecase-payout-brcode
avenia-guide://usecase-pix2stable-onchain
avenia-guide://usecase-pix2stable-stable2pix
avenia-guide://webhooks-management
avenia-guide://webhooks-events
avenia-guide://webhooks-verifying-authenticity
```

`resources/templates/list` foi testado e respondeu JSON-RPC `-32601 Method not found`. O anúncio de `resources` não implicou suporte observado a templates.

### Prompts

`prompts/list` retornou nove prompts. Os argumentos são templates de fluxo, não chamadas de operação por si só:

- `avenia_flow_pix_to_stablecoin_onchain`
- `avenia_flow_stablecoin_to_pix`
- `avenia_flow_kyc_level_1`
- `avenia_flow_kyb_level_1_api`
- `avenia_flow_register_webhook`
- `avenia_flow_sandbox_mock_funds`
- `avenia_flow_create_subaccount_with_kyc`
- `avenia_flow_payout_brcode`
- `avenia_flow_create_api_key`

Argumentos declarados nos prompts observados:

| Prompt | Obrigatórios | Opcionais |
|---|---|---|
| `avenia_flow_pix_to_stablecoin_onchain` | `amountBRL`, `walletAddress` | `outputCurrency`, `outputChain`, `subAccountId` |
| `avenia_flow_stablecoin_to_pix` | `amountStable`, `pixKey` | `inputCurrency`, `subAccountId` |
| `avenia_flow_kyc_level_1` | Nenhum | `submissionMode`, `subAccountId` |
| `avenia_flow_kyb_level_1_api` | Nenhum | `subAccountId` |
| `avenia_flow_register_webhook` | `endpointUrl`, `events` | Nenhum |
| `avenia_flow_sandbox_mock_funds` | `amountBRL` | `subAccountId` |
| `avenia_flow_create_subaccount_with_kyc` | `name` | `accountType` |
| `avenia_flow_payout_brcode` | `brCode` | `subAccountId` |
| `avenia_flow_create_api_key` | Nenhum | `keyName` |

Um prompt é um template, mas pode orientar o modelo a executar ferramentas. Os argumentos incluem valores financeiros, endereço de carteira, chave Pix, IDs de subconta e passos para criação de chave API. Use apenas valores fictícios e, para esta Spike, não invoque prompts operacionais. Antes de permitir chamadas em resposta a um prompt, confira ferramentas e parâmetros propostos.

### Catálogo de endpoints e fluxos Avenia

As chamadas de leitura `avenia_list_endpoints` e `avenia_list_flows` retornaram, respectivamente, 66 especificações de API e nove fluxos. A tabela registra o nome do item no catálogo, método, path e requisito de autenticação; detalhes de query/body são obtidos por `avenia_describe_endpoint`. Esses 66 são operações documentadas da API, não operações executadas pelo MCP hospedado. A resposta de `avenia_describe_endpoint` informa bases sandbox/produção e campos de request, mas não fornece schema de resposta completo nem códigos de erro.

Inventário nomeado observado em 19/10/2026 UTC:

| Ferramenta de catálogo | Método | Path da API | Auth required |
|---|---|---|---|
| `avenia_create_beneficiary_ars_bank_account` | `POST` | `/v2/account/beneficiaries/bank-accounts/ars` | sim |
| `avenia_create_beneficiary_brl_bank_account` | `POST` | `/v2/account/beneficiaries/bank-accounts/brl` | sim |
| `avenia_create_beneficiary_eur_bank_account` | `POST` | `/v2/account/beneficiaries/bank-accounts/eur` | sim |
| `avenia_create_beneficiary_usd_bank_account` | `POST` | `/v2/account/beneficiaries/bank-accounts/usd` | sim |
| `avenia_create_beneficiary_wallet` | `POST` | `/v2/account/beneficiaries/wallets` | sim |
| `avenia_create_payment_session` | `POST` | `/v2/account/payment-session` | sim |
| `avenia_create_payment_session_ticket` | `POST` | `/v2/payment-session/{payment-session-id}/create-ticket` | sim |
| `avenia_create_sub_account` | `POST` | `/v2/account/sub-accounts` | sim |
| `avenia_create_ticket` | `POST` | `/v2/account/tickets` | sim |
| `avenia_create_w8_ben` | `POST` | `/v2/kyc/w8ben` | sim |
| `avenia_delete_beneficiary_brl_bank_account` | `DELETE` | `/v2/account/beneficiaries/bank-accounts/brl/{bank-account-id}` | sim |
| `avenia_delete_beneficiary_wallet` | `DELETE` | `/v2/account/beneficiaries/wallets/{wallet-id}` | sim |
| `avenia_delete_webhook` | `DELETE` | `/v2/notifications/webhooks/{webhook-id}` | sim |
| `avenia_get_access_info` | `GET` | `/v2/account/access-info` | sim |
| `avenia_get_account_info` | `GET` | `/v2/account/account-info` | sim |
| `avenia_get_account_wallet_balances` | `GET` | `/v2/account/balances` | sim |
| `avenia_get_attempt_by_id` | `GET` | `/v2/kyc/attempts/{attempt-id}` | sim |
| `avenia_get_attempts` | `GET` | `/v2/kyc/attempts` | sim |
| `avenia_get_beneficiary_ars_bank_account_by_id` | `GET` | `/v2/account/beneficiaries/bank-accounts/ars/{bank-account-id}` | sim |
| `avenia_get_beneficiary_ars_bank_accounts` | `GET` | `/v2/account/beneficiaries/bank-accounts/ars` | sim |
| `avenia_get_beneficiary_brl_bank_account_by_id` | `GET` | `/v2/account/beneficiaries/bank-accounts/brl/{bank-account-id}` | sim |
| `avenia_get_beneficiary_brl_bank_accounts` | `GET` | `/v2/account/beneficiaries/bank-accounts/brl` | sim |
| `avenia_get_beneficiary_eur_bank_account_by_id` | `GET` | `/v2/account/beneficiaries/bank-accounts/eur/{bank-account-id}` | sim |
| `avenia_get_beneficiary_eur_bank_accounts` | `GET` | `/v2/account/beneficiaries/bank-accounts/eur` | sim |
| `avenia_get_beneficiary_usd_bank_account_by_id` | `GET` | `/v2/account/beneficiaries/bank-accounts/usd/{bank-account-id}` | sim |
| `avenia_get_beneficiary_usd_bank_accounts` | `GET` | `/v2/account/beneficiaries/bank-accounts/usd` | sim |
| `avenia_get_beneficiary_wallet_by_id` | `GET` | `/v2/account/beneficiaries/wallets/{wallet-id}` | sim |
| `avenia_get_beneficiary_wallets` | `GET` | `/v2/account/beneficiaries/wallets` | sim |
| `avenia_get_brl_fiat_rail` | `GET` | `/v2/account/bank-accounts/brl` | sim |
| `avenia_get_document_by_id` | `GET` | `/v2/documents/{document-id}` | sim |
| `avenia_get_email_notification_config` | `GET` | `/v2/notifications/email-config` | sim |
| `avenia_get_fixed_rate_quote` | `GET` | `/v2/account/quote/fixed-rate` | sim |
| `avenia_get_limits` | `GET` | `/v2/account/limits` | sim |
| `avenia_get_metadata` | `GET` | `/v2/account/metadata` | sim |
| `avenia_get_payment_session_by_id` | `GET` | `/v2/payment-session/{payment-session-id}` | sim |
| `avenia_get_payment_session_fixed_quote` | `GET` | `/v2/payment-session/{payment-session-id}/fixed-quote` | sim |
| `avenia_get_payment_session_preferences` | `GET` | `/v2/account/payment-session/preferences` | sim |
| `avenia_get_payment_session_ticket` | `GET` | `/v2/payment-session/{payment-session-id}/get-ticket` | sim |
| `avenia_get_payment_sessions` | `GET` | `/v2/account/payment-session` | sim |
| `avenia_get_pix_info` | `GET` | `/v2/account/bank-accounts/brl/pix-info` | sim |
| `avenia_get_public_key` | `GET` | `/v2/public-key` | não |
| `avenia_get_statement` | `GET` | `/v2/account/statement` | sim |
| `avenia_get_static_br_code` | `GET` | `/v2/account/bank-accounts/brl/static-br-code` | sim |
| `avenia_get_sub_account_by_id` | `GET` | `/v2/account/sub-accounts/{sub-account-id}` | sim |
| `avenia_get_sub_accounts` | `GET` | `/v2/account/sub-accounts` | sim |
| `avenia_get_ticket_by_id` | `GET` | `/v2/account/tickets/{ticket-id}` | sim |
| `avenia_get_ticket_receipt` | `GET` | `/v2/account/tickets/{ticket-id}/receipt` | sim |
| `avenia_get_tickets` | `GET` | `/v2/account/tickets` | sim |
| `avenia_get_w8_ben_by_id` | `GET` | `/v2/kyc/w8ben/{w8ben-id}` | sim |
| `avenia_get_w8_ben_forms` | `GET` | `/v2/kyc/w8ben` | sim |
| `avenia_get_webhook_by_id` | `GET` | `/v2/notifications/webhooks/{webhook-id}` | sim |
| `avenia_get_webhook_event_by_id` | `GET` | `/v2/notifications/webhooks/events/{event-id}` | sim |
| `avenia_get_webhook_event_submission_by_id` | `GET` | `/v2/notifications/webhooks/attempts/{attempt-id}` | sim |
| `avenia_get_webhook_event_submissions` | `GET` | `/v2/notifications/webhooks/attempts` | sim |
| `avenia_get_webhook_events` | `GET` | `/v2/notifications/webhooks/events` | sim |
| `avenia_get_webhooks` | `GET` | `/v2/notifications/webhooks` | sim |
| `avenia_import_token` | `POST` | `/v2/kyc/import-token` | sim |
| `avenia_list_documents` | `GET` | `/v2/documents` | sim |
| `avenia_new_level_1_api` | `POST` | `/v2/kyc/new-level-1/api` | sim |
| `avenia_new_level_1_web_sdk` | `POST` | `/v2/kyc/new-level-1/web-sdk` | sim |
| `avenia_register_webhook` | `POST` | `/v2/notifications/webhooks` | sim |
| `avenia_save_payment_session_preferences` | `POST` | `/v2/account/payment-session/preferences` | sim |
| `avenia_update_webhook` | `PATCH` | `/v2/notifications/webhooks` | sim |
| `avenia_update_webhook_event_submission` | `PATCH` | `/v2/notifications/webhooks/attempts` | sim |
| `avenia_upload_document` | `POST` | `/v2/documents` | sim |
| `avenia_upsert_email_notification_config` | `PUT` | `/v2/notifications/email-config` | sim |

## Divergência com o manual anexo

O [Manual Avenia MCP fornecido para a Spike](</home/ev176849@Sulamerica.br/Documentos/faculdade/ages/Manual_Avenia_MCP.pdf>) registra sete ferramentas, 37 guias, 68 endpoints e nove fluxos. No discovery feito em 18/10/2026, o servidor retornou sete ferramentas, 36 recursos/guias, 66 endpoints e nove fluxos. As contagens do servidor foram obtidas das respostas MCP, e não copiadas do manual.

O guia `operations-overview` também traz um exemplo de `curl` para quote sem cabeçalhos de autenticação, enquanto `avenia_describe_endpoint` marca esse endpoint como exigindo autenticação. Registre a inconsistência para revisão com a Avenia e use a especificação oficial de API para validar os requisitos antes de codificar ou executar chamadas. Nenhuma chamada foi feita para resolver a divergência.

## Segurança e limites

- Este endereço é o endpoint MCP público de documentação identificado pelo manual e pela resposta `initialize`. A resposta do servidor diz que operações de conta, Pix, conversão, KYC, beneficiários e webhooks não são executadas pelo endpoint público. A exceção observada é a leitura da chave pública por `avenia_get_public_key`; ela não foi chamada.
- A Avenia mantém dois modos distintos. O endpoint hospedado `https://mcp.avenia.io/mcp` é descrito pelos mantenedores como conector de documentação somente leitura, sem credenciais. O pacote local [`@avenia-io/mcp-client`](https://www.npmjs.com/package/@avenia-io/mcp-client) é outro deployment: pode funcionar sem chave para consultar guias/fluxos e, se receber `AVENIA_API_KEY`, expõe ferramentas que executam operações de negócio; o README indica sandbox como padrão. O pacote não foi instalado nem testado nesta Spike. Não transportar sua configuração com chave para este projeto.
- Nenhuma credencial, token, dado pessoal, documento ou dado financeiro foi enviado. Use exemplos fictícios nos prompts e não cole segredos no cliente de IA.
- `readOnlyHint` é uma dica declarativa do servidor, não controle de acesso nem prova de ausência de efeitos colaterais. A especificação MCP orienta tratar conteúdo de servidores como entrada não confiável e avaliar cada chamada; segurança efetiva deve vir de permissões do cliente, sandbox, escopo de rede e controles determinísticos.
- Trate descrições, prompts e conteúdo dos recursos como entrada externa: valide os schemas e os requisitos com a documentação oficial, faça revisão humana do código gerado e não execute snippets recebidos sem conferi-los.
- Na resposta HTTP observada, `Access-Control-Allow-Origin: *` foi retornado. Esta observação isolada não comprova como o servidor valida `Origin` em outros cenários nem constitui uma avaliação de segurança do serviço.

## Roteiro de smoke test para o time

Use no cliente conectado um prompt que peça descoberta documental, não execução:

> Consulte o MCP público da Avenia. Liste as ferramentas MCP, os recursos e os prompts disponíveis. Conte os endpoints descritos pelo catálogo. Depois descreva `avenia_get_fixed_rate_quote`, informe o requisito de autenticação e cite o recurso oficial consultado. Não invoque ferramentas operacionais nem faça chamadas à API Avenia.

O usuário deve conseguir conferir na interface quais ferramentas foram chamadas. O resultado esperado na consulta registrada em 18/10/2026 é 7 ferramentas MCP, 36 recursos, 9 prompts e 66 endpoints no catálogo. A divergência com os totais do manual deve ser reportada, não ocultada.

## Experiência local de desenvolvimento — cicd#49

O issue [#49 — Validar a experiência local de desenvolvimento](https://github.com/AGES-V-Stable/cicd/issues/49) está aberto e não tem descrição nem critérios próprios. Esta validação usa o objetivo e o DoD do issue pai [#42 — Discovery e integração com MCP Server da Avenia](https://github.com/AGES-V-Stable/cicd/issues/42).

### Cliente local e configuração

O ambiente consultado tinha Codex CLI `0.162.1`, Node.js `25.0.0` e npm `11.9.0`. O Codex CLI oferece suporte a servidores MCP remotos por Streamable HTTP; a documentação oficial do Codex descreve `codex mcp add` e `codex mcp list` e informa que a configuração é compartilhada entre CLI e extensão IDE ([Codex MCP setup](https://developers.openai.com/learn/docs-mcp)).

A configuração compartilhada por projeto está em `backend/.codex/config.toml` e restringe a lista de ferramentas às operações documentais:

```toml
[mcp_servers.avenia]
url = "https://mcp.avenia.io/mcp"
enabled_tools = [
  "avenia_list_guides",
  "avenia_read_guide",
  "avenia_list_flows",
  "avenia_get_flow",
  "avenia_list_endpoints",
  "avenia_describe_endpoint",
]
default_tools_approval_mode = "prompt"
```

A allowlist omite `avenia_get_public_key` e novas ferramentas não entram automaticamente. O manual Avenia não pede OAuth Client ID nem Client Secret para o endpoint hospedado. O Codex carrega `.codex/config.toml` apenas em repositórios confiáveis; esta configuração é compartilhada entre CLI e extensão IDE.

Valide a configuração do projeto com:

```sh
codex mcp list
codex mcp get avenia
```

Ambos os comandos foram executados no backend. A saída mostrou o servidor habilitado, transporte `streamable_http`, URL correta, as seis ferramentas permitidas e aprovação `prompt`. Não foi alterado o perfil global com `codex mcp add`. A configuração de projeto usa as opções `enabled_tools` e `default_tools_approval_mode` documentadas oficialmente pelo Codex.

### Resultado do smoke test do cliente

As primeiras tentativas de `codex exec --ephemeral` no sandbox falharam antes de iniciar o app-server, com `failed to initialize in-process app-server client: Read-only file system`, inclusive usando `/tmp` como diretório de trabalho. Com a execução autorizada fora do sandbox restrito, a sessão local iniciou e conectou ao endpoint MCP usando o override temporário acima.

Na sessão, o Codex conseguiu chamar `avenia_list_guides` e `avenia_list_flows` e ler o recurso `avenia-guide://operations-overview`. Isso comprova a experiência local básica de conexão, invocação de ferramentas documentais e leitura de recurso. A saída do MCP incluiu os fluxos de conversão entre moedas fiat e cripto, pagamentos/saques e swaps entre blockchains. Nenhuma API de negócio foi chamada e não foram consultados dados de conta ou credenciais.

O cliente tentou `resources/templates/list` durante a descoberta automática e o servidor respondeu `-32601 Method not found`. Esse método não é necessário para listar os recursos já conhecidos ou ler o URI acima; a consulta direta `resources/read` funcionou. A listagem completa das capacidades (`tools/list`, `resources/list` e `prompts/list`) foi validada separadamente por JSON-RPC direto, conforme registrado anteriormente.

Também tentei executar o [MCP Inspector oficial](https://github.com/modelcontextprotocol/inspector) em modo CLI contra o endpoint público:

```sh
npx -y @modelcontextprotocol/inspector --cli https://mcp.avenia.io/mcp --transport http --method tools/list
```

A execução não chegou ao Inspector. O primeiro acesso ao registry falhou por resolução DNS (`EAI_AGAIN`); após autorização para repetir fora do sandbox, o registry npm configurado retornou `403 Forbidden`. O pacote não foi instalado e não tentei contornar a política usando outro registry. A documentação oficial do Inspector confirma que a versão atual oferece CLI para consultar tools, resources e prompts, além da interface Web.

### Conclusão e próximo teste no computador do desenvolvedor

A experiência local no Codex foi validada com configuração por projeto: `codex mcp list/get` reconheceu endpoint, allowlist documental e aprovação `prompt`; uma sessão conectou, executou ferramentas de guia/fluxo e leu `operations-overview`. O comando global `codex mcp add` não foi executado para não alterar o perfil do usuário. O smoke test independente pelo Inspector permanece impedido pelo acesso ao registry npm; esse impedimento não indica falha do endpoint Avenia.

O teste restante no Inspector é opcional para a validação básica do issue: em ambiente com acesso autorizado ao registry, repetir o comando acima e registrar as listagens. A validação do Codex pode ser reproduzida com `codex mcp add avenia --url https://mcp.avenia.io/mcp`, `codex mcp list` e um prompt restrito a guias e schemas. Não chamar `avenia_get_public_key` nem executar operações de API.


## Comparação do fluxo quote → ticket com o backend

O fluxo foi validado por leitura documental do MCP e inspeção estática do código, sem enviar quote/ticket a Avenia. As referências são o recurso `avenia-guide://operations-quotes-and-tickets`, `avenia_describe_endpoint` para `avenia_get_fixed_rate_quote` e `avenia_create_ticket`, e o [Integration Guide — Quotes and Tickets](https://integration-guide.avenia.io/docs/Operations/quotesAndTickets/).

| Etapa | Contrato consultado | Implementação V-Stable | Resultado |
|---|---|---|---|
| Quote | `GET /v2/account/quote/fixed-rate`; auth requerida; query inclui moeda e método de entrada/saída, `inputAmount` ou `outputAmount`, flags third-party e `subAccountId` | `AveniaClient` constrói query e assina `GET` com `X-API-Key`, timestamp e assinatura. `AveniaQuoteRequest` exige exatamente um dos valores de amount e normaliza flags third-party ausentes para `false`. A criação da transferência usa subconta individual | Método/path, subconta e validações centrais estão alinhados; o backend impõe XOR de amount, regra que o schema de ferramenta não expressa |
| Quote → ticket | O guia descreve quote antes do ticket e uso de `quoteToken`; `POST /v2/account/tickets`, auth requerida e `subAccountId` opcional | `AveniaTransferService.createTransfer` obtém quote nova, valida token, moedas, métodos, valores, preço e taxas, injeta o token recebido no DTO e só então cria ticket | Sequência alinhada e protegida contra ticket com quote incompleta/inconsistente |
| Ticket por destino | A documentação MCP descreve body com `quoteToken` e estruturas de dados de ticket | `TransferService` monta uma única variante de destino de acordo com beneficiário/canal (PIX, wallet, ACH, wire ou SEPA), referenciando beneficiário registrado por ID; serialização omite campos nulos | O schema de `avenia_describe_endpoint` consultado marca simultaneamente `ticketBrlPixInput`, `ticketBlockchainInput` e `ticketMarkupInput` como required. Esses nomes não correspondem ao DTO do backend (`ticketBlockchainOutput`, `ticketBrlPixOutput`, `ticketUsdOutput`, etc.) e o schema retornado não representa claramente as variantes mutuamente exclusivas. Tratar como divergência documental a validar no API Reference antes de alterar código |
| Respostas/erros | A ferramenta de descrição não inclui schema de resposta nem catálogo de erros; o guia/OpenAPI são as referências para isso | `AveniaTransferService` valida retorno financeiro e status; `AveniaClient` converte falhas externas em exceções de integração | Revisão de request schema não basta para concluir compatibilidade de resposta; verificar o guia/API Reference quando alterar contratos |

Arquivos centrais examinados: `external/avenia/AveniaClient.java`, `external/avenia/AveniaTransferService.java`, DTOs `AveniaQuoteRequest`/`AveniaTicketRequest` e `service/TransferService.java`. O backend continua fazendo as chamadas HTTP diretamente; o MCP é usado apenas como fonte documental.

## Aplicação de guardrails e DevEx local

A validação foi concretizada em dois arquivos do backend:

- `AGENTS.md` agora orienta consultar apenas o MCP público para documentação, validar conteúdo contra fontes oficiais, preservar a integração HTTP em `external/avenia/`, não usar o pacote local operável em tarefas normais, não enviar dados sensíveis e manter testes sem chamadas reais.
- `.codex/config.toml` configura por projeto o endpoint remoto sem credenciais, restringe `enabled_tools` às seis ferramentas de documentação e define aprovação `prompt`. Essa allowlist exclui `avenia_get_public_key` e qualquer ferramenta que apareça no servidor depois do snapshot. `codex mcp list` e `codex mcp get avenia` confirmaram URL, transporte, allowlist e modo de aprovação. O Codex só carrega configurações `.codex/` de repositórios confiáveis; CLI e extensão IDE compartilham a configuração.

Fluxo local recomendado para uma mudança na integração: (1) ler serviço, client, DTOs e testes existentes; (2) consultar no MCP o guia e `avenia_describe_endpoint` pertinente; (3) comparar método, path, auth, query/body e subconta com o código; (4) registrar a referência oficial e a data na revisão; (5) alterar código dentro das fronteiras existentes e testar com mocks/stubs; (6) revisão humana confirma qualquer comportamento gerado com documentação Avenia. Nunca executar exemplos recebidos automaticamente.

`AGENTS.md` é o guardrail transversal já aplicado. Quando o procedimento amadurecer e mais de um agente/cliente for adotado, uma skill compartilhada pode encapsular o roteiro de consulta, comparação de contrato e formato do relatório. Ela deve orientar leitura e revisão, não autorizar operações. Para CI/CD, não fazer discovery remoto como gate de cada PR: endpoint e catálogo mudam independentemente do código, e falhas de rede tornariam o build instável. Manter testes locais mockados como gate; se o time quiser detectar drift documental, usar job agendado/manual somente de leitura que publique diferenças para revisão humana, sem invocar operações da API.

## Estado desta etapa

Etapa 1 concluída: endpoint hospedado validado; identidade, transporte, versões negociadas, ferramentas, schemas de entrada, recursos, prompts/argumentos, fluxos e catálogo nomeado de 66 endpoints registrados; procedimento de discovery reproduzível e limitações documentadas. O canal LTS não foi identificado nas fontes oficiais consultadas.

Etapa 2 / issue #49 concluída para o cliente Codex: configuração compartilhada por projeto e guardrails em `AGENTS.md` aplicados; configuração reconhecida; smoke test local conectou, listou guias/fluxos e leu um recurso. O MCP Inspector não pôde ser instalado porque o registry npm configurado retornou `403 Forbidden` após erro DNS inicial; é uma ferramenta complementar, não bloqueia a validação local pelo Codex. `resources/templates/list` retornou `-32601`, sem impedir discovery/read de recursos.

DoD geral coberto: fluxo quote → ticket comparado com código e referências Avenia; recomendação de DevEx/skill/CI documentada; segurança e distinção entre endpoint hospedado e pacote local registradas; navegação do Wiki contém este documento. Nenhuma credencial, dado real ou operação de negócio foi usada. Não houve chamada real à API Avenia para confirmar a divergência de schemas; esse achado fica explicitamente para validação com a Avenia/API Reference.

## Fontes

- [Avenia Public MCP](https://mcp.avenia.io/mcp) — endpoint consultado.
- [Avenia Integration Guide — Operations](https://integration-guide.avenia.io/docs/Operations/) — fonte indicada pelo recurso MCP `operations-overview`.
- [Avenia Integration Guide — Quotes and Tickets](https://integration-guide.avenia.io/docs/Operations/quotesAndTickets) — fonte indicada pelo recurso MCP lido.
- [Avenia API Reference](https://api-reference.avenia.io) — referência OpenAPI indicada pelo `initialize` do MCP.
- [Avenia MCP Server — repositório dos mantenedores](https://github.com/Avenia-io/avenia-mcp) — separa o endpoint hospedado de documentação do pacote local com possibilidade de operações.
- [Avenia MCP client — npm](https://www.npmjs.com/package/@avenia-io/mcp-client) — instruções publicadas para o pacote local e seus modos.
- [Codex MCP setup](https://developers.openai.com/learn/docs-mcp) — referência oficial para registrar e listar servidores MCP no Codex CLI.
- [Codex configuration reference](https://developers.openai.com/codex/config-reference) — configuração de projeto, allowlist `enabled_tools` e aprovação das ferramentas MCP.
- [MCP tool annotations — mantenedores do protocolo](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/) — `readOnlyHint` é uma dica, não enforcement de segurança.
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) — repositório oficial do Inspector; referência para modo CLI.
- [MCP Specification 2025-11-25 — Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports), [Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools), [Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) e [Prompts](https://modelcontextprotocol.io/specification/2025-11-25/server/prompts).
