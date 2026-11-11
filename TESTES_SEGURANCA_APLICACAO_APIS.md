# Testes de segurança da aplicação e APIs — Issue #85

**Issue:** [AGES-V-Stable/cicd#85](https://github.com/AGES-V-Stable/cicd/issues/85)  
**Épico:** [cicd#80](https://github.com/AGES-V-Stable/cicd/issues/80)  
**Planejamento de referência:** [Relatório da #83](PLANEJAMENTO_E_RECONHECIMENTO_TECNICO.md)  
**Data:** 10/11/2026 — America/Sao_Paulo  
**Condição:** avaliação local; entidades sintéticas; integrações externas simuladas.  
**Situação:** avaliação local complementada e evidências registradas. Os controles existentes foram avaliados nas condições acordadas; funcionalidades ausentes e limites da avaliação estão identificados nas seções 9 e 10. Não representa aprovação de segurança nem certificação.

## 1. Resumo executivo

Os testes locais reproduziram oito grupos de fragilidades: confirmação de oferta sem TOTP; perda do controle de reutilização e bloqueio TOTP entre instâncias/reinícios; permissões administrativas excessivas; validação e consumo de TOTP sem exclusão entre operações concorrentes; aceitação de arquivos truncados identificados apenas pelo prefixo PDF/PNG/JPEG; manutenção de sessão após alteração de senha na base isolada; perda de precisão de valores cripto no histórico; e aceitação de tarifa negativa no adapter BlindPay.

Também foram observados controles efetivos nos cenários executados: rejeição de JWT inválido/expirado e de challengeToken em rotas protegidas, bloqueio de usuários empresariais nas rotas administrativas, isolamento de recursos de empresas diferentes, rejeição de oferta expirada ou valores adulterados, unicidade de uma execução sob cinco confirmações concorrentes e autenticação/deduplicação de webhooks BlindPay.

| Execução | Casos | PASS | FAIL | Ignorados |
|---|---:|---:|---:|---:|
| Regressão existente do backend | 1.244 | 1.238 | 0 | 6 |
| Regressão existente do frontend | 772 | 772 | 0 | 0 |
| Diagnóstico específico da #85, após complementação | 217 | 198 | 19 | 0 |

Os 19 FAIL são asserções de comportamento seguro que a versão avaliada não satisfaz; **não são 19 vulnerabilidades independentes**. A seção 6 agrupa as causas e limita o impacto ao que foi demonstrado. A execução diagnóstica definitiva não apresentou falhas de inicialização, consultas SQL dos testes ou configuração dos mocks.

Foram acrescentados e executados **46 casos** para completar a avaliação local de sessão, moedas, escalas, tarifas, persistência, recotação até payout simulado, estados terminais, falhas dos clientes e limites multipart em HTTP real. As verificações locais disponíveis anteriormente pendentes foram substituídas por resultados PASS/FAIL. As linhas PARTIAL restantes identificam funcionalidades ausentes, configurações externas não exercitadas ou conclusões mais amplas que os casos demonstram; não significam que os cenários locais aqui definidos ficaram sem execução.

A cobertura do frontend foi de 95,74% de linhas, 93,67% de statements, 93,90% de funções e 87,25% de branches. Cobertura de código e regressão funcional não equivalem a cobertura completa dos requisitos de segurança.

## 2. Versões e decisões de escopo

As referências remotas foram atualizadas e os seguintes HEAD locais foram avaliados:

| Repositório | Branch | Commit completo |
|---|---|---|
| backend | main | `07c2302fea9434dfd107f803010a6d066efb548a` |
| frontend | main | `49946566b6023d36ebf738b7971ecd878e530f71` |
| cicd | main | `940f9d142e733cfdfe69f5cbd14be0a1bcd51695` |
| wiki | main | `a853aedfc9d64b8dab452e7e1b969fbbca56a10a` |

Os novos testes e este relatório são alterações locais adicionais, ainda não publicadas. Os hashes SHA-256 dos arquivos da avaliação são registrados na evidência diagnóstica para identificar exatamente a instrumentação utilizada.

O backend contém mudanças posteriores ao reconhecimento da #83, incluindo vínculos empresariais e as migrations V9–V11. Não se presumiu que os achados de 08/11 permanecessem válidos.

O prompt final fornecido pelo solicitante definiu os HEAD locais e provedores simulados como escopo. As branches `iachat-backend` e `ia-chatbot` não foram integradas nem avaliadas dinamicamente. O endpoint `/v1/support/chat` não existe no HEAD de backend avaliado.

### 2.1. Matriz administrativa utilizada

A regra de somente leitura para SUPPORT foi confirmada pelo solicitante após identificar divergência com uma definição anterior.

| Ação | USER | SUPPORT | COMPLIANCE_ANALYST | SUPER_ADMIN |
|---|---|---|---|---|
| Consultar recursos empresariais | Apenas conforme vínculo/permissão e empresa | Consulta administrativa | Consulta necessária ao compliance | Consulta administrativa |
| Aprovar/reprovar compliance | Negado | Negado | Permitido | Permitido |
| Alterar cadastro empresarial | Negado | Negado | Negado | Permitido |
| Excluir empresa | Negado | Negado | Negado | Permitido |

USER não recebe administração global. Os perfis empresariais `OWNER`, `OPERATOR` e `VIEWER` seguem as permissões implementadas e documentadas em `backend/docs/company-memberships.md`; são distintos dos níveis de administrador.

## 3. Metodologia, ambiente e evidências

### 3.1. Referências

- [OWASP WSTG 4.2](https://owasp.org/www-project-web-security-testing-guide/v42/): organização dos testes de autenticação, autorização, entradas, sessão e lógica de negócio.
- [OWASP ASVS 5.0.0](https://asvs.owasp.org/): referência de requisitos para os controles examinados. Não foi realizada verificação integral de todos os requisitos de um nível ASVS, e não se declara conformidade.
- [OWASP API Security Top 10 — 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/): API1/API3/API5 para autorização, API2 para autenticação, API4/API6 para recursos e abuso, API7 para SSRF, API8/API9 para configuração/inventário e API10 para consumo de provedores.
- [CWE](https://cwe.mitre.org/): classificação das causas demonstradas na seção 6.
- [NIST SP 800-115](https://csrc.nist.gov/pubs/sp/800/115/final): planejamento, execução controlada, registro de evidências e limitações.
- [RFC 6238, seção 5.2](https://www.rfc-editor.org/rfc/rfc6238): referência do uso único de TOTP após uma validação bem-sucedida.
- PCI DSS: não avaliado. Não foi estabelecido um escopo de dados de cartão; nenhuma conclusão de aplicabilidade ou conformidade PCI é emitida.

### 3.2. Condições

- Windows, Java Temurin 25.0.3, Gradle Wrapper 9.7.1, Node 24.16.0, JUnit, MockMvc, Vitest e PostgreSQL 16.4 em Testcontainers.
- O PostgreSQL diagnóstico é novo e efêmero. Sua URL é definida pelo container dentro da classe de teste. A limpeza SQL afeta exclusivamente esse banco, sem usar bancos existentes do desenvolvedor.
- Contas com domínio `@example.test`, empresas e beneficiários sintéticos; documentos reduzidos a bytes inertes. Os CNPJs de teste têm dígitos verificadores válidos e são usados somente na base isolada.
- JWT e segredos TOTP/webhook de teste; nenhuma credencial real foi carregada para executar operações de negócio.
- Clientes Avenia e BlindPay substituídos por mocks na integração diagnóstica. Também foi simulado o disparo do processamento BlindPay nas verificações de registro/idempotência financeira. Os testes existentes de clientes utilizam respostas simuladas.
- Máximo de cinco operações concorrentes por caso. Não houve varredura/carga em AWS, sandbox de terceiros ou produção.
- Uma reprodução com duas instâncias Java é evidência dinâmica do comportamento do serviço. Sua aplicação a múltiplos ambientes Lambda permanece uma inferência arquitetural; a AWS não foi exercitada.

### 3.3. Tipos e status

**Teste reproduzido:** código executado via JUnit/MockMvc/Testcontainers ou Vitest. **Código/arquitetura:** inspeção sem comprovação de execução no ambiente externo. Quando ambos aparecem, a conclusão delimita qual parte foi reproduzida.

**PASS:** condição específica satisfeita. **FAIL:** comportamento observado contrariou o controle esperado. **PARTIAL:** verificação limitada, funcionalidade ausente ou evidência insuficiente para concluir o requisito inteiro. Um PARTIAL não é automaticamente uma vulnerabilidade confirmada.

As evidências sanitizadas ficam no repositório backend, em [docs/security/evidencias-85](https://github.com/AGES-V-Stable/backend/tree/main/docs/security/evidencias-85). O relatório Markdown é publicado manualmente na wiki; testes, exportador e evidências são entregues em um único PR no backend. Os links abaixo ficam disponíveis na main depois do merge desse PR; antes disso, os arquivos podem ser revisados no próprio PR.

| Arquivo | Conteúdo |
|---|---|
| [baseline-backend.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/baseline-backend.json) | Resultado individual dos casos existentes do backend, incluindo seis ignorados. |
| [baseline-frontend.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/baseline-frontend.json) | Resultado individual dos 772 casos do frontend. |
| [assessment.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/assessment.json) | Resultado individual da execução diagnóstica definitiva; commits e hashes dos testes. |
| [assessment-inicial.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/assessment-inicial.json) | Snapshot da primeira entrega, de 171 casos; histórico preservado, não utilizado como resultado final. |
| [endpoints.csv](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/endpoints.csv) | Inventário obtido do mapeamento MVC real, com método, caminho, controller, handler e classificação de acesso. |

O exportador `backend/scripts/export-security-evidence.ps1` retém nomes, resultados, durações e tipos de falha. Não copia corpos HTTP, headers, tokens, mensagens de exceção, stack traces ou stdout/stderr. Os relatórios JUnit/HTML brutos permanecem em diretórios de build e não devem ser publicados sem revisão.

Em 11/11/2026, os seis arquivos de evidência foram apenas movidos para o backend, preservando seu conteúdo e as datas de captura de 10/11/2026. Essa reorganização não é uma nova execução dos testes. O exportador passou a gravar no novo diretório; seu hash nos snapshots históricos corresponde à versão usada na captura original.

Nomes de parametrizações que serializam DTOs são reduzidos a um índice com parâmetros omitidos. Cada registro também recebe evidence_id; a condição é descrita nas tabelas e no código do teste, evitando publicar payloads no nome da evidência.

O comando `gradlew.bat build --no-daemon --console=plain` também terminou com **BUILD SUCCESSFUL**, após a inclusão dos testes e da tarefa diagnóstica. A suíte funcional manteve os mesmos 1.244 casos, seis deles desabilitados na origem. O diagnóstico permanece separado e reprova os controles documentados.

A última verificação conjunta executou `gradlew.bat securityAssessmentTest build --continue --no-daemon --console=plain`: a regressão manteve zero falhas e as tarefas check/build foram concluídas. O processo conjunto retornou 1 exclusivamente pelas 19 asserções reprovadas da tarefa diagnóstica. Esse resultado não é erro de compilação, de inicialização dos testes ou uma remediação aprovada. O resumo está em [verificacao-final.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/verificacao-final.json).

## 4. Inventário de endpoints e cobertura

Foram identificadas **59 combinações de método e caminho** nos controllers ativos do backend local: quatro públicas e 55 protegidas por autenticação. Entre elas, 18 têm autorização administrativa por `@PreAuthorize`.

| Grupo | Execução |
|---|---|
| 55 rotas protegidas | Requisição sem token: 55 verificações, todas com HTTP 401. |
| Mesmas 55 rotas | JWT de desafio 2FA no lugar de sessão: 55 verificações, todas com HTTP 401. |
| 18 rotas administrativas | JWT de usuário empresarial: 18 verificações, todas com HTTP 403. |
| `POST /v1/auth/login` | Login real local e senha incorreta; casos de contrato na regressão. |
| `POST /v1/auth/login/verify-2fa` | Serviço de verificação e cadeia de segurança; replay e bloqueio examinados separadamente. |
| `POST /v1/onboarding` | Cadastro HTTP real, persistência e recusa de repetição. |
| `POST /v1/webhooks/blindpay` | Assinatura real com chave sintética, adulteração, expiração, deduplicação e eventos fora de ordem. |

Essas 128 verificações geradas pertencem a `inventoryAndAccessChecks()` e constam individualmente em `assessment.json`. A lista não inclui Swagger/OpenAPI gerados pelo framework, `/error` nem o endpoint de seed de um profile inativo. Eles não são silenciosamente contados como rotas de negócio avaliadas.

Um HTTP 401/403 comprova a barreira na condição testada; não comprova todos os comportamentos possíveis daquele endpoint após autenticação. As operações positivas e as condições de propriedade adicionais estão nas tabelas seguintes.

## 5. Casos de teste por domínio

As condições de ambiente da seção 3 aplicam-se a todas as linhas. IDs presentes no nome dos testes podem ser procurados diretamente em `assessment.json`. Para casos reaproveitados, o nome da suíte é a chave de busca nos arquivos de baseline.

### 5.1. Autenticação, TOTP e sessão

| ID do Caso | Domínio / Requisito | Endpoint / Componente | Condição e Dados Sintéticos | Resultado Esperado | Resultado Obtido | Tipo de Evidência (Teste Reproduzido vs. Código) | Status (PASS / FAIL / PARTIAL) |
|---|---|---|---|---|---|---|---|
| TC-AUTH-01 | Uso único TOTP | `TwoFactorAuthService` | Mesmo usuário, código e instância | Segunda utilização negada | Segunda chamada lança `BadCredentialsException` | Teste reproduzido | PASS |
| TC-AUTH-02 | Replay entre instâncias | Mesmo serviço | Duas instâncias compartilham conta, segredo e challenge | B reconhece consumo feito por A | B autentica novamente | Teste reproduzido em Java; extrapolação AWS é estática | FAIL |
| TC-AUTH-03 | Replay após reinício | Mesmo serviço | Instância recriada após consumir código | Consumo preservado | Nova instância aceita o mesmo código | Teste reproduzido | FAIL |
| TC-AUTH-04 | Bloqueio global | Mesmo serviço | Cinco códigos malformados em A; código válido em B | Conta bloqueada em ambas | A bloqueia; B autentica | Teste reproduzido | FAIL |
| TC-AUTH-05 | Separação de tokens | 55 rotas protegidas | JWT `purpose=2FA_CHALLENGE` | HTTP 401 em todas | 55 respostas 401 | Teste reproduzido/MockMvc | PASS |
| TC-AUTH-06 | Integridade/expiração JWT | `/v1/users/me` | JWT expirado, assinatura alterada e token sem assinatura | HTTP 401 | Rejeitados | Teste reproduzido/MockMvc | PASS |
| TC-AUTH-07 | Revogação por desativação | `/v1/users/me` | Token emitido antes de desativar a conta na base isolada | Token deixa de dar acesso | Antes: 200; depois: 401 | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTH-08 | Login primário e erro | `/v1/auth/login` | Conta e senha geradas no teste; senha incorreta em seguida | Login aceito; erro sem hash/stack trace | 200 e 401; sem os dados internos verificados | Teste reproduzido/MockMvc | PASS |
| TC-AUTH-09 | Janela de tolerância | `TotpServiceTest` | Relógio controlado; janela atual, ±1 e fora da tolerância | Aceitar apenas códigos/janelas válidos | Condições satisfeitas | Testes reproduzidos da baseline | PASS |
| TC-AUTH-10 | Reconfiguração 2FA | `TwoFactorAuthServiceTest`, `AuthControllerTest` | Conta com 2FA ativo; ausência/presença de código atual | Sem código: negar; com código válido: permitir troca | Cenários da suíte passaram | Testes reproduzidos da baseline | PASS |
| TC-AUTH-11 | Proteção do segredo em repouso | `TwoFactorSecretConverterTest` | Segredos e chaves exclusivamente de teste | Criptografar/descriptografar e rejeitar chave/conteúdo inválidos | Suíte passou | Testes reproduzidos da baseline | PASS |
| TC-AUTH-12 | TTL e armazenamento | `JwtTokenUtils`, `authToken.ts` | Sessão JWT e armazenamento web | Expiração aplicada; armazenamento identificado | TC-AUTH-16 confirma TTL real de 5h; Vitest verifica login em localStorage e onboarding em sessionStorage | Teste reproduzido + Vitest | PASS |
| TC-AUTH-13 | Logout/revogação de sessão | `authToken.ts`, filtro JWT | Remoção do token no cliente | Avaliar revogação além do cliente | Limpeza local testada; não há endpoint/lista de revogação identificado; efeito após logout não reproduzido ponta a ponta | Código + Vitest, limitado | PARTIAL |
| TC-AUTH-14 | Recuperação de acesso | `frontend/src/features/auth/api/twoFactor.ts` | Contratos de recuperação/reset | Fluxo completo verificável | Frontend declara contratos ainda inexistentes no backend; não há recuperação completa para exercitar | Código + testes de contrato do frontend | PARTIAL |
| TC-AUTH-15 | Revogação após alteração de credencial | Login e `/v1/users/me` | Atualiza hash de senha da conta sintética; tenta senha antiga e JWT anterior | Senha antiga e sessão anterior recusadas | Senha antiga:401; JWT anterior:200 | Teste reproduzido/MockMvc/PostgreSQL; mudança de senha simulada no banco | FAIL |
| TC-AUTH-16 | Expiração concreta | JWT de sessão e verificação 2FA | Compara exp/iat; challenge assinado já expirado | TTL de 18.000 segundos; challenge expirado:401 | Ambas as condições satisfeitas | Teste reproduzido | PASS |

### 5.2. Autorização e isolamento

| ID do Caso | Domínio / Requisito | Endpoint / Componente | Condição e Dados Sintéticos | Resultado Esperado | Resultado Obtido | Tipo de Evidência (Teste Reproduzido vs. Código) | Status (PASS / FAIL / PARTIAL) |
|---|---|---|---|---|---|---|---|
| TC-AUTHZ-01 | BFLA | 18 rotas administrativas | Token USER; payload válido quando exigido | HTTP 403 | 18 respostas 403 | Teste reproduzido/MockMvc | PASS |
| TC-AUTHZ-02 | Propriedade de oferta | `/v1/quotes/offers/{offerId}/confirm` | A tenta confirmar oferta de B | 404; nenhuma execução | Bloqueado antes do processamento | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTHZ-03 | Associação beneficiário/empresa | Mesmo endpoint | Oferta de A associada artificialmente a beneficiário de B | Negar a associação inconsistente | 404; provedor não acionado | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTHZ-04 | BOLA de beneficiário | Rotas por empresa/beneficiário | Troca do companyId e depois do beneficiaryId | 403 ou 404 sem exposição | 403 para empresa B; 404 para recurso B no caminho A | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTHZ-05 | BOLA financeiro e propriedade de filtro | Histórico, execução, detalhes e comprovante | Recurso real de B; JWT de A com KYC aprovado; filtro companyId adulterado | Não retornar recursos de B | Execução/detalhes/PDF: 404; histórico não contém a transação de B | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTHZ-06 | Propriedade KYC | Upload de documento por kycVerificationId | A usa verificação de representante de B | HTTP 403 antes do provedor | HTTP 403; cliente externo não chamado | Teste reproduzido/PostgreSQL | PASS |
| TC-AUTHZ-07 | Vínculo e permissão atuais | `CompanyAuthorizationIntegrationTest`, `CompanyAccessTest` | OWNER/OPERATOR/VIEWER, vínculo removido, token anterior à mudança | Respeitar vínculo atual e negar ações sem permissão | Suítes passaram | Testes reproduzidos da baseline/PostgreSQL | PASS |
| TC-AUTHZ-08 | Defesa no banco | Migrations e Terraform | Inspeção das políticas e usuário de aplicação | Identificar privilégio e isolamento efetivos | Sem políticas RLS identificadas; Terraform usa master_username; permissões reais do RDS não consultadas | Código/arquitetura | PARTIAL |
| TC-RBAC-01 | Aprovar compliance | `PATCH /v1/companies/{id}/compliance-status` | Três níveis administrativos | SUPPORT:403; demais:200 | Todos obtiveram 200 | Teste reproduzido/PostgreSQL | FAIL |
| TC-RBAC-02 | Excluir empresas | `DELETE /v1/companies/{id}` | Empresa sintética sem dependências; três níveis | Apenas SUPER_ADMIN:204 | Todos obtiveram 204 | Teste reproduzido/PostgreSQL | FAIL |
| TC-RBAC-03 | Alterar cadastro | `PUT /v1/companies/{id}` | Cadastro sintético válido; três níveis | Apenas SUPER_ADMIN:200 | Todos obtiveram 200 | Teste reproduzido/PostgreSQL | FAIL |

### 5.3. Entradas, contratos, exposição e recursos

| ID do Caso | Domínio / Requisito | Endpoint / Componente | Condição e Dados Sintéticos | Resultado Esperado | Resultado Obtido | Tipo de Evidência (Teste Reproduzido vs. Código) | Status (PASS / FAIL / PARTIAL) |
|---|---|---|---|---|---|---|---|
| TC-API-01 | Autenticação na superfície da API | 55 rotas protegidas | Sem token | HTTP 401 | Todas negadas | Teste reproduzido/MockMvc | PASS |
| TC-API-02 | Validação monetária de entrada | Confirmação de oferta | Zero, negativo, abaixo de 0,01 e null | HTTP 400 | Quatro casos negados | Teste reproduzido/MockMvc | PASS |
| TC-API-03 | Entrada de identificador | `GET /v1/companies/{id}` | String inerte com sintaxe SQL no lugar de UUID | 400; dados preservados | 400 e contagem de empresas inalterada | Teste reproduzido/PostgreSQL | PASS |
| TC-API-04 | Cadastro/repetição | `/v1/onboarding` | Mesmo cadastro sintético duas vezes | 201; depois 409, sem duplicação | Uma empresa, usuário e fluxo de cadastro; repetição recusada | Teste reproduzido/PostgreSQL | PASS |
| TC-API-05 | Validação estrutural de arquivo | `BlindPayComplianceService.uploadDocument` | Apenas prefixos PDF/PNG/JPEG, sem documento válido | Recusar antes do upload | Três conteúdos aceitos e enviados ao mock | Teste reproduzido de serviço | FAIL |
| TC-API-06 | Limites e tipos de upload | `BlindPayComplianceServiceTest` | Arquivo vazio, tipo não aceito, mais de 3 MB, imagem/documento incompatível | Recusar entradas inválidas | Suíte passou para esses cenários | Testes reproduzidos da baseline | PASS |
| TC-API-07 | Tratamento de erro | Handlers e cadeia HTTP | JSON inválido, erro interno sintético, erro de provedor | Resposta controlada sem stack trace | Casos de erro e logs sanitizados passaram | Testes reproduzidos da baseline + TC-AUTH-08/TC-INT-05 | PASS |
| TC-API-08 | Validação de respostas no frontend | Adapters Zod, httpClient e login | Contratos incorretos, 401, redirecionamento externo | Rejeitar contratos inválidos e destino externo; limpar token rejeitado | Cenários Vitest passaram | Testes reproduzidos da baseline | PASS |
| TC-API-09 | Rate limiting e consumo | Login, onboarding, quotes; API Gateway IaC | Busca por limitadores e limites de execução | Identificar controle e limite aplicáveis | Não identificado limitador nas três rotas; apenas bloqueio TOTP local. Não se determinou um orçamento seguro global de requisições/custos | Código/arquitetura; sem teste de saturação | PARTIAL |
| TC-API-10 | SSRF e navegação de rotas externas | AveniaClient/BlindPayClient | Origens configuradas e IDs recebidos | Não transformar ID em destino arbitrário | Teste de escaping de path da BlindPay passou; inspeção limitada das origens fixas. Não prova ausência geral de SSRF | Teste de cliente + código | PARTIAL |
| TC-API-11 | Upload na borda HTTP | Multipart de compliance | Configuração 3 MB/4 MB e TC-API-13 em servidor real | Limites efetivos em HTTP, sem gateway externo | Arquivo vazio:400; acima de 3 MB por arquivo ou 4 MB por request:413 | Teste reproduzido HTTP/PostgreSQL local | PASS |
| TC-API-12 | Chatbot | `/v1/support/chat` | HEAD local | Verificar controles se implementados | Controller e cliente ausentes nessa versão; sem teste dinâmico do chatbot | Inventário/código | PARTIAL |
| TC-API-13 | Parser multipart real | `/v1/blindpay/compliance/documents` | Tomcat em porta efêmera, JWT real e arquivo de 0, 3 MB+1 e 4 MB+1 bytes | 400/413 sem acionar provedor | Três condições HTTP verificadas fora de MockMvc | Teste reproduzido de rede local | PASS |

### 5.4. Integridade financeira, idempotência e estados

| ID do Caso | Domínio / Requisito | Endpoint / Componente | Condição e Dados Sintéticos | Resultado Esperado | Resultado Obtido | Tipo de Evidência (Teste Reproduzido vs. Código) | Status (PASS / FAIL / PARTIAL) |
|---|---|---|---|---|---|---|---|
| TC-FIN-01 | Segundo fator em oferta | `/v1/quotes/offers/{offerId}/confirm` | Usuário sem 2FA ativo, payload sem TOTP | Negar antes de registrar/submeter | HTTP 200; execução e transação registradas; caminho de submissão simulada alcançado | Teste reproduzido/MockMvc/PostgreSQL | FAIL |
| TC-FIN-02 | Validade temporal | Mesmo endpoint | expiresAt passado | 422; não iniciar processamento | 422; mock de processamento não chamado | Teste reproduzido/PostgreSQL | PASS |
| TC-FIN-03 | Manipulação de valor | Mesmo endpoint | Valor menor, maior, fração adicional e magnitude elevada | 422 por divergência da oferta | Quatro rejeições antes do processamento | Teste reproduzido/PostgreSQL | PASS |
| TC-FIN-04 | TOTP em rotas alternativas | `/v1/transfers` e confirmação de recotação | Comparação dos fluxos e TC-FIN-20 | Mesmo requisito de autenticação forte | Rota direta valida TOTP; recotação sem TOTP chega a COMPLETED com payout simulado | Testes existentes + reprodução HTTP/PostgreSQL | FAIL |
| TC-FIN-05 | Comparação de ofertas, taxas e valores fixos | `QuoteOrchestratorServiceTest`, `BlindPayQuoteProviderTest` | SOURCE/TARGET fixos; oferta expirada; valores nulos/zero/negativos de provedor; tarifas em moedas diferentes | Seleção coerente e rejeição de oferta inválida | Cenários das suítes passaram | Testes reproduzidos da baseline | PASS |
| TC-FIN-06 | Moedas/rotas e casas decimais | Provedores e entidades monetárias | Matriz de TC-FIN-13 a 19: fiat com 2 casas, cripto com 8, centavos, overflow e rota inválida | Preservar valores e rejeitar entradas incompatíveis | Unidades e rotas inválidas verificadas; perda de precisão cripto reproduzida | Testes reproduzidos/PostgreSQL e adapters | FAIL |
| TC-FIN-07 | Consistência persistida | Execução, transação e ticket Avenia simulado | Código real até o cliente externo substituído; compara snapshot da oferta/execução com histórico persistido | Mesma moeda, propriedade e quantidade em cada representação | Fiat consistente; cripto arredondado no histórico, com execução ainda em 8 casas | Teste reproduzido/PostgreSQL, ticket simulado | FAIL |
| TC-FIN-08 | Estados e repetições | `BlindPayExecutionProcessorTest`, `TransferExecutionServiceTest` | Oferta repetida, recotação sem pendência, payout incerto, refund/failure, evento duplicado | Não duplicar payout nem avançar indevidamente nos casos definidos | Suítes passaram; repetição sequencial da oferta devolve execução existente | Testes reproduzidos da baseline | PASS |
| TC-FIN-09 | Imutabilidade após conclusão | Processador real, confirmação e banco | TC-FIN-15 em COMPLETED/FAILED/REFUNDED; start, reconcile, acceptRequote, replay e alteração de valores | Não reabrir nem alterar o registro pelos caminhos exercitados | Snapshot integral da execução preservado; recotação recusada e disparo não repetido | Teste reproduzido/MockMvc/PostgreSQL | PASS |
| TC-FIN-10 | Tarifas Avenia inválidas | `AveniaTransferServiceTest` | Taxa nula/incompleta, negativa, maior que inputAmount ou em outra moeda | Rejeitar antes de criar ticket | Casos parametrizados, markup e appliedFees passaram | Testes reproduzidos da baseline | PASS |
| TC-FIN-11 | Integridade de resposta Avenia | Mesmo componente | Moedas/métodos diferentes do pedido, preço base zero/nulo, valores não positivos e quoteToken ausente | Rejeitar resposta inconsistente | Casos da suíte passaram; nenhum ticket nos casos bloqueados | Testes reproduzidos da baseline | PASS |
| TC-FIN-12 | Proteção de preço na execução | `TransferExecutionServiceTest`, `AveniaTransferServiceTest` | Cotação nova pior e feeSettlement diferente do solicitado | Não criar ticket nem aceitar preço pior silenciosamente | Guard e validação de modo de taxa passaram | Testes reproduzidos da baseline | PASS |
| TC-FIN-13 | Persistência fiat | Confirmação real e duas representações no banco | USD/EUR/BRL, montantes fracionários de 2 casas e beneficiário sintético | Moeda, montantes e beneficiário consistentes | Três combinações aprovadas | Teste reproduzido/PostgreSQL | PASS |
| TC-FIN-14 | Persistência cripto | Fluxo Avenia até ticket simulado | BRL/PIX para USDC/POLYGON, USDT/ETHEREUM e BRLA/POLYGON; destino com 8 casas | Quantidade confirmada igual à armazenada no histórico | Três valores arredondados para 2 casas no histórico | Teste reproduzido/MockMvc/PostgreSQL | FAIL |
| TC-FIN-15 | Estados terminais | `BlindPayExecutionProcessor` real e endpoint de confirmação | COMPLETED, FAILED e REFUNDED | start/reconcile não reabrem; recotação negada; replay não altera; valores adulterados negados | Três estados protegidos nos caminhos exercitados | Teste reproduzido/PostgreSQL | PASS |
| TC-FIN-16 | Centavos e lado de tarifa | `BlindPayQuoteProvider` real | USDC/USDT/USDB; SOURCE/TARGET; fronteira de 500 centavos e frações exatas | requestAmount inteiro exato e coverFees coerente | Seis combinações aprovadas; composição de tarifa e taxa efetiva verificadas | Teste reproduzido de adapter | PASS |
| TC-FIN-17 | Casas excedentes e overflow | Mesmo adapter | 5.001, 5.009 e dois valores fora do intervalo de long em centavos | Não arredondar silenciosamente nem chamar gateway | Quatro entradas recusadas antes do gateway; ArithmeticException interna | Teste reproduzido de adapter; status HTTP desses erros não afirmado | PASS |
| TC-FIN-18 | Moeda/rede incompatível | Mesmo adapter | BTC, EURC, rede desconhecida, origem não correspondente ao token, destino divergente do beneficiário | Não selecionar provedor para rota incompatível | Cinco combinações negadas por supports, sem gateway | Teste reproduzido de adapter | PASS |
| TC-FIN-19 | Sinal de tarifa externa | Mesmo adapter | Tarifas de 0, 1, 123 e -1 centavos | Aceitar zero/positivas; rejeitar negativa | Três controles positivos aprovados; tarifa negativa aceita | Teste reproduzido de adapter | FAIL |
| TC-FIN-20 | Segundo fator na recotação | `/v1/transfer-executions/{id}/requote/confirm` | Estado REQUOTE_REQUIRED e cotação vigente; usuário sem 2FA; processador real por delegação e gateway simulado | 422 antes de enviar payout | HTTP 200, uma chamada de payout e estado COMPLETED no PostgreSQL | Teste reproduzido HTTP/serviço/PostgreSQL; sem liquidação externa | FAIL |
| TC-CONC-01 | TOCTOU no segundo fator | `validateTransferTotp` / `consumeTransferTotp` | Cinco operações com barreira entre validar e consumir | No máximo uma autorização por TOTP | As cinco validam e consomem sem erro | Teste determinístico de serviço; sem cinco transferências reais | FAIL |
| TC-CONC-02 | Idempotência concorrente | Confirmação HTTP de oferta | Cinco requisições simultâneas contra PostgreSQL | Mesmo executionId, uma execução/transação e um disparo | Cinco HTTP 200; um executionId, uma linha de cada recurso e um start no mock | Teste reproduzido/MockMvc/PostgreSQL | PASS |

### 5.5. Integrações e webhooks

| ID do Caso | Domínio / Requisito | Endpoint / Componente | Condição e Dados Sintéticos | Resultado Esperado | Resultado Obtido | Tipo de Evidência (Teste Reproduzido vs. Código) | Status (PASS / FAIL / PARTIAL) |
|---|---|---|---|---|---|---|---|
| TC-INT-01 | Autenticidade e deduplicação | `/v1/webhooks/blindpay` | Mesmo payin.complete, id e HMAC válido duas vezes | Processar uma vez | 204 duas vezes; um evento persistido e uma chamada ao processador simulado | Teste reproduzido/MockMvc/PostgreSQL | PASS |
| TC-INT-02 | Integridade e replay temporal | Mesmo endpoint | Corpo alterado após assinatura, headers ausentes, timestamp antigo | 401; nenhum evento persistido | Três rejeições; nenhuma linha | Teste reproduzido/MockMvc/PostgreSQL | PASS |
| TC-INT-03 | Evento fora de ordem | Mesmo endpoint, customer.update | Aprovação nova seguida de rejeição mais antiga | Não regredir estado | kyc_status aprovado preservado | Teste reproduzido/PostgreSQL | PASS |
| TC-INT-04 | Vinculação externa/empresa | Mesmo endpoint | HMAC válido, cliente de A, external_id de B | Ignorar associação divergente | Conta permaneceu PENDING | Teste reproduzido/PostgreSQL | PASS |
| TC-INT-05 | Parsing e transação | Mesmo endpoint | JSON malformado com HMAC válido | 400, sem registro persistido | 400 e zero eventos | Teste reproduzido/PostgreSQL | PASS |
| TC-INT-06 | Algoritmo/janela/rotação | `BlindPayWebhookVerifierTest` | Vetor de referência, assinatura diferente, timestamps ±300/±301, versões inválidas | Aceitar só condições válidas | Suíte passou | Testes reproduzidos da baseline | PASS |
| TC-INT-07 | Consumo de respostas e credenciais | `AveniaClientTest`, `BlindPayClientTest`, testes de logs | Resposta vazia/não JSON, falha de comunicação, erro de autenticação e payloads sintéticos | Tratar erro sem expor dados; preservar contratos de autenticação | Suítes passaram com servidor/respostas simulados | Testes reproduzidos da baseline | PASS |
| TC-INT-08 | Timeout e repetição segura | Clientes, agregador e processador | TC-INT-10/11/12 e casos de submissão incerta da baseline | Encerrar espera e evitar repetição cega de liquidação | Transporte/HTTP/timeout local exercitados; reconciliação/idempotência verificadas em simulação | Testes reproduzidos; comportamento de terceiros excluído | PASS |
| TC-INT-09 | Webhook/reconciliação Avenia | Controllers e `TransferService` | Inventário e cenário de timeout com ticket incerto | Reconciliação rastreável | Caminho RECONCILIATION_REQUIRED testado; receptor webhook Avenia e reconciliação automática não identificados | Testes de serviço + código; sem provedor real | PARTIAL |
| TC-INT-10 | Falha de transporte | Clientes Avenia/BlindPay reais com MockRestServiceServer | Timeout e erro TLS simulados | Exceção de integração controlada; sem retry automático ilimitado | Quatro casos tratados como retryable; uma tentativa por chamada | Teste reproduzido de cliente; não mede handshake TLS real | PASS |
| TC-INT-11 | Resposta transitória | Mesmos clientes | HTTP 429/500/503 | Exceção retryable com número limitado de chamadas | Seis casos tratados; uma tentativa por chamada | Teste reproduzido de cliente | PASS |
| TC-INT-12 | Deadline de leitura | Mesmos clientes com servidor em 127.0.0.1 e porta efêmera | Servidor retém resposta até liberação por latch; readTimeout de teste de 200 ms | Timeout controlado e encerramento do cliente | Ambos os clientes encerraram a espera; servidor confirmou recebimento | Teste reproduzido de transporte local; não afirma timeout da configuração de produção | PASS |
| TC-INT-13 | Reconciliação após falha | `TransferExecutionReconcilerTest` e testes dos dois processadores | Resposta perdida; mesma chave; payout incerto; falha em uma execução do lote | Reusar chave quando seguro, encaminhar incerteza para conciliação e continuar o lote | Casos da baseline passaram, incluindo Avenia sem repetição de ticket incerto | Testes reproduzidos de serviço | PASS |

## 6. Achados reproduzíveis e recomendações

Severidades abaixo são prioridades qualitativas preliminares, sem atribuição de CVSS/EPSS ou estimativas financeiras. A consolidação e a remediação pertencem às etapas correspondentes do épico.

### SEC85-01 — Confirmação de oferta aceita sem TOTP

**Prioridade:** alta. **Referência anterior:** VULN-01. **Classificação:** CWE-287; OWASP API2:2023 e API6:2023.

**Código:** [TransferExecutionController.java:41](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/controller/TransferExecutionController.java#L41), [TransferExecutionService.java:115](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/transfer/TransferExecutionService.java#L115), [TransferService.java:165](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TransferService.java#L165). A rota direta valida TOTP em [TransferService.java:131](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TransferService.java#L131).

**Reprodução segura:** executar `ApplicationSecurityAssessmentTest.offerConfirmationMustRequireTotp` pela tarefa diagnóstica. O teste cria um usuário com 2FA desativado, vínculo OWNER, beneficiário próprio e oferta sintética válida; envia a confirmação com os dois valores, sem código TOTP.

```text
TC-FIN-01
Esperado: HTTP 422 antes de registrar/submeter.
Obtido:   HTTP 200.
Autenticacao: Bearer [REDACTED]
```

Na confirmação inicial, o registro local alcança o ponto de disparo ao processador simulado. A recotação também foi reproduzida em TC-FIN-20: a chamada do bean substituído delega a acceptRequote do processador real, com transações e PostgreSQL reais; apenas o provisionamento da carteira e a resposta de payout são simulados. Sem TOTP, houve uma chamada de payout e o estado chegou a COMPLETED, em vez de rejeitar a operação com 422. Não houve transferência/liquidação em provedor real.

**Recomendação:** aplicar a mesma autorização forte em todas as entradas de execução e recotação, antes de efeitos externos, com consumo atômico e sem eliminar a idempotência de requisições já autorizadas.

### SEC85-02 — Anti-replay e bloqueio TOTP não compartilhados

**Prioridade:** alta. **Referência anterior:** VULN-02. **Classificação:** CWE-294 e CWE-307; OWASP API2:2023.

**Código:** [TwoFactorAuthService.java:38](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L38), [verificação:226](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L226) e [bloqueio:257](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L257).

**Reprodução segura:** `TwoFactorStateAssessmentTest`, TC-AUTH-02/03/04. As instâncias usam a mesma conta sintética, segredo e challenge; a validação TOTP e a assinatura do challenge são reais. Os acessos à conta são simulados.

```text
TC-AUTH-02: A consome o TOTP; B aceita novamente. Excecao esperada nao lancada.
TC-AUTH-03: nova instancia aceita TOTP consumido antes da recriacao.
TC-AUTH-04: cinco erros bloqueiam A; B aceita o codigo correto.
TOTP, segredo e challenge: [REDACTED]
```

O cenário foi reproduzido em instâncias Java locais. A conclusão de que ambientes Lambda separados também não compartilham esses mapas é arquitetural. Não foi realizado teste na AWS nem medido o roteamento de requisições entre ambientes Lambda.

**Recomendação:** persistir o consumo da janela temporal e o bloqueio por conta em armazenamento compartilhado, com atualização atômica. Definir também o identificador inequívoco da conta/tipo de conta e o tratamento de troca do segredo. Evitar registrar indefinidamente apenas os seis dígitos, pois podem se repetir em janelas futuras.

### SEC85-03 — Níveis administrativos recebem permissões irrestritas

**Prioridade:** alta. **Referência anterior:** VULN-06. **Classificação:** CWE-863; OWASP API5:2023.

**Código:** [AdministratorEntity.java:78](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/entity/AdministratorEntity.java#L78) retorna `ROLE_ADMIN` para todos os níveis. Em CompanyController, os métodos de [compliance:118](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/controller/CompanyController.java#L118), [edição:135](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/controller/CompanyController.java#L135) e [exclusão:156](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/controller/CompanyController.java#L156) usam a mesma autoridade para operações de poderes diferentes.

**Reprodução segura:** TC-RBAC-01/02/03 em `ApplicationSecurityAssessmentTest`. São administradores reais da base efêmera, com JWT emitido pela aplicação. Não se usa `@WithMockUser` para forjar essas permissões.

| Operação indevida | Perfil | Esperado | Obtido |
|---|---|---:|---:|
| Aprovar compliance | SUPPORT | 403 | 200 |
| Excluir empresa sintética sem dependências | SUPPORT | 403 | 204 |
| Excluir empresa sintética sem dependências | COMPLIANCE_ANALYST | 403 | 204 |
| Alterar cadastro | SUPPORT | 403 | 200 |
| Alterar cadastro | COMPLIANCE_ANALYST | 403 | 200 |

**Recomendação:** mapear autoridades/permissões por accessLevel e autorizar cada ação conforme a matriz da seção 2.1. Manter os testes negativos e positivos e verificar que alterações de privilégio valem para tokens existentes. A exclusão de empresas com histórico financeiro exige política própria; não foi demonstrada remoção em cascata de toda a base financeira.

### SEC85-04 — Validação e consumo de TOTP permitem intercalamento

**Prioridade:** alta para investigação do fluxo financeiro. **Classificação:** CWE-367 e CWE-362; OWASP API2:2023/API6:2023.

**Código:** [TwoFactorAuthService.java:187](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L187), [consumo:208](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L208) e intervalo entre [TransferService.java:131](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TransferService.java#L131) e linha 147.

**Reprodução segura:** TC-CONC-01, `validationAndConsumptionMustBeAtomicAcrossFiveOperations`. Cinco threads validam o mesmo TOTP; uma barreira garante que todas concluam a checagem antes de consumir. Não se usa sleep e não são enviadas transferências.

```text
Operacoes que validaram e consumiram o mesmo TOTP:
Esperado: no maximo 1.
Obtido: 5.
```

O teste demonstra a separação sem exclusão entre as fases do serviço. Não demonstra cinco pagamentos efetivos: limitações posteriores e idempotência por oferta podem impedir duplicação. O TC-CONC-02, de outra camada, comprovou a unicidade da mesma oferta no PostgreSQL.

**Recomendação:** reservar/consumir atomicamente a autorização antes do efeito crítico, vinculando-a à operação/idempotência. Tratar falhas incertas sem liberar uma autorização que já possa ter produzido efeito externo.

### SEC85-05 — Prefixos de arquivo aceitos como documentos completos

**Prioridade:** média para endurecimento de validação. **Classificação:** [CWE-1286](https://cwe.mitre.org/data/definitions/1286.html); OWASP API8:2023/API10:2023.

**Código:** [BlindPayComplianceService.java:128](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/blindpay/BlindPayComplianceService.java#L128), [detecção:699](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/blindpay/BlindPayComplianceService.java#L699).

**Reprodução segura:** TC-API-05, `UploadContentAssessmentTest`. Conta sintética com termos aceitos; arquivo de quatro bytes `%PDF`, assinatura PNG de oito bytes e prefixo JPEG de três bytes. Esses conteúdos são inertes e não constituem documentos completos.

```text
PDF / PNG / JPEG truncados:
Esperado: IllegalArgumentException antes do envio.
Obtido: retorno de URL do cliente simulado, sem rejeicao estrutural.
```

A validação detecta o prefixo, mas não comprova a estrutura do documento. O efeito demonstrado é encaminhamento de arquivo inválido ao mock. Não houve comprovação de malware, execução de script, armazenamento público ou bypass da validação que o provedor real possa fazer; não se classifica como RCE.

**Recomendação:** validar a estrutura com parsers adequados e limites de recursos, considerando processamento isolado. Manter os limites de tamanho e nomes genéricos já existentes. Definir a política para conteúdo ativo em PDFs e verificação adicional antes de disponibilizar arquivos.

### SEC85-06 — Sessão continua válida após alteração do hash de senha

**Prioridade:** alta para revogação de credenciais comprometidas. **Classificação:** [CWE-1464](https://cwe.mitre.org/data/definitions/1464.html); OWASP API2:2023. Foi utilizado o identificador atual de invalidação de sessão; CWE-613 foi descontinuado na versão 5.0 do CWE.

**Código:** [JwtTokenUtils.java:140](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/utils/JwtTokenUtils.java#L140) verifica identidade/expiração/finalidade, sem comparar versão de credencial ou instante de revogação. O [JwtTokenFilter.java:78](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/configuration/security/JwtTokenFilter.java#L78) utiliza essa validação após carregar a conta.

**Reprodução segura:** TC-AUTH-15 em `ApplicationSecurityAssessmentTest.passwordChangeMustRevokeExistingSession`. Uma conta sintética recebe JWT. O teste altera seu hash de senha somente no PostgreSQL efêmero. A senha anterior é recusada pelo login real, comprovando que a credencial mudou, mas o JWT anterior ainda acessa `/v1/users/me`.

```text
Login com senha anterior: HTTP 401.
Acesso com JWT anterior: esperado HTTP 401; obtido HTTP 200.
Senha, hash e JWT: [REDACTED]
```

Não existe endpoint de troca/recuperação de senha no HEAD para executar esse evento pela interface pública. O teste demonstra o comportamento do token diante da alteração de credencial persistida, sem afirmar que um fluxo de recuperação inexistente foi executado. A desativação da conta continua revogando acesso, conforme TC-AUTH-07.

**Recomendação:** definir a política de revogação por troca/recuperação de credencial e aplicá-la no servidor, por versão de sessão/credencial ou instante de invalidação. Ao implementar logout global e recuperação, verificar que tokens anteriores deixam de funcionar.

### SEC85-07 — Histórico financeiro perde precisão de montantes cripto

**Prioridade:** alta para integridade e conciliação. **Classificação indicativa:** [CWE-681](https://cwe.mitre.org/data/definitions/681.html); OWASP API8:2023/API10:2023.

**Código:** [BaseTransactionEntity.java:48](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/entity/BaseTransactionEntity.java#L48) e `V1__init.sql:153` definem foreign_amount com escala 2. [TransferExecutionEntity.java:62](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/entity/TransferExecutionEntity.java#L62) preserva 8 casas no destino. [TransferService.java:202](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TransferService.java#L202) transfere a quantidade da resposta para a entidade, que o banco arredonda.

**Reprodução segura:** TC-FIN-14. Cria-se oferta própria de BRL/PIX para carteira sintética com subconta vinculada. O fluxo HTTP, serviços, transações e PostgreSQL são reais; apenas AveniaClient.createQuote/createTicket são simulados. O teste verifica uma chamada de ticket, preservação de transfer_executions.target_amount e igualdade com base_transactions.foreign_amount.

| Moeda sintética | Quantidade confirmada e mantida na execução | Quantidade gravada no histórico |
|---|---:|---:|
| USDC | 19.12345678 | 19.12 |
| USDT | 19.87654321 | 19.88 |
| BRLA | 99.12345678 | 99.12 |

São valores de teste, sem saldo ou pagamento real. O efeito comprovado é divergência entre duas representações persistidas da mesma operação, após um ticket simulado. Não foi medido prejuízo financeiro nem liquidação em terceiros.

**Recomendação:** definir escalas por ativo e preservar a precisão necessária no registro financeiro; alterar schema por nova migration, alinhando DTOs, histórico e comprovantes. Arredondamento de apresentação não deve substituir a quantidade contábil. Aplicar os mesmos critérios às demais colunas monetárias e taxas.

### SEC85-08 — Adapter BlindPay aceita tarifa negativa recebida do provedor

**Prioridade:** média para validação defensiva de respostas. **Classificação:** [CWE-1284](https://cwe.mitre.org/data/definitions/1284.html); OWASP API10:2023.

**Código:** [BlindPayQuoteProvider.java:96](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/quote/BlindPayQuoteProvider.java#L96), [payoutFees:253](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/quote/BlindPayQuoteProvider.java#L253) e [sum:299](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/quote/BlindPayQuoteProvider.java#L299) somam tarifas sem rejeitar valores negativos.

**Reprodução segura:** TC-FIN-19 em `FinancialBoundaryAssessmentTest.providerFeeMustNotBeNegative`. O adapter é real; gateway e provisionamento são mocks. A resposta sintética tem montantes positivos e flatFee=-1 centavo. O adapter devolve tarifa total negativa em vez da exceção esperada. Os controles de tarifa zero e positiva passaram.

```text
Tarifa sintética recebida: -1 centavo.
Esperado: rejeitar a resposta externa inconsistente.
Obtido: ProviderQuoteResult aceito, com tarifa total negativa.
```

Não se afirma que a BlindPay real emita essa resposta, que um usuário consiga controlar o provedor nem que uma transferência tenha sido liquidada com esse valor. A falha demonstrada é falta de validação defensiva no adapter.

**Recomendação:** validar sinal, escala, moeda e coerência de cada parcela de tarifa antes de selecionar/persistir a oferta. Eventuais descontos ou rebates devem ter representação e regra explícitas, em vez de serem aceitos implicitamente como tarifas negativas.

## 7. Revisão dos apontamentos VULN-01 a VULN-11 da #83

| ID anterior | Resultado nesta avaliação | Evidência e limite |
|---|---|---|
| VULN-01 | Confirmado para confirmação inicial e recotação local | SEC85-01, TC-FIN-01/20; recotação até estado COMPLETED com payout simulado, sem movimentação real. |
| VULN-02 | Confirmado no serviço local | SEC85-02; comportamento AWS não executado. |
| VULN-03 | Observação estática mantida em parte | Allowlist de `LambdaSecretsEnvironmentPostProcessor.java:24` não inclui TWO_FACTOR_ENCRYPTION_KEY; nenhum deploy Lambda foi realizado. AI_SUPPORT_API_KEY não é requisito funcional do HEAD sem chatbot. |
| VULN-04 | Não revalidado | Exposição de credencial em issue pública foi herdada do planejamento; a credencial não foi consultada, testada nem reproduzida aqui. Gestão de segredos fica na análise correspondente do épico. |
| VULN-05 | Não comprovado como BOLA geral | Não se identificou RLS nas migrations; Terraform liga a aplicação ao master_username. Isso aumenta dependência dos controles da aplicação e privilégio potencial, mas não comprova acesso cruzado. Os testes horizontais executados passaram. |
| VULN-06 | Confirmado dinamicamente | SEC85-03, conforme matriz aprovada pelo solicitante. |
| VULN-07 | Observação de IaC | `database/main.tf:23` define Single-AZ, linha 24 backup_retention_period=0 e linha 27 skip_final_snapshot=true. Não comprova a configuração efetivamente aplicada na AWS nem ausência de qualquer backup externo. |
| VULN-08 | Revogação após alteração persistida de senha reproduzida; demais aspectos estáticos | SEC85-06 e TC-AUTH-16. Armazenamento e TTL identificados. Não foi demonstrado XSS, furto de sessão ou exploração de headers de CloudFront. |
| VULN-09 | Parcial | Ausência de limitador identificada nas rotas de login/onboarding/quotes e de throttling explícito no stage IaC. Não foram testados limites efetivos da AWS ou saturação. Chatbot ausente no HEAD. |
| VULN-10 | Fora da execução dinâmica desta issue | Este relatório não substitui SAST/SCA/secret scanning/IaC da #84; nenhuma conclusão de CVEs ou segurança integral do pipeline. |
| VULN-11 | Parcial | Estado incerto Avenia testado por simulação; receptor webhook e atualização automática Avenia não identificados. Não se provocou timeout real nem perda de webhook em provedor. |

## 8. Reprodução e roteiro manual

### 8.1. Comandos locais

Pré-requisitos: Java 25 e Docker Desktop em execução; dependências do projeto disponíveis. A tarefa `securityAssessmentTest` não utiliza sandbox externo e configura origens de fallback em loopback.

```powershell
# A partir de backend/
.\gradlew.bat securityAssessmentTest --no-daemon --console=plain

# Um grupo por vez; as classes têm tag security-assessment.
.\gradlew.bat securityAssessmentTest --tests '*TwoFactorStateAssessmentTest' --no-daemon
.\gradlew.bat securityAssessmentTest --tests '*ApplicationSecurityAssessmentTest' --no-daemon
.\gradlew.bat securityAssessmentTest --tests '*UploadContentAssessmentTest' --no-daemon
.\gradlew.bat securityAssessmentTest --tests '*FinancialBoundaryAssessmentTest' --no-daemon
.\gradlew.bat securityAssessmentTest --tests '*ProviderFailureAssessmentTest' --no-daemon

# Regressão/build: diagnóstico separado, sem esconder a situação de segurança.
.\gradlew.bat build --no-daemon --console=plain

# Exporta metadados sanitizados para backend/docs/security/evidencias-85/.
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\export-security-evidence.ps1 -Phase assessment
```

```powershell
# A partir de frontend/; dependências fixadas no package-lock.json.
npm ci --ignore-scripts --no-audit --no-fund
npm run test:coverage -- --maxWorkers=2 --reporter=default --reporter=junit --outputFile.junit=coverage/security-baseline.xml
```

O uso de ExecutionPolicy Bypass limita-se ao processo que executa o exportador e não altera a política global do Windows. Esse exportador não faz requisições às APIs.

**Resultado esperado na versão vulnerável:** a tarefa diagnóstica termina com código de saída diferente de zero pelos 19 FAIL documentados. Não use `ignoreFailures` para apresentar o diagnóstico como aprovado. A tarefa não faz parte de `build/check` nem do PIT; a suíte funcional permanece separada para distinguir regressão e diagnóstico. A integração dos testes corrigidos ao gate deve ser tratada na remediação.

Execuções filtradas sobrescrevem os XMLs daquela tarefa; para regenerar o conjunto completo de evidências, rode a tarefa sem `--tests` antes de exportar. Arquivos exportados de execuções antigas não devem ser confundidos com reteste de uma correção.

### 8.2. Conferência manual recomendada

O navegador não substitui os testes de API: ocultar um botão não impede chamadas diretas. Para revisão pelo responsável, o caminho mais seguro é rodar os grupos acima e abrir `backend/build/reports/tests/securityAssessmentTest/index.html`, acompanhando estes casos:

1. TC-AUTH-02: primeira e segunda instâncias recebem o mesmo código; observar a ausência da rejeição na segunda.
2. TC-RBAC-01/02/03: comparar os status HTTP com a matriz de permissões, exclusivamente na base efêmera.
3. TC-FIN-01 e TC-CONC-02: distinguir falta de TOTP da unicidade de execução; um controle não substitui o outro.
4. TC-INT-01/02: acompanhar a aceitação de assinatura válida e a rejeição do corpo adulterado, sem cadastrar webhooks em terceiros.
5. TC-API-05: verificar que os arquivos contêm apenas prefixos e que o retorno é de um cliente simulado.

Essas conferências pelo responsável ainda **não foram realizadas** e não são registradas como evidência manual executada. Não é necessário usar dados pessoais, configurar credenciais reais ou transferir dinheiro para reproduzir os testes.

## 9. Limitações e riscos residuais

1. **Camadas simuladas:** mocks não comprovam os contratos atuais, assinaturas aceitas ou efeitos reais das APIs de terceiros. Não houve chamada de negócio à Avenia/BlindPay, AWS ou chatbot.
2. **Concorrência:** cinco threads/requests e uma interlevação determinística não esgotam condições de corrida. A deduplicação demonstrada é de uma oferta e um disparo local; não é uma prova geral de liquidação única distribuída.
3. **Recotação:** fluxo local até COMPLETED reproduzido com transações reais e resposta de payout simulada. Não se afirma que houve liquidação em sandbox ou produção.
4. **Valores/moedas/tarifas:** a matriz TC-FIN-13–19 cobre casos representativos, fronteiras de centavos/long, sinal de tarifas e persistência fiat/cripto. Os defeitos encontrados são FAIL, não ausência de teste. Novos ativos/regras de arredondamento ou limites de negócio demandam manutenção da matriz; não se afirma cobertura de todas as combinações matematicamente possíveis.
5. **Imutabilidade:** COMPLETED/FAILED/REFUNDED foram exercitados em confirmação, recotação, start e reconciliação com comparação integral do registro de execução. Isso não é uma proteção contra escrita SQL direta por um administrador de banco nem comprova comportamento interno de terceiros.
6. **Recuperação/sessão:** recuperação, logout no servidor e troca de senha via endpoint não existem no HEAD. Limpeza do cliente é coberta pelo Vitest; a alteração de senha foi simulada no banco isolado e demonstrou sessão antiga ainda aceita. A lacuna de revogação está registrada em SEC85-06.
7. **Uploads:** serviço e limites reais do parser HTTP exercitados. Antivírus, PDFs ativos e documentos pessoais não foram usados; o teste de documento truncado não comprova execução de conteúdo malicioso.
8. **SSRF/injeção/abuso:** teste de UUID, escaping de path e revisão de origens não constituem exploração completa de toda a superfície. Nenhum stress/DoS foi feito; ausência de rate limiter no código não prova ausência de qualquer controle externo.
9. **Ignorados preexistentes:** `OnboardingIntegrationTest` possui dois casos desabilitados e `TransactionQueryIntegrationTest` quatro. Isso é configuração `@Disabled`, não falha do Docker nesta execução. Casos novos cobrem cadastro e isolamento do histórico, sem alegar reativação ou equivalência integral aos seis ignorados.
10. **Publicação:** nada foi publicado no GitHub, nenhuma correção de produção foi aplicada e nenhum achado foi marcado como remediado.

Os itens parciais das tabelas não recebem automaticamente CWE: ausência de funcionalidade, cobertura incompleta e limitação de ambiente não demonstram por si só uma fraqueza explorável. As fragilidades confirmadas têm classificação, evidência e reprodução na seção 6.

## 10. Checklist do DoD da #85

As caixas abaixo indicam a conclusão das **atividades de avaliação no escopo local autorizado**, após a complementação. Não significam que todos os controles passaram, que todas as funcionalidades existem ou que o sistema é seguro. PASS/FAIL/PARTIAL dos casos continuam visíveis na seção 5. A recuperação e o webhook Avenia ausentes são limitações registradas, conforme entendimento do solicitante; não foram implementados nesta issue de testes.

- [x] **Autenticação, autorização, sessão e isolamento entre empresas avaliados.** Seções 5.1/5.2, TC-AUTH-15/16 e SEC85-06: comportamento de revogação/TTL complementado; endpoints de recuperação/logout ausentes identificados como limitação, sem alegar execução de funcionalidade inexistente.
- [x] **Endpoints e APIs do escopo local submetidos a casos documentados.** Seção 4 e endpoints.csv: 59 operações identificadas, barreiras testadas nas 55 protegidas e casos específicos nas quatro públicas. Isso não afirma cobertura de todos os payloads de cada operação.
- [x] **Fluxos de negócio e controles contra abuso avaliados com dados sintéticos.** Cadastro, perfis, repetição, isolamento e confirmação avaliados nas seções 5.2–5.4; rate limiting e funcionalidades ausentes explicitamente limitados.
- [x] **Integridade de valores, moedas, cotações, beneficiários e transferências avaliada.** Seção 5.4 e TC-FIN-13–19: matriz monetária executada, comparação da persistência e do ticket simulado, fronteiras e tarifas. Perda de precisão e tarifa negativa registradas como falhas em SEC85-07/08; liquidação real fora do escopo.
- [x] **Idempotência, concorrência, condições de corrida e transições avaliadas nas condições locais definidas.** TC-CONC-01/02 e TC-FIN-08/09; limites de abrangência nas seções 6 e 9.
- [x] **Integrações e webhooks avaliados dentro das condições autorizadas.** Seção 5.5 e TC-INT-10–13: respostas transitórias, falhas de transporte, deadlines em loopback, repetição e reconciliação simuladas. Receptor Avenia ausente, chatbot fora do HEAD e comportamento AWS/terceiros identificados como limitações, sem testes ativos externos.
- [x] **Resultados, evidências e limitações registrados para cada caso executado.** Seções 3–5 e arquivos JSON sanitizados; casos individuais da regressão e do diagnóstico enumerados.
- [x] **Vulnerabilidades reproduzíveis registradas com passos seguros.** Seções 6 e 8; sem movimentação real nem exposição de segredos.

A entrega de testes locais está complementada. O responsável pela issue deve revisar as evidências e as limitações antes do encerramento administrativo no GitHub, que não foi realizado nesta execução. Corrigir os achados e comprovar sua remediação pertence à etapa de correção/reteste; não é condição para registrar que o teste encontrou uma falha.
