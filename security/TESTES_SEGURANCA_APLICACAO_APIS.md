# Testes de segurança da aplicação e APIs — Issue #85

**Issue:** [AGES-V-Stable/cicd#85](https://github.com/AGES-V-Stable/cicd/issues/85)  
**Épico:** [cicd#80](https://github.com/AGES-V-Stable/cicd/issues/80)  
**Planejamento de referência:** [Relatório da #83](PLANEJAMENTO_E_RECONHECIMENTO_TECNICO.md)  
**Data:** 10/11/2026  
**Revisão da regra de acesso:** 12/11/2026 — OWNER (Representante), OPERATOR (Operador) e VIEWER (somente leitura) são perfis empresariais; Admin é uma conta separada da V-Stable; decisões de compliance exclusivas das APIs dos provedores.
**Condição:** capturas locais com entidades sintéticas e integrações simuladas; complementações reais BlindPay Development (seção 3.6) e Avenia sandbox (seções 3.8 e 5.7) em 13/11/2026. Recebimento externo de webhooks não executado.
**Situação:** avaliação local complementada em 12/11 e instrumentação revisada após o review do PR em 13/11 (seção 3.7). Alguns testes anteriores contradiziam o comportamento seguro ou duplicavam verificações; seus totais são históricos e não comprovam capacidade de verificar uma remediação. Permanecem limitações de cobertura explícitas na matriz e nas seções 9 e 10. Não representa aprovação de segurança nem certificação.

**Complementação externa em 13/11:** BlindPay Development na seção 3.6: 77 subcasos, 60 PASS, 14 FAIL relativos a dois achados, um bloqueio de cadastro externo e duas exclusões. Avenia sandbox nas seções 3.8 e 5.7: 26 IDs, 22 PASS, três FAIL e um grupo de webhooks NÃO EXECUTADO; 40 chamadas reais e dois tickets PAID. Permanecem as limitações da seção 9.1 e o recebimento de webhook real. O fluxo de convites está fora destas etapas, mesmo havendo código correspondente no HEAD mais recente.

## 1. Resumo executivo

Os testes locais reproduziram oito grupos de fragilidades: confirmação de oferta sem TOTP; perda do controle de reutilização e bloqueio TOTP entre instâncias/reinícios; possibilidade de decisão manual de compliance por Admin; validação e consumo de TOTP sem exclusão entre operações concorrentes; aceitação de arquivos truncados identificados apenas pelo prefixo PDF/PNG/JPEG; manutenção de sessão após alteração de senha na base isolada; perda de precisão de valores cripto no histórico; e aceitação de tarifa negativa no adapter BlindPay. A conclusão anterior sobre segregação entre SUPPORT, COMPLIANCE_ANALYST e SUPER_ADMIN foi retirada: esses três níveis não fazem parte da regra de negócio revisada.

Também foram observados controles efetivos nos cenários executados: rejeição de JWT inválido/expirado e de challengeToken em rotas protegidas, bloqueio de usuários empresariais nas rotas administrativas, isolamento de recursos de empresas diferentes, rejeição de oferta expirada ou valores adulterados, unicidade de uma execução sob cinco confirmações concorrentes e autenticação/deduplicação de webhooks BlindPay.

A complementação externa da BlindPay concluiu os oito cenários funcionais de sucesso, falha/reembolso de payin e payout, expiração, recotação e idempotência. Reproduziu SEC85-01 com conta de TOTP ativo e identificou SEC85-11, tarifa em token persistida como BRL sem conversão. Foram 80 chamadas na tentativa completa, 123 confirmadas na campanha e 139 posições no orçamento conservador com reserva pela interrupção, abaixo do teto 180. Esses números são independentes dos 385 casos da captura local de 12/11.

A complementação Avenia confirmou em sandbox SEC85-01 (confirmação sem TOTP financeiro) e SEC85-07 (1,123456 USDC na execução versus 1,12 no histórico). Também registrou o estado local SUBMITTED após PAID no provedor como falha operacional de acompanhamento, sem exploração demonstrada. A confirmação repetida devolveu a mesma execução sem nova chamada externa. Os resultados, recomendações, recursos criados e reprodução estão integrados às seções correspondentes deste relatório.

| Execução | Casos | PASS | FAIL | Ignorados |
|---|---:|---:|---:|---:|
| Regressão histórica do backend, em 10/11 | 1.244 | 1.238 | 0 | 6 |
| Regressão histórica do frontend, em 10/11 | 772 | 772 | 0 | 0 |
| Diagnóstico histórico da #85, em 10/11, com a matriz administrativa anterior | 217 | 198 | 19 | 0 |
| Diagnóstico intermediário de 12/11, antes do esclarecimento sobre VIEWER e OWNER | 221 | 204 | 17 | 0 |
| Reteste anterior de 12/11, com OWNER/OPERATOR/VIEWER e Admin separado | 228 | 212 | 16 | 0 |
| Complementação local de 12/11, antes da revisão da instrumentação | 385 | 364 | 21 | 0 |
| Regressão do backend no commit atualizado, em 12/11 | 1.241 | 1.235 | 0 | 6 |
| Revisão da suíte local do PR em 13/11, branch test/85-seguranca-aplicacao-apis | 371 | 344 | 27 | 0 |
| Regressão final da etapa Avenia em 13/11, APIs reais desativadas | 1.522 | 1.516 | 0 | 6 |

Os 19 FAIL da captura de 10/11 incluem cinco reprovações baseadas na matriz administrativa substituída. A captura de 228 casos foi sucedida pela complementação de 385 casos e pela revisão de 371 casos da seção 3.7. **Falhas de casos não equivalem a vulnerabilidades independentes.** Dos 21 FAIL de 12/11, 16 reproduzem os grupos SEC85-01–08, três correspondem a SEC85-09/10 e dois são desvios OBS85-01/02. Na revisão do PR há seis reprovações adicionais: dois cenários de TOTP ausente com 2FA ativo e quatro de erro monetário não controlado (OBS85-04). A diferença não representa seis vulnerabilidades novas nem alteração de produção. Os snapshots anteriores foram preservados.

Na complementação histórica de 10/11, foram acrescentados e executados **46 casos** de sessão, moedas, escalas, tarifas, persistência, recotação até payout simulado, estados terminais, falhas dos clientes e limites multipart em HTTP real. A primeira revisão de 12/11 produziu 221 casos, mas exigia incorretamente permissões idênticas para OWNER e OPERATOR e deixava VIEWER fora da matriz. Essa interpretação foi substituída após esclarecimento da equipe. As capturas antigas permanecem como histórico, não como resultado da matriz vigente. PARTIAL identifica funcionalidades ausentes, cobertura incompleta ou conclusões mais amplas que os casos demonstram; não equivale a requisito integralmente atendido.

A cobertura do frontend foi de 95,74% de linhas, 93,67% de statements, 93,90% de funções e 87,25% de branches. Cobertura de código e regressão funcional não equivalem a cobertura completa dos requisitos de segurança.

## 2. Versões e decisões de escopo

Na avaliação histórica de 10/11, os seguintes HEAD locais foram avaliados. O commit do reteste de 12/11 está na seção 3.4; resultados antigos não são automaticamente atribuídos à versão mais recente:

| Repositório | Branch | Commit completo |
|---|---|---|
| backend | main | `07c2302fea9434dfd107f803010a6d066efb548a` |
| frontend | main | `49946566b6023d36ebf738b7971ecd878e530f71` |
| cicd | main | `940f9d142e733cfdfe69f5cbd14be0a1bcd51695` |
| wiki | main | `a853aedfc9d64b8dab452e7e1b969fbbca56a10a` |

Cada captura identifica os commits e os hashes SHA-256 dos arquivos da avaliação, incluindo alterações locais adicionais ao HEAD, para identificar exatamente a instrumentação utilizada. Publicar ou integrar os artefatos não equivale a executar novamente os testes.

O backend contém mudanças posteriores ao reconhecimento da #83, incluindo vínculos empresariais e as migrations V9–V11. Não se presumiu que os achados de 08/11 permanecessem válidos.

O prompt final fornecido pelo solicitante definiu os HEAD locais e provedores simulados como escopo. As branches `iachat-backend` e `ia-chatbot` não foram integradas nem avaliadas dinamicamente. O endpoint `/v1/support/chat` não existe no HEAD de backend avaliado.

### 2.1. Matriz de acesso revisada em 12/11/2026

A regra abaixo incorpora o esclarecimento da equipe: existem três perfis de **usuário empresarial**, vinculados à empresa, e uma **conta administrativa da plataforma**, separada desses usuários. "Operador" havia sido usado genericamente para usuários associados à empresa; isso não elimina VIEWER nem torna todos os perfis equivalentes. "Visualizar tudo" não concede acesso a outras empresas.

| Ação | OWNER — Representante | OPERATOR — Operador | VIEWER — Visualizador | ADMIN — V-Stable |
|---|---|---|---|---|
| Consultar dados e recursos empresariais | Da própria empresa | Da própria empresa | Da própria empresa | Acesso administrativo global |
| Operar transferências e cadastrar beneficiários | Permitido na própria empresa | Permitido na própria empresa | Negado; somente leitura | Acesso administrativo conforme endpoints; cobertura operacional ainda parcial |
| Permissões empresariais de membros e representação | Capacidades de proprietário, incluindo MEMBERS_READ/MEMBERS_MANAGE anunciadas pelo app | Sem capacidades exclusivas de proprietário | Sem capacidades de escrita/gestão | Administração da plataforma; sem vínculo empresarial automático |
| Enviar termos, documentos, KYB e RFI | Fluxos de representação do OWNER | Sem acesso exclusivo do OWNER | Negado | Não pressupõe papel de representante empresarial |
| Editar/excluir empresas e gerenciar permissões gerais administrativamente | Sem privilégio administrativo global | Sem privilégio administrativo global | Sem privilégio administrativo global | Permitido, conforme regra administrativa informada |
| Aprovar/reprovar compliance manualmente | Negado | Negado | Negado | Negado |

Mapeamento confirmado pela equipe: `OWNER` é Representante, `OPERATOR` é quem pode operar e `VIEWER` é quem apenas visualiza. Os três autenticam como `USER`, com permissões calculadas pelo vínculo empresarial. Admin autentica como `ADMIN`/`ROLE_ADMIN`, sem `companyId` ou `companyAccess` empresarial. O código ainda contém o enum legado `AdminAccessLevel`; `SUPER_ADMIN` é apenas a fixture exigida pelo schema atual, não um novo perfil de negócio.

**Interpretação corrigida:** as capacidades adicionais de OWNER não constituem falha apenas por diferirem de OPERATOR. TC-RBAC-07 passou a verificar as permissões próprias de cada perfil, retirando a exigência de igualdade. A presença de MEMBERS_MANAGE na resposta não comprova que todas as operações de gestão estejam implementadas ou tenham sido testadas. Enviar documentos ou solicitar avaliação não equivale a aprovar/reprovar compliance: a decisão continua exclusiva das APIs dos provedores e de seus fluxos autenticados/validados. A revisão altera somente testes e documentação; nenhuma permissão de produção foi modificada.

## 3. Metodologia, ambiente e evidências

### 3.1. Referências

- [OWASP WSTG 4.2](https://owasp.org/www-project-web-security-testing-guide/v42/): organização dos testes de autenticação, autorização, entradas, sessão e lógica de negócio.
- [OWASP ASVS 5.0.0](https://asvs.owasp.org/): referência de requisitos para os controles examinados. Não foi realizada verificação integral de todos os requisitos de um nível ASVS, e não se declara conformidade.
- [OWASP API Security Top 10 — 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/): API1/API3/API5 para autorização, API2 para autenticação, API4/API6 para recursos e abuso, API7 para SSRF, API8/API9 para configuração/inventário e API10 para consumo de provedores.
- [CWE](https://cwe.mitre.org/): classificação das causas demonstradas na seção 6.
- [NIST SP 800-115](https://csrc.nist.gov/pubs/sp/800/115/final): planejamento, execução controlada, registro de evidências e limitações.
- [RFC 6238, seção 5.2](https://www.rfc-editor.org/rfc/rfc6238): referência do uso único de TOTP após uma validação bem-sucedida.
- PCI DSS: não avaliado. Não foi estabelecido um escopo de dados de cartão; nenhuma conclusão de aplicabilidade ou conformidade PCI é emitida.

### 3.2. Condições das capturas locais de 10–12/11

- Windows, Java Temurin 25.0.3, Gradle Wrapper 9.7.1, Node 24.16.0, JUnit, MockMvc, Vitest e PostgreSQL 16.4 em Testcontainers.
- O PostgreSQL diagnóstico é novo e efêmero. Sua URL é definida pelo container dentro da classe de teste. A limpeza SQL afeta exclusivamente esse banco, sem usar bancos existentes do desenvolvedor.
- Contas com domínio `@example.test`, empresas e beneficiários sintéticos; documentos reduzidos a bytes inertes. Os CNPJs de teste têm dígitos verificadores válidos e são usados somente na base isolada.
- JWT e segredos TOTP/webhook de teste; nas capturas locais de 10–12/11 nenhuma credencial real foi carregada para operações de negócio. A etapa externa da seção 3.6 usa credenciais da BlindPay Development sem exportar seus valores.
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
| [assessment.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/assessment.json) | Snapshot histórico de 10/11 (217 casos); não é a captura atual. |
| [complementacao-local-2026-11-12](https://github.com/AGES-V-Stable/backend/tree/main/docs/security/evidencias-85/complementacao-local-2026-11-12) | Captura atual: diagnóstico, regressões, inventário, matrizes de cobertura e verificação; seção 3.5. |
| [assessment-inicial.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/assessment-inicial.json) | Snapshot da primeira entrega, de 171 casos; histórico preservado, não utilizado como resultado final. |
| [endpoints.csv](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/endpoints.csv) | Inventário obtido do mapeamento MVC real, com método, caminho, controller, handler e classificação de acesso. |

O exportador `backend/scripts/export-security-evidence.ps1` retém nomes, resultados, durações e tipos de falha. Não copia corpos HTTP, headers, tokens, mensagens de exceção, stack traces ou stdout/stderr. Os relatórios JUnit/HTML brutos permanecem em diretórios de build e não devem ser publicados sem revisão.

Em 11/11/2026, os seis arquivos de evidência foram apenas movidos para o backend, preservando seu conteúdo e as datas de captura de 10/11/2026. Essa reorganização não é uma nova execução dos testes. O exportador passou a gravar no novo diretório; seu hash nos snapshots históricos corresponde à versão usada na captura original.

Nomes de parametrizações que serializam DTOs são reduzidos a um índice com parâmetros omitidos. Cada registro também recebe evidence_id; a condição é descrita nas tabelas e no código do teste, evitando publicar payloads no nome da evidência.

O comando `gradlew.bat build --no-daemon --console=plain` também terminou com **BUILD SUCCESSFUL**, após a inclusão dos testes e da tarefa diagnóstica. A suíte funcional manteve os mesmos 1.244 casos, seis deles desabilitados na origem. O diagnóstico permanece separado e reprova os controles documentados.

A verificação conjunta histórica de 10/11 executou `gradlew.bat securityAssessmentTest build --continue --no-daemon --console=plain`: a regressão manteve zero falhas e as tarefas check/build foram concluídas. O processo conjunto retornou 1 pelas 19 asserções reprovadas daquela captura. Esse resultado não é erro de compilação, de inicialização dos testes ou uma remediação aprovada. O resumo histórico está em [verificacao-final.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/verificacao-final.json).

### 3.4. Reteste da matriz revisada — 12/11/2026

Backend avaliado: commit `61a9662c8f9d8051540bdf551b399d36c8bec719`, com as alterações locais da classe `ApplicationSecurityAssessmentTest` identificadas pelo SHA-256 na evidência. Ambiente local, PostgreSQL efêmero e integrações simuladas; nenhum sandbox foi acessado. A pasta `revisao-perfis-2026-11-12` preserva a captura intermediária de 221 casos, cuja exigência de igualdade entre OWNER e OPERATOR foi substituída. A captura vigente fica em `perfis-confirmados-2026-11-12`.

**Correção de interpretação:** a reprovação intermediária de TC-RBAC-07 por diferença de permissões entre OWNER e OPERATOR foi retirada. A equipe esclareceu que são perfis distintos; não se trata de remediação do produto. Os cinco FAIL históricos que exigiam segregação entre níveis administrativos também permanecem retirados. A restrição de decisões manuais de compliance por Admin continua sendo requisito.

**Resultado vigente:** 228 casos, 212 PASS, 16 FAIL e zero ignorados, conforme [assessment.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/perfis-confirmados-2026-11-12/assessment.json). A tarefa diagnóstica retornou código 1 por asserções; não houve falha de compilação/inicialização na captura final. Das 16 reprovações, 14 correspondem aos demais achados já documentados e duas à decisão manual de compliance por Admin. A matriz TC-RBAC-01–09 contém 20 casos: 18 PASS e dois FAIL. A redução de 17 para 16 reprovações vem da correção da interpretação de TC-RBAC-07, não de alteração de produção.

| Caso | Quantidade | Resultado observado | Status |
|---|---:|---|---|
| TC-RBAC-01 | 2 | Admin enviou APPROVED/REJECTED: HTTP 200 e KYB/AML alterados de PENDING para a decisão enviada; esperado 403 e estado inalterado | FAIL |
| TC-RBAC-02 | 1 | Admin excluiu a empresa sintética sem dependências; HTTP 204 e ausência confirmada no banco | PASS |
| TC-RBAC-03 | 1 | Admin editou cadastro; HTTP 200 e novo nome confirmado no banco | PASS |
| TC-RBAC-04 | 3 | OWNER, OPERATOR e VIEWER consultaram empresa/beneficiários próprios (200); outra empresa bloqueada (403) | PASS |
| TC-RBAC-05 | 2 | Ambos confirmaram oferta própria (200), com uma execução persistida; envio ao provedor simulado | PASS |
| TC-RBAC-06 | 6 | OWNER/OPERATOR/VIEWER não conseguiram aprovar nem reprovar compliance: 403 e estado inalterado | PASS |
| TC-RBAC-07 | 3 | Permissões anunciadas refletem o perfil: OWNER com representação/gestão empresarial; OPERATOR com operação; VIEWER somente leitura | PASS |
| TC-RBAC-08 | 1 | VIEWER recebeu 403 ao confirmar oferta e cadastrar beneficiário, sem novas operações persistidas | PASS |
| TC-RBAC-09 | 1 | Admin retornou ADMIN/ROLE_ADMIN, sem companyId/companyAccess de usuário empresarial | PASS |

TC-RBAC-07 verifica o contrato de permissões próprio de cada perfil, sem exigir igualdade. Não demonstra execução de todos os endpoints de gestão de usuários/permissões. A revisão não altera o comportamento de produção nem precisa alterar a definição atual de `CompanyRole` para excluir VIEWER ou igualar OWNER a OPERATOR.

**Regressão vigente:** o comando conjunto `gradlew.bat securityAssessmentTest build --continue --no-daemon --console=plain` concluiu as tarefas `test`, `check` e `build`, com 1.241 casos de regressão, 1.235 PASS, zero FAIL e seis ignorados. O processo retornou código 1 exclusivamente pelas 16 asserções da tarefa diagnóstica; portanto, não se apresenta o comando conjunto como BUILD SUCCESSFUL. A quantidade de regressão pertence ao commit atualizado e difere da captura histórica; o frontend não foi reexecutado. Evidências: [integration-retest.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/perfis-confirmados-2026-11-12/integration-retest.json) e [verificacao.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/perfis-confirmados-2026-11-12/verificacao.json). Nenhum código de produção foi alterado e nenhum achado foi corrigido nesta revisão.

### 3.5. Complementação da cobertura local — 12/11/2026 (captura histórica)

**Nota de revisão:** a seção 3.7 corrige limitações da própria suíte e da matriz desta captura. Os números abaixo registram a execução de 12/11; não devem ser usados para afirmar que todos os casos passariam após uma remediação nem que toda a API foi avaliada.

Backend: commit `3983b755e412e6552a1509992fc7d7485497b3bc`, acrescido dos testes identificados por SHA-256 na nova captura. O plano de saída desta etapa é finito: executar LC-API/AUTH/AUTHZ/FIN/INT/CONC/CONFIG/OUT/ABUSE abaixo, reexecutar as regressões e vincular os requisitos da #85 aos casos efetivamente executados. Não se exige testar combinações infinitas nem corrigir os controles reprovados para concluir sua avaliação.

**Execução diagnóstica:** 385 casos, 364 PASS, 21 FAIL, zero ignorados. São **157 novos casos**, dos quais 152 PASS e cinco FAIL. Todos os casos previstos abaixo foram executados; as cinco reprovações novas são detalhadas em SEC85-09/10 e OBS85-01/02. Uma reprovação de controle é diferente de um teste pendente ou de erro da infraestrutura.

**Regressões reexecutadas:** backend 1.241 casos (1.235 PASS, zero FAIL, seis desabilitados na origem); frontend 772 PASS, zero FAIL, código de saída 0. O comando conjunto `gradlew.bat securityAssessmentTest build --continue --no-daemon --console=plain` concluiu `test/check/build/jacocoTestReport` e retornou 1 exclusivamente pela tarefa diagnóstica com 21 asserções reprovadas; não se apresenta esse comando como BUILD SUCCESSFUL. A cobertura do frontend permaneceu em 95,74% linhas, 93,67% statements, 93,90% funções e 87,25% branches. O código de produção não foi modificado.

Artefatos atuais no backend: [assessment.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/complementacao-local-2026-11-12/assessment.json), [endpoints.csv](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/complementacao-local-2026-11-12/endpoints.csv), [cobertura-endpoints.csv](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/complementacao-local-2026-11-12/cobertura-endpoints.csv), [cobertura-requisitos.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/complementacao-local-2026-11-12/cobertura-requisitos.json) e [verificacao.json](https://github.com/AGES-V-Stable/backend/blob/main/docs/security/evidencias-85/complementacao-local-2026-11-12/verificacao.json). Os diretórios anteriores permanecem como histórico.

| ID do caso | Domínio/requisito | Endpoint/componente | Condição e dados sintéticos | Resultado esperado | Resultado obtido | Tipo de evidência/ambiente | Status |
|---|---|---|---|---|---|---|---|
| LC-API-01 (59) | Entradas/erros/exposição | 59 operações do inventário MVC | JSON truncado, UUID inválido ou leitura sem corpo | Sem crash/dados internos; entrada inválida em 4xx | Upload com JSON retorna 500; demais 58 casos satisfazem o contrato, OBS85-01 | Teste reproduzido, MockMvc/JWT/PostgreSQL; termos simulados | 58 PASS / 1 FAIL |
| LC-AUTHZ-01 (54) | BFLA | 18 operações administrativas | OWNER/OPERATOR/VIEWER × operação | Negar função global | 54 respostas 403 | Teste reproduzido, MockMvc/JWT/PostgreSQL | PASS |
| LC-AUTHZ-02 (6) | Propriedade pessoal | Seis operações KYC/documento/liveness | Id de outro representante e payload válido | 403 sem provedor/mutação | Seis 403, sem provedor e sem alteração de documento | Teste reproduzido, MockMvc/JWT/PostgreSQL | PASS |
| LC-AUTHZ-03 | Mass assignment | Cadastro público | companyId alheio, ADMIN, accessLevel e compliance aprovado | Não conceder privilégio/empresa alheia | Empresa própria PENDING; nenhum administrador criado | Teste reproduzido, HTTP/serviço/SQL reais | PASS |
| LC-AUTHZ-04 (2) | BOLA | Duas rotas de cotação | Beneficiário de outra empresa, payload válido | Negar antes do provedor | 404 nas duas, sem Avenia/BlindPay | Teste reproduzido, MockMvc/JWT/PostgreSQL | PASS |
| LC-FIN-01 (5) | Associação/validade | Confirmação de oferta | Beneficiário alheio/movido; oferta não selecionada, expirada ou margem 10s | Negar antes da execução | 404/422 sem execução/histórico/submissão | Teste reproduzido, MockMvc/JWT/PostgreSQL | PASS |
| LC-FIN-02 | Propriedades financeiras | Confirmação de oferta | Injeta empresa/beneficiário/usuário/moedas/taxa/provedor/COMPLETED | Preservar dados autorizados | Valores persistidos vêm da oferta/principal; campos extras ignorados | Teste reproduzido, SQL real; processor simulado | PASS |
| LC-FIN-03 | BOLA | Processor de recotação real | Empresa B aceita recotação da A | Negar sem payout | NotFound; REQUOTE_REQUIRED inalterado | Teste reproduzido, processor/transações/SQL reais | PASS |
| LC-API-02 (3) | Injeção | Filtro de beneficiários | Tautologia SQL, lookup e script inertes | Não ampliar escopo nem executar conteúdo | Sem registro alheio, dados internos ou perda de tabela | Teste reproduzido, MockMvc/JPA/PostgreSQL | PASS |
| LC-API-03 | Recursos | Lista de beneficiários | size=2147483647; tabela pequena | Limitar página sem carga | pageSize limitado a 2000 | Teste reproduzido, MockMvc/Spring Data | PASS |
| LC-ABUSE-01 | Repetição de login | Login | Dez senhas erradas sequenciais | Negar acesso sem token/erro interno/mutação | Dez 401; limitação de taxa avaliada separadamente em OBS85-03 | Teste reproduzido local + inspeção; não mede gateway | PASS |
| LC-INT-01 (2) | Validade/replay | Webhook | HMAC válido; timestamp ±601s | 401 sem evento/processor | Duas rejeições e zero persistência | Teste reproduzido, verificador/SQL reais | PASS |
| LC-INT-02 | Duplicidade concorrente | Webhook | Cinco entregas simultâneas do mesmo svix-id | Processar uma vez | Cinco 204; uma linha/chamada ao processor | Teste reproduzido, HMAC/transações/PostgreSQL | PASS |
| LC-INT-03 | Rollback/retry | Webhook | Processor falha; evento válido repetido | Permitir retry após rollback | Primeira tentativa revertida; segunda processa | Teste reproduzido, transações reais; falha simulada | PASS |
| LC-CONC-01 | Idempotência/estado | Recotação/processor/conciliação | Uma confirmação e quatro conciliações | Chave estável e um histórico | Uma chave distinta, um histórico, COMPLETED | Teste reproduzido, locks/SQL reais; payout simulado | PASS |
| LC-CONC-02 | TOCTOU | Processor de payout | Cotação válida; relógio avança 600s durante provisionamento | Não submeter vencida | Chamada de payout após expirar, SEC85-09 | Teste reproduzido, relógio/SQL reais; gateway simulado | FAIL |
| LC-AUTH-01 | Ciclo de autenticação | Setup/ativação/login/verify-2fa | Segredo novo, challenge, código válido/repetido e reconfiguração sem código | Fluxo válido; negar replay/reconfiguração | Login funciona, replay/reconfiguração negados, segredo cifrado no SQL | Teste reproduzido, HTTP/JWT/TOTP/conversor/SQL | PASS |
| LC-CONFIG-01 | Configuração/recursos | Spring Security/executor | CORS não confiável, headers, query/cookie sem Bearer | Negar origem/acesso indevidos e limitar executor | nosniff/DENY/no-store, sem JSESSIONID; pool 8/fila 50 | Teste reproduzido, configuração real | PASS |
| LC-OUT-01 (4) | Respostas externas | Clientes Avenia/BlindPay | JSON ilegível/array | Falha de integração controlada | Quatro rejeições controladas | Teste reproduzido, clientes reais/MockRestServiceServer | PASS |
| LC-OUT-02 (2) | Redirect/origem | Clientes com builder padrão | 302 para outro servidor local | Não seguir outra origem | Avenia segue; BlindPay não segue, SEC85-10 | Teste reproduzido, dois servidores loopback | 1 PASS / 1 FAIL |
| LC-OUT-03 (3) | SSRF/path/query | Cliente BlindPay | IDs com URL, travessia ou query | Preservar autoridade/rota | Prefixo preservado e query não injetada | Teste reproduzido, cliente real/MockRestServiceServer | PASS |
| LC-OUT-04 (4) | Resposta financeira | Adapter BlindPay | Resposta nula; origem nula/zero; destino negativo | Rejeição controlada | Três rejeições; resposta nula causa NPE, OBS85-02 | Teste reproduzido, adapter real/gateway simulado | 3 PASS / 1 FAIL |
| LC-OUT-05 (2) | Configuração HTTP real | Clientes com builder Spring | 302 entre origens locais | Não seguir outra origem | Confirma Avenia seguindo; BlindPay não segue | Teste reproduzido, contexto real/loopback | 1 PASS / 1 FAIL |

As referências de negócio por operação também incluem os casos **executados** dos controllers e serviços existentes, por exemplo QuoteOrchestratorServiceTest e BlindPayExecutionProcessorTest. Isso aproveita os cenários positivos/negativos de seleção de preço, falha parcial, quantidade fixa, prazo, saldo, financiamento, tarifas, recotação, falha/reembolso e reconciliação. A matriz distingue esses casos do componente dos testes diretos por rota; não afirma que cada caso de serviço passou pela rede HTTP nem que todos os payloads possíveis de cada endpoint foram explorados.

`export-security-coverage.ps1` rejeita operação sem caso LC-API-01 ou componente sem referência executada. Os 15 requisitos de escopo recebem IDs R01–R15 e evidências individuais existentes; resultados FAIL permanecem visíveis. Perfis `seed`/`migration` expõem `/events` em processos separados e não estão ativos na aplicação financeira avaliada; InternalJobSecurityTest/InternalJobControllerTest exercitam seus controles separadamente na regressão. Swagger/documentação automática são configuração auxiliar, não novas operações financeiras dentro das 59 rotas dos controllers.

### 3.6. Complementação BlindPay Development — 13/11/2026

Backend: `01af9fcd48431ecb2d059c710c80d23d69318f92`, branch `test/85-integracoes-sandbox`, acrescido da instrumentação identificada por hashes em `execution.json`. Aplicação em rede loopback, autenticação JWT/TOTP real e PostgreSQL 16.4 efêmero; `BlindPayClient` real apenas na instância confirmada como Development. Avenia substituída por mock. Não houve acesso a AWS, produção ou pagamento real.

Procedimento, achados, reprodução e índice das evidências: [relatório complementar no backend](https://github.com/AGES-V-Stable/backend/blob/test/85-integracoes-sandbox/docs/security/TESTES_SANDBOX_BLINDPAY.md). Capturas: [evidencias-85-sandbox/2026-11-13](https://github.com/AGES-V-Stable/backend/tree/test/85-integracoes-sandbox/docs/security/evidencias-85-sandbox/2026-11-13). Os links referem-se a arquivos dessa branch e ficam disponíveis após sua publicação pela equipe; esta avaliação não faz push.

A primeira preparação parou por uma exigência de AML `clear` que os dados de demonstração não comprovavam. No detalhe do cliente, KYC estava aprovado e os termos aceitos; AML veio nulo, sem alegação de triagem aprovada. Não houve cadastro externo novo: o contrato atual exige termos no navegador e documentos apropriados. Reutilizou-se cliente `individual` de demonstração e criaram-se conta bancária e carteira sintéticas próprias; isso não comprova KYB de empresa nem onboarding completo.

O desligamento do computador interrompeu a segunda tentativa. Seus checkpoints confirmam sucesso, falha e reembolso de payin; não comprovam término do cenário de falha de payout. O banco efêmero foi recuperado, e uma consulta de leitura confirmou que o payin interrompido completou no provedor, enquanto a base local permanecia AWAITING_FUNDS, sem payout registrado. Essa operação não foi reenviada. `attempt-03` é uma reexecução com fixtures separadas; somente `execution_state=FINISHED` comprova o término, pois arquivos RUNNING são checkpoints.

**Correção de interpretação de tarifa:** os quatro FAIL `*-fee-currency` em `attempt-02` foram retirados. A OpenAPI documenta tarifas de payin e payout em centavos do token; não se pode inferir a unidade de `flat_fee` pela moeda BRL de `sender_amount`. O defeito demonstrado é outro: o escalar de tarifa em USDB é copiado para `service_fee_brl` sem conversão, SEC85-11. A retirada da hipótese não é remediação da aplicação.

O orçamento finito é de 180 chamadas, intervalo mínimo de três segundos e até 25 minutos por execução externa, sem concorrência, fuzzing ou saturação. Antes da reexecução há 43 chamadas confirmadas e reserva conservadora de 16 para possíveis chamadas não salvas antes do desligamento. As 59 posições anteriores são descontadas do teto; a reserva não é apresentada como tráfego comprovado. Há parada em rate limit, erro de servidor, falha de autenticação ou incerteza de transporte; reconciliação automática desativada.

**Resultado final da etapa externa:** `attempt-03/assessment.json` e `execution.json` registram FINISHED e saída 1 diagnóstica: **77 subcasos, 60 PASS, 14 FAIL, um BLOQUEADO e dois NÃO EXECUTADO**. São sete ocorrências de SEC85-01 e sete de SEC85-11. Todos os oito cenários de estado/idempotência previstos foram executados; o bloqueio é de cadastro externo novo, e as exclusões são Avenia e webhook real. Foram 80 chamadas nesta tentativa, sem HTTP externo diferente de 200. Os hashes dos quatro arquivos da instrumentação coincidiram com a captura. `summary.json` contém a vinculação TC-SBX-BP-01–04 aos subcasos reais.

| Cenário | Resultado | Status do controle de estado |
|---|---|---|
| Sucesso | COMPLETED | PASS |
| Falha de payin | FAILED sem payout | PASS |
| Reembolso de payin | REFUNDED sem payout | PASS |
| Falha de payout | FAILED após financiamento | PASS |
| Reembolso de payout | REFUNDED | PASS |
| Oferta expirada | 422, sem chamada externa de confirmação | PASS |
| Cotação de payout expirada durante financiamento | Uma recotação real; COMPLETED | PASS |
| Idempotência/repetição | Mesma execução e mesmo payin; COMPLETED | PASS |

A nova cotação no cenário de expiração permitiu seguir automaticamente: `requote_attempts=1`, `requote_accepted=false`. **Aceitação explícita de oferta pior pelo usuário não foi executada no sandbox**, sem eliminar a evidência local correspondente. A leitura adicional `database-consistency.json` comparou sete vínculos empresariais/beneficiários e valores de origem/destino entre cotação, execução e histórico, todos consistentes; isso não torna a unidade incorreta da tarifa um PASS. A consulta ocorreu durante o último cenário e não substitui sua observação terminal.

**Validação local no HEAD de 13/11:** três testes do controle de saída aprovados. Regressão padrão com 1.522 casos, 1.516 PASS, zero FAIL e seis desabilitados; tarefas `test/check/build/jacocoTestReport` concluídas, sem suíte real de sandbox no gate. O comando conjunto `build securityAssessmentTest --continue` retornou 1 pelo diagnóstico, não por regressão funcional. Seu primeiro diagnóstico teve 421 casos/23 FAIL: dois exigiam indevidamente 401 na nova rota pública `/v1/onboarding/account`. A lista PUBLIC_ROUTES do teste foi alinhada ao contrato já explícito em SecurityConfiguration; não houve mudança de produção. O reteste local terminou com **419 casos, 398 PASS, 21 FAIL, zero ignorados**. Esses 21 resultados conhecidos permanecem, sem alegação de remediação. Inventário atualizado: 71 operações, cinco públicas, 66 protegidas e 18 administrativas. Capturas em `local-regression/`, incluindo a versão anterior à correção de classificação. Nenhum teste frontend foi reexecutado nesta etapa.

### 3.7. Revisão da instrumentação após os comentários do PR — 13/11/2026

**Versão desta revisão:** backend `7b46aa6`, branch `test/85-seguranca-aplicacao-apis`, acrescido das alterações de testes/scripts identificadas por SHA-256 na nova evidência. Esta branch é anterior ao HEAD da etapa externa da seção 3.6: os resultados não são intercambiáveis. A revisão executa somente mocks/loopback, MockMvc e PostgreSQL efêmero; não repete o sandbox.

**Resultado diagnóstico:** 371 casos, 344 PASS, 27 FAIL, nenhum ignorado. A composição é `385 - 18 + 2 + 2`: 18 repetições de OWNER/rotas administrativas removidas, dois cenários negativos adicionais com 2FA ativo e duas contraprovas do mecanismo de concorrência. OWNER continua coberto junto de OPERATOR/VIEWER em LC-AUTHZ-01. Os dois casos LC-HARNESS-01, que simulam recusa atômica na validação ou no consumo, passaram; não comprovam uma correção do serviço real.

Correções da instrumentação:

- **TC-FIN-01/20:** apenas asserções do comportamento seguro: recusar a ausência do fator, não iniciar o processamento/payout e preservar o estado. Não se exige mais payout/COMPLETED antes de esperar a recusa. Há casos com 2FA desligado (422) e ativo sem código (401), conforme a semântica do serviço TOTP já existente.
- **Demais casos financeiros:** helper envia `totpCode` válido da conta sintética com 2FA ativo, usando códigos distintos por requisição. O DTO `ConfirmOfferRequest` deste HEAD ainda ignora esse campo: a fixture prepara os casos para a remediação; não prova validação já implementada. Se o contrato final de remediação mudar, o helper deve acompanhá-lo. A concorrência por oferta usa cinco usuários OPERATOR da mesma empresa com fatores independentes, separando idempotência de replay TOTP.
- **TC-CONC-01:** recusa por `BadCredentialsException` conta como operação não autorizada. Todas as threads chegam à barreira entre as fases, inclusive as recusadas; exceções de infraestrutura continuam erros. A implementação real ainda permite cinco autorizações; o teste continua FAIL pela propriedade avaliada.
- **TC-AUTHZ-05:** aceita 403/404 ou 200 com histórico filtrado, sempre sem os identificadores financeiros da outra empresa. Não obriga a manter HTTP 200.
- **TC-FIN-17:** exige erro de validação controlado (`IllegalArgumentException`/400 ou `UnprocessableEntityException`/422), sem gateway. Os quatro casos antes PASS por `ArithmeticException` passam a FAIL, conforme OBS85-04; não houve correção do adapter.
- **LC-API-01:** reset dos mocks e verificação de interações dentro de cada DynamicTest. Só a rota de termos permite sua chamada específica ao mock; nenhuma outra interação com os provedores é admitida por esse caso. Não se depende de `@AfterEach` por teste dinâmico.
- **Inventário:** 59 operações dos controllers ativos no perfil padrão; duas definições condicionais `POST /events`, uma por perfil seed/migration, listadas separadamente. Há referências à regressão dos jobs, com limites específicos de contexto HTTP e implantação.
- **Rastreabilidade:** grupos obrigatórios explícitos; ausência/skip gera cobertura parcial ou não avaliada, FAIL de controle permanece visível. R15 referencia `validacao-evidencias.json` e a checagem efetiva de metadados/schema/hashes, sem regex que associe todos os testes. Referência de componente não comprova todas as entradas de cada rota.
- **Duplicações de preparo:** fixture compartilhada dos clientes com credenciais descartáveis e helper do processador de recotação; nenhum segredo real.

Artefatos desta revisão: `backend/docs/security/evidencias-85/revisao-pr-2026-11-13/`, incluindo `assessment.json`, `integration-retest.json`, inventários, matrizes, `validacao-evidencias.json`, `verificacao.json` e `REVISAO.md`. São uma captura nova; os JSON/CSV de 12/11 permanecem históricos. O build/regressão e a validação dos exportadores são registrados nessa captura. Frontend e sandbox não são reexecutados por esta revisão.

### 3.8. Complementação Avenia sandbox — 13/11/2026

Em **13/11/2026**, foram executadas **40 chamadas HTTP reais** à Avenia sandbox, dentro do teto de 50. Houve **dois tickets de teste pagos pelo simulador**: um na conta principal, com saída INTERNAL para a própria conta, e outro na subconta sintética, pela aplicação local, com saída USDC/POLYGON. Nenhum PIX foi pago em aplicativo bancário, nem houve envio de ativos reais.

A consolidação contém **26 IDs: 22 PASS, 3 FAIL e 1 NÃO EXECUTADO**. Isso corresponde a 25 subcasos avaliados e um grupo de webhooks não executado; não significa cobertura integral da issue. Estados financeiros de falha/cancelamento, recuperação de resultado incerto e outros limites abaixo não estão incluídos como testes aprovados.

Os três FAIL finais são:

- **TC-SBX-AV-05.2 / SEC85-01:** a confirmação HTTP da oferta criou um ticket real de sandbox sem TOTP da transferência, com usuário sintético cujo TOTP estava ativo e que havia feito login com senha e segundo fator. A rota direta, em contraste, recusou a ausência do fator com HTTP 422 e zero chamadas externas.
- **TC-SBX-AV-03.4 / SEC85-07:** a execução preservou **1,123456 USDC**, mas o histórico armazenou **1,12 USDC**. Ambos os valores são massa de teste, originados no fluxo com provedor real.
- **TC-SBX-AV-06.3 / apontamento anterior VULN-11:** o ticket chegou a `PAID`, enquanto o GET local permaneceu em `SUBMITTED`. É uma observação operacional de acompanhamento, não uma exploração de segurança demonstrada. Os agendamentos estavam desativados para controlar o orçamento; além disso, a inspeção do HEAD mostra que o reconciliador existente filtra `BLINDPAY`, e não existe receptor Avenia. Portanto, não se atribui o resultado apenas à configuração do teste, nem se afirma ter validado um reconciliador Avenia habilitado.

Nenhuma vulnerabilidade foi corrigida nesta entrega. Os achados locais anteriores não são apresentados como novos achados independentes.

#### Versão, ambiente e autorização

| Item | Condição avaliada |
|---|---|
| Repositório | AGES-V-Stable/backend |
| Branch | `test/85-seguranca-avenia-sandbox` |
| HEAD base | `1a3add10ab0032b9834ff552e9af74a14cd859d8` |
| Código executado | HEAD acima mais instrumentação de teste não commitada; hashes por tentativa em `runner.json` |
| Aplicação | Spring Boot local, servidor HTTP em loopback e porta aleatória |
| Banco | PostgreSQL `16.4` efêmero, criado e descartado por Testcontainers |
| Ferramentas | Java Temurin `25.0.3`, Gradle Wrapper `9.7.1`, Docker `29.8.0` |
| Único destino de negócio externo | `https://api.sandbox.avenia.io:10952` |
| Credenciais | Novas chaves em `.env.avenia-security/`, ignoradas pelo Git; conta esperada confirmada |
| Conta do provedor | INDIVIDUAL; não equivale a KYB institucional/produção |
| Outras integrações | BlindPay e assistente simulados; verificadas sem interações; nenhum acesso AWS |
| Execução | Sequencial, sem fuzzing, carga, concorrência ou falhas de transporte induzidas |

O prompt do responsável autorizou especificamente essas chamadas funcionais como exceção às instruções gerais de testes simulados do `AGENTS.md`. Aplicam-se o planejamento da #83 e seus critérios de parada. Não houve alteração de chaves, permissões, destinos de webhook, configuração compartilhada, código de produção ou migrations.

O transporte de teste injeta timeout, bloqueio de redirects e allowlist de origem/método/path. Isso protege a execução e **não comprova que o builder HTTP padrão da aplicação tenha esses mesmos controles**; o achado anterior sobre redirects não foi marcado como corrigido.

#### Evidências e tentativas preservadas

Diretório: [evidencias-85-avenia-sandbox/2026-11-13](https://github.com/AGES-V-Stable/backend/blob/test/85-seguranca-avenia-sandbox/docs/security/evidencias-85-avenia-sandbox/2026-11-13/).

- [`summary.json`](https://github.com/AGES-V-Stable/backend/blob/test/85-seguranca-avenia-sandbox/docs/security/evidencias-85-avenia-sandbox/2026-11-13/summary.json): última observação de cada ID, origem da captura e totais; não soma os retestes como novos casos independentes.
- `assessment.json` em cada tentativa: esperado, observado, camada, detalhes permitidos, status, chamadas e horários UTC.
- `runner.json`: branch, HEAD, hashes dos arquivos e encerramento efetivo do Gradle. As revisões da instrumentação têm hashes diferentes; não são uma única captura imutável.
- `validation.json`: resultado da regressão, sanitização e verificação final, quando concluídas.

| Tentativa | HTTP reais | Estado da avaliação | Saída Gradle | Interpretação |
|---|---:|---|---:|---|
| `av85-preflight-01-preflight` | 5 | FINISHED | 0 | Conta, metadados, limites e ausência de callbacks |
| `av85-assessment-01-assessment` | 13 | FINISHED | 1 | Contratos e primeiro ticket PAID; caso local com payload incompleto refeito depois |
| `av85-business-01-business` | 4 | INTERRUPTED | 1 | Preparação do beneficiário rejeitada com HTTP 400; nenhum ticket empresarial enviado |
| `av85-business-02-business` | 5 | FINISHED | 0 | Beneficiário GNOSIS criado; cotação externa HTTP 500, oferta local HTTP 422; cenários dependentes bloqueados |
| `av85-business-03-business` | 13 | FINISHED | 1 | Alternativa POLYGON funcional, segundo ticket PAID, três asserções finais FAIL |
| **Total desta avaliação** | **40** | **Processos encerrados** | — | **Sem `ignoreFailures`** |

As duas consultas GET anteriores de validação das novas chaves são históricas e **não estão somadas às 40 chamadas**. Incluindo-as, são 42 chamadas conhecidas. Não houve 429 ou resultado financeiro desconhecido. Um 500 foi observado, sem persistência na rota alternativa; a parada global estava configurada para o segundo 5xx.

O primeiro `TC-SBX-AV-02.1` retornou 400, não o 404 esperado. O corpo de teste foi ajustado para fornecer os campos conforme o DTO, inclusive valor numérico e `coverFees`; o reteste retornou 404. O FAIL original foi preservado como falha de instrumentação e não contado como vulnerabilidade.

O primeiro payload de carteira incluía `isSameEconomicGroup=false` para país BR. A documentação orienta omitir o campo nessa condição; o payload ajustado foi aceito. A mensagem sanitizada da primeira rejeição não foi capturada, portanto a causa exata daquele 400 não foi demonstrada. Não se classifica essa tentativa como defeito confirmado do produto.

Na primeira captura de `01.2`, apenas o sucesso HTTP ficou evidenciado; as flags não foram extraídas pelo formato inicialmente esperado. `01.6` complementou a leitura por allowlist e comprovou as permissões operacionais. Não se inferem permissões a partir do HTTP 200 isolado.

#### Fontes e metodologia Avenia

Consultadas em 13/11/2026: [início e ambientes](https://integration-guide.avenia.io/docs/Getting-Started/gettingStarted/), [primeira operação](https://integration-guide.avenia.io/docs/Getting-Started/firstOperation/), [operações suportadas](https://integration-guide.avenia.io/docs/Operations/combinations/), [metadados e limites](https://integration-guide.avenia.io/docs/Avenia-Account-Management/aboutLogin/), [subcontas](https://integration-guide.avenia.io/docs/Avenia%20Subaccounts/subAccountManagement/), [carteiras](https://integration-guide.avenia.io/docs/Beneficiaries-Wallets/walletsGuide/), [recebimento simulado](https://integration-guide.avenia.io/docs/Sandbox%20Usecases/receiveMockFunds/), [gestão de webhooks](https://integration-guide.avenia.io/docs/Webhooks/webhookManagement/) e [comportamento dos eventos](https://integration-guide.avenia.io/docs/Concepts/webhooksInPractice/).

O MCP documental Avenia não estava configurado nesta sessão; foram usados guias oficiais, sem instalar MCP de negócio ou enviar segredos. OWASP WSTG 4.2, ASVS 5.0.0, API Top 10:2023 e NIST SP 800-115 orientam os requisitos e a rastreabilidade. Não se declara conformidade integral. Nenhum dado de cartão foi processado e não foi demonstrada aplicabilidade PCI DSS nesta etapa.

## 4. Inventário de endpoints e cobertura

Na captura local de 12/11 foram identificadas **59 combinações de método e caminho** nos controllers ativos: quatro públicas e 55 protegidas por autenticação; 18 com autorização administrativa por `@PreAuthorize`. Esses números são históricos. O HEAD da complementação de 13/11 contém 71 operações, incluindo a nova rota pública de cadastro de conta; a revalidação local e a atualização de sua classificação são registradas na seção 3.6, sem atribuir a captura antiga ao código novo.

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

As condições locais da seção 3.2 aplicam-se às tabelas históricas TC-AUTH/AUTHZ/RBAC/API/FIN/INT. A complementação TC-SBX-BP tem ambiente próprio nas seções 3.6 e 5.6. IDs presentes no nome dos testes podem ser procurados em suas respectivas capturas. Para casos reaproveitados, o nome da suíte é a chave de busca nos arquivos de baseline.

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
| TC-RBAC-01 | Proibir decisão manual de compliance | `PATCH /v1/companies/{id}/compliance-status` | Admin; APPROVED e REJECTED, em KYB e AML | HTTP 403 e estado persistido inalterado | HTTP 200 e decisões persistidas nos dois casos | Reteste local, seção 3.4 | FAIL |
| TC-RBAC-02 | Excluir empresas | `DELETE /v1/companies/{id}` | Admin; empresa sintética sem dependências | HTTP 204 e empresa removida | HTTP 204 e ausência confirmada no banco | Reteste local, seção 3.4 | PASS |
| TC-RBAC-03 | Alterar cadastro | `PUT /v1/companies/{id}` | Admin; cadastro sintético válido | HTTP 200 e novo nome persistido | HTTP 200 e nome persistido | Reteste local, seção 3.4 | PASS |
| TC-RBAC-04 | Consulta e isolamento | Empresas e lista de beneficiários | OWNER, OPERATOR e VIEWER | HTTP 200 na própria empresa; 403 na outra | Três perfis respeitaram esses acessos e bloqueios | Reteste local, seção 3.4 | PASS |
| TC-RBAC-05 | Operação financeira por perfil | Confirmação de oferta própria | OWNER e OPERATOR | HTTP 200 e uma execução registrada | Ambos obtiveram 200 e uma execução; TOTP avaliado separadamente em TC-FIN-01 | Reteste local, seção 3.4 | PASS |
| TC-RBAC-06 | Proibir decisão manual de usuários | Atualização de compliance da própria empresa | OWNER/OPERATOR/VIEWER × APPROVED/REJECTED | HTTP 403 e estado inalterado | Seis bloqueios 403 sem alteração de KYB/AML | Reteste local, seção 3.4 | PASS |
| TC-RBAC-07 | Permissões próprias de cada perfil | `GET /v1/users/me` | OWNER, OPERATOR e VIEWER | Capacidades de cada papel e accountType USER, sem exigir igualdade | Três conjuntos de permissões conferidos | Reteste local, seção 3.4 | PASS |
| TC-RBAC-08 | Somente leitura | Confirmação de oferta e cadastro de beneficiário | VIEWER; payloads sintéticos | HTTP 403 sem nova transferência/beneficiário | Duas respostas 403; contagem de registros inalterada | Reteste local, seção 3.4 | PASS |
| TC-RBAC-09 | Separação de conta administrativa | `GET /v1/users/me` | Conta Admin sintética | ADMIN/ROLE_ADMIN sem vínculo empresarial | Tipo/autoridade conferidos e vínculo empresarial ausente | Reteste local, seção 3.4 | PASS |

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
| TC-FIN-17 | Casas excedentes e overflow | Mesmo adapter | 5.001, 5.009 e dois valores fora do intervalo de long em centavos | Validação controlada, sem arredondamento nem gateway | ArithmeticException interna não satisfaz o contrato seguro revisado | Teste de adapter, sem afirmar HTTP 500; OBS85-04 e seção 3.7 | FAIL (4) |
| TC-FIN-18 | Moeda/rede incompatível | Mesmo adapter | BTC, EURC, rede desconhecida, origem não correspondente ao token, destino divergente do beneficiário | Não selecionar provedor para rota incompatível | Cinco combinações negadas por supports, sem gateway | Teste reproduzido de adapter | PASS |
| TC-FIN-19 | Sinal de tarifa externa | Mesmo adapter | Tarifas de 0, 1, 123 e -1 centavos | Aceitar zero/positivas; rejeitar negativa | Três controles positivos aprovados; tarifa negativa aceita | Teste reproduzido de adapter | FAIL |
| TC-FIN-20 | Segundo fator na recotação | `/v1/transfer-executions/{id}/requote/confirm` | Estado REQUOTE_REQUIRED, cotação vigente, sem código; casos com 2FA desligado e ativo | 422/401, sem processor/payout e sem mudar o estado | HTTP 200 e processamento simulado nos dois casos | HTTP/serviço/PostgreSQL; somente asserções seguras, seção 3.7 | FAIL (2) |
| TC-CONC-01 | TOCTOU no segundo fator | `validateTransferTotp` / `consumeTransferTotp` | Cinco operações, barreiras comuns inclusive para recusas | Exatamente uma autorização, recusas de credencial não são erros | As cinco validam e consomem sem erro | Serviço real; duas contraprovas atômicas simuladas PASS; seção 3.7 | FAIL |
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

### 5.6. Casos de sandbox BlindPay

Os grupos TC-SBX-BP-01–04 são detalhados no relatório complementar e nos registros individuais da tentativa completa. Autenticação da integração, associação de recursos e isolamento por HTTP local, cotação SOURCE/TARGET com centavos/validade/tarifas e máquina de estados real são separados em subcasos PASS/FAIL/BLOQUEADO/NÃO EXECUTADO. Não se aprova um grupo inteiro apenas porque uma chamada passou. Falha/reembolso provocado no sandbox pode representar PASS do tratamento da aplicação.

A confirmação utiliza conta local com TOTP ativo e sessão emitida após challenge válido; a requisição de transferência omite TOTP. O isolamento é exercitado com empresa B acessando o beneficiário e as ofertas de A, medindo a ausência de chamadas externas na rejeição. Polling não equivale a entrega de webhook. Expiração durante financiamento é provocada exclusivamente no banco efêmero e assim identificada.

### 5.7. Casos de sandbox Avenia

As pré-condições comuns são a conta esperada, chaves novas, orçamento disponível e transporte restrito. Os casos HTTP locais usam empresas e representantes OWNER sintéticos A/B com login real e TOTP ativo. Não reavaliam integralmente OPERATOR/VIEWER/Admin ou convites. Quando os registros financeiros iniciais são fixtures locais, isso está declarado no JSON; não são apresentados como operações remotas.

| ID TC-SBX-AV | Requisito / esperado | Observado e camada | Resultado final |
|---|---|---|---|
| 01.1 | Assinatura RSA e conta correta | GET real pelo transporte/signer do cliente; conta confirmada | PASS |
| 01.2 | Consultar acesso sem expor dados | Consulta autenticada; interpretação das flags complementada em 01.6 | PASS limitado |
| 01.3 | Conhecer liberações | BRL/USD liberados; demais flags restritas no JSON; não prova KYB | PASS |
| 01.4 | Verificar bloqueio e limites | Conta não bloqueada; valores da conta omitidos | PASS |
| 01.5 | Evitar callbacks fora do escopo | Lista de destinos vazia, preservada | PASS |
| 01.6 | Confirmar permissões antes da escrita | Flags operacionais conhecidas e conta novamente confirmada | PASS |
| 02.1 | B não cota beneficiário de A | 404 e zero chamadas externas; fixture local | PASS após reteste |
| 02.2 | B não confirma oferta de A | 404 e zero chamadas externas; fixture local | PASS |
| 02.3 | B não consulta execução de A | 404 e zero chamadas externas; fixture local | PASS |
| 02.4 | Criar subconta própria | Cliente real retornou 201; BRL liberado sem submissão KYC | PASS |
| 02.5 | Preservar vínculos após negativas | SQL confirmou empresa proprietária; nenhum efeito remoto | PASS |
| 02.6 | Vincular carteira remota à empresa A | Carteira própria na subconta correta; vínculo local, KYC PENDING | PASS |
| 02.7 | Impedir uso da carteira remota por B | HTTP 404 antes da Avenia | PASS |
| 02.8 | Impedir confirmação por B da oferta real de A | HTTP 404, sem novo efeito remoto | PASS |
| 03.1 | Cotação com origem fixa | 10 BRL preservados; saída USDC, DTO tipado, tarifas exportadas por moeda | PASS |
| 03.2 | Cotação com destino fixo e precisão | 1,123456 USDC preservados; tarifas observadas sem somar moedas diferentes | PASS |
| 03.3 | Oferta real e persistência | HTTP local, agregador e provider_quotes preservaram 1,123456 USDC; origem confirmada 6,71 BRL | PASS após bloqueio GNOSIS |
| 03.4 | Histórico preservar precisão da execução | 1,12345600 USDC versus 1,12 USDC | **FAIL — SEC85-07** |
| 04.1 | Criar e acompanhar ticket de simulação | Conta principal, PIX → USDC INTERNAL; PROCESSING → PAID | PASS |
| 04.2 | Executar oferta na subconta e consultar status | PIX → USDC POLYGON, PAID; sem atualizar banco artificialmente | PASS |
| 05.1 | Rota direta exigir TOTP financeiro | HTTP 422 sem fator e sem chamada externa | PASS |
| 05.2 | Confirmação de oferta também exigir fator | HTTP 200 e ticket criado sem TOTP financeiro | **FAIL — SEC85-01** |
| 06.1 | Repetir confirmação sem novo efeito | Mesma execução, um registro e zero chamadas adicionais | PASS |
| 06.2 | Ticket terminal permanecer estável | Segunda consulta ao primeiro ticket manteve PAID | PASS limitado à leitura |
| 06.3 | Refletir liquidação no estado local | Provedor PAID versus API local SUBMITTED | **FAIL operacional / limitação de acompanhamento** |
| 08.1 | Receber e autenticar eventos reais | Receptor Avenia e destino público próprio ausentes; nenhum túnel | **NÃO EXECUTADO** |

Cada linha tem sua captura individual em `summary.json`. A numeração 08 preserva o grupo de webhooks solicitado; não existem grupos executados 07 ou novos testes inferidos dos números.

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

Na captura local, a confirmação inicial alcança o ponto de disparo ao processador simulado. A recotação também foi reproduzida em TC-FIN-20: a chamada do bean substituído delega a acceptRequote do processador real, com transações e PostgreSQL reais; apenas o provisionamento da carteira e a resposta de payout são simulados. Sem TOTP, houve uma chamada de payout e o estado chegou a COMPLETED, em vez de rejeitar a operação com 422. Essa captura não usou provedor real. A seção 3.6 complementa o achado com confirmação HTTP por conta com TOTP ativo e acionamento real da BlindPay Development, sem dinheiro real; não se atribui esse efeito à Avenia.

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

### SEC85-03 — Admin pode decidir compliance manualmente

**Prioridade:** alta. **Referência anterior:** VULN-06. **Classificação:** CWE-863; OWASP API5:2023.

**Regra revisada:** nenhum usuário humano, incluindo Admin, pode aprovar ou reprovar compliance. O resultado deve ser determinado pelos provedores e atualizado pelos fluxos de integração. Ter uma única autoridade `ROLE_ADMIN` deixa de ser, por si só, uma falha.

**Código:** `CompanyController.updateComplianceStatus` permite `PATCH /v1/companies/{id}/compliance-status` para `ROLE_ADMIN`; `CompanyService.updateComplianceStatus` persiste os estados enviados pelo cliente sem exigir uma decisão recebida do provedor. A associação de edição/exclusão a Admin é compatível com a nova regra e não integra mais este achado.

**Reprodução segura revisada:** executar TC-RBAC-01 na base efêmera com Admin sintético e JWT real. Para cada decisão APPROVED/REJECTED, enviar KYB/AML no PATCH e comparar o banco antes/depois. O esperado é HTTP 403 sem alteração. TC-RBAC-06 faz o mesmo com Operador e Representante. O resultado da execução revisada consta na seção 3.4.

**Recomendação:** impedir decisões manuais de compliance por credenciais humanas, inclusive Admin; manter a atualização somente nos fluxos validados dos provedores e registrar origem/evento da decisão. Preservar a possibilidade de consulta e o envio de documentação conforme a regra de acesso. Não reintroduzir os três níveis administrativos antigos. A exclusão de empresas com histórico financeiro exige política própria; não foi demonstrada remoção em cascata de toda a base financeira.

### SEC85-04 — Validação e consumo de TOTP permitem intercalamento

**Prioridade:** alta para investigação do fluxo financeiro. **Classificação:** CWE-367 e CWE-362; OWASP API2:2023/API6:2023.

**Código:** [TwoFactorAuthService.java:187](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L187), [consumo:208](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TwoFactorAuthService.java#L208) e intervalo entre [TransferService.java:131](https://github.com/AGES-V-Stable/backend/blob/07c2302fea9434dfd107f803010a6d066efb548a/src/main/java/ages/vstable/backend/service/TransferService.java#L131) e linha 147.

**Reprodução segura:** TC-CONC-01, `validationAndConsumptionMustBeAtomicAcrossFiveOperations`. Cinco threads iniciam juntas e chegam à barreira intermediária mesmo se recusadas na validação; somente as validadas tentam consumir. Recusas de credencial são `false`, sem mascarar timeouts ou outras exceções. A versão anterior não tratava corretamente essas recusas após uma correção atômica; a revisão e suas duas contraprovas estão na seção 3.7. Não se usa sleep e não são enviadas transferências.

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

### SEC85-09 — Cotação pode vencer entre a validação e o envio do payout

**Prioridade:** alta para integridade temporal. **Classificação:** [CWE-367](https://cwe.mitre.org/data/definitions/367.html). **Caso:** LC-CONC-02.

**Código no commit da seção 3.5:** `BlindPayExecutionProcessor.acceptRequote` valida a validade e muda o estado para PAYOUT_SUBMITTING. Em `submitPayout`, o processor carrega o contexto, provisiona a carteira e chama `createEvmPayout` sem nova checagem temporal depois do provisionamento.

**Reprodução local segura:** registrar execução sintética, colocá-la em REQUOTE_REQUIRED com cotação válida por cinco minutos, aceitar os valores 100/98 e usar um relógio controlado que avança dez minutos durante `ensureManagedWallet`. O gateway simulado registra uma chamada de payout após a expiração; a asserção esperava zero. O teste não espera dez minutos, não consulta carteiras reais e não envia dinheiro.

**Limite do impacto:** demonstra que o backend tenta submeter uma cotação vencida. O provedor real pode recusá-la; não se comprovou liquidação com preço inválido ou prejuízo. **Recomendação:** checar novamente validade/margem imediatamente antes da submissão, limitar duração do provisionamento e encaminhar cotação vencida ao fluxo de recotação/estado seguro, preservando idempotência.

### SEC85-10 — Cliente Avenia segue redirect para outra origem

**Prioridade:** média, condicionada a resposta/redirecionamento do provedor. **Referência:** OWASP API10:2023; risco relacionado a [CWE-918](https://cwe.mitre.org/data/definitions/918.html), sem comprovar SSRF diretamente controlável por usuário não autenticado. **Casos:** LC-OUT-02 e LC-OUT-05 para Avenia.

**Código no commit da seção 3.5:** `AveniaClient` usa o builder HTTP injetado e não restringe redirects à origem configurada. **Reprodução:** um servidor em 127.0.0.1 devolve 302/Location para outro servidor local em porta distinta. Uma requisição autenticada com credenciais exclusivamente sintéticas chega à segunda origem. LC-OUT-05 repete com o builder real da aplicação Spring, confirmando o comportamento na configuração local avaliada. Os casos equivalentes BlindPay não seguiram o redirect.

**Limites:** os dois destinos são loopback; nenhum serviço de metadata, provedor ou host privado real foi acessado. O teste mede a chegada da requisição, não exfiltração de chave nem comprometimento do provedor. Não se demonstrou controle desse redirect por um usuário comum. **Recomendação:** desabilitar redirects de APIs autenticadas ou validar explicitamente esquema, host e porta a cada salto, protegendo headers de credenciais e aplicando limites de saltos/tempo.

### OBS85-01/02 — Desvios de contrato e robustez, sem exploração de segurança demonstrada

| Observação | Caso e evidência | Resultado e limite | Encaminhamento |
|---|---|---|---|
| OBS85-01 | LC-API-01 POST `/v1/blindpay/compliance/documents`; assessment-0094 | JSON em rota multipart resulta em HTTP 500, corpo genérico; deveria ser erro de entrada 4xx. Nenhuma exposição de stack/credencial demonstrada | Tratar media type/multipart inválido explicitamente no handler de erros |
| OBS85-02 | LC-OUT-04 null-response; assessment-0368 | Resposta nula do gateway causa NullPointerException no adapter de cotação. A orquestração já captura RuntimeException; não foi demonstrado payout, vazamento ou queda global | Normalizar resposta ausente como exceção de integração controlada |

Esses dois FAIL são relevantes ao contrato/consumo seguro de API, mas não são apresentados como duas vulnerabilidades exploráveis nem recebem CWE automaticamente.

### OBS85-04 — Casas excedentes e overflow geram exceção aritmética sem validação controlada

**Casos:** quatro entradas TC-FIN-17 no adapter real com gateway/provisionamento simulados. `toCents` lança `ArithmeticException` para 5.001, 5.009 e valores que excedem `long` em centavos. O teste anterior aceitava essa exceção e passava; o contrato revisado exige validação controlada, sem arredondar nem chamar o gateway. Os quatro casos agora falham por tipo de exceção.

**Limite:** esta observação é de contrato/robustez no adapter. Não se demonstrou HTTP 500, liquidação incorreta ou exposição de dados: a orquestração captura RuntimeException e o resultado HTTP depende do caminho exercitado. **Encaminhamento:** validar escala/faixa antes da conversão ou traduzir a falha para erro de validação do domínio, mantendo o gateway sem chamadas. Nenhuma correção de produção foi feita nesta revisão.

### OBS85-03 — Rate limiting local não identificado

**Avaliação executada:** inspeção de SecurityConfiguration, filtros, controllers, dependências Gradle e configuração; não se identificou rate limiter de entrada da aplicação. LC-ABUSE-01 fez somente dez logins errados sequenciais e recebeu dez 401, sem sessão nem erro interno. LC-API-03 e LC-CONFIG-01 confirmaram limites de paginação e pool/fila; TC-API-13 já cobre limites multipart. Timeouts e repetição de saída têm casos próprios.

Não se inventou a regra "bloquear depois de dez tentativas", pois a equipe não definiu esse limiar. A ausência de limitador local é uma observação de configuração e recomendação de defesa, relacionada à [OWASP API4:2023](https://api-security.owasp.org/editions/2023/en/0xa4-unrestricted-resource-consumption/); o teste limitado não prova ausência de proteção AWS nem resistência a DoS. Definir budgets por identidade/origem/fluxo e avaliar a configuração do gateway na etapa autorizada correspondente. Nenhum teste de saturação foi realizado.

### SEC85-11 — Tarifa em token gravada no campo de BRL sem conversão

**Prioridade:** média para consistência financeira/contábil, sem prejuízo real demonstrado. **Evidência:** seção 3.6, `fee-persistence.json` e subcasos `*-fee-persistence`. **Código no HEAD da etapa externa:** `BlindPayQuoteProvider.quoteFundedWithBrl` retorna tarifa em token; `TransferExecutionService.prepareBlindPay` copia `offer.getTotalFee()` para `serviceFeeBrl` sem conversão.

No exemplo sintético recuperado, a tarifa de 15,10 USDB foi persistida como `service_fee_brl=15.10`, junto de origem de 100,00 BRL e destino de 2,49 USD. A unidade vem do contrato OpenAPI atual, não de suposição baseada no sender_amount. O problema é rotular o escalar do token como BRL; a hipótese anterior de soma de moedas distintas foi retirada. Não se demonstrou cobrança indevida pelo provedor, liquidação em produção ou perda real.

**Recomendação:** preservar a moeda de cada tarifa, distinguir cobranças por operação de parcelas faturadas posteriormente e converter explicitamente com regra de câmbio/arredondamento definida antes de persistir em campos BRL. Nenhuma correção de produção foi realizada.

### Complementação dos achados com evidência Avenia sandbox

#### TOTP financeiro ausente na confirmação — SEC85-01

Reprodução segura: preparar usuário local com TOTP ativo e autenticação real, obter oferta própria pela API e chamar `POST /v1/quotes/offers/{OWN_OFFER}/confirm` apenas com os montantes confirmados. O ticket de sandbox foi criado. Repetir a mesma confirmação não criou outro efeito, mas esse controle não substitui o segundo fator.

Referências do código avaliado: [TransferExecutionController](https://github.com/AGES-V-Stable/backend/blob/test/85-seguranca-avenia-sandbox/src/main/java/ages/vstable/backend/controller/TransferExecutionController.java), [TransferExecutionService](https://github.com/AGES-V-Stable/backend/blob/test/85-seguranca-avenia-sandbox/src/main/java/ages/vstable/backend/service/transfer/TransferExecutionService.java) e `TransferService.createConfirmedOffer`, que não valida o fator como `TransferService.create`.

Correção recomendada: exigir e validar o fator financeiro em todos os caminhos que podem submeter a transferência, com consumo/replay e concorrência tratados de forma compartilhada e atômica. Classificação herdada: CWE-287, OWASP API2/API6:2023. A eficácia da correção exige reteste separado.

#### Precisão do histórico — SEC85-07

Reprodução segura: confirmar uma oferta de destino sintético 1,123456 USDC e comparar `transfer_executions.target_amount` com `base_transactions.foreign_amount`. A primeira coluna preservou seis casas úteis; a segunda possui escala 2 em [BaseTransactionEntity](https://github.com/AGES-V-Stable/backend/blob/test/85-seguranca-avenia-sandbox/src/main/java/ages/vstable/backend/entity/BaseTransactionEntity.java).

Correção recomendada: definir precisão por ativo e preservar os montantes efetivos em todo o histórico e na execução; revisar schema, DTOs e arredondamento em remediação própria. Não arredondar cripto como fiat. Relação com CWE-197/CWE-682; o teste não demonstrou desvio de dinheiro real.

#### Acompanhamento Avenia ausente — VULN-11 da #83

O polling do teste comprovou o estado PAID, mas não conciliou a aplicação. `TransferExecutionReconciler` seleciona apenas BLINDPAY; `TransferService` documenta que persiste somente o snapshot da criação. Não há receptor Avenia nos controllers deste HEAD.

Recomendação: implementar acompanhamento autorizado por consulta ou webhook autenticado, com validação de propriedade, deduplicação, ordenação e transições válidas. Tratar estados incertos consultando a operação original; `externalId` não é uma garantia de idempotência da Avenia. Essa ausência funcional não foi convertida automaticamente em CWE ou exploração comprovada.

## 7. Revisão dos apontamentos VULN-01 a VULN-11 da #83

| ID anterior | Resultado nesta avaliação | Evidência e limite |
|---|---|---|
| VULN-01 | Confirmado para confirmação inicial e recotação local | SEC85-01, TC-FIN-01/20; recotação até estado COMPLETED com payout simulado, sem movimentação real. |
| VULN-02 | Confirmado no serviço local | SEC85-02; comportamento AWS não executado. |
| VULN-03 | Observação estática mantida em parte | Allowlist de `LambdaSecretsEnvironmentPostProcessor.java:24` não inclui TWO_FACTOR_ENCRYPTION_KEY; nenhum deploy Lambda foi realizado. AI_SUPPORT_API_KEY não é requisito funcional do HEAD sem chatbot. |
| VULN-04 | Não revalidado | Exposição de credencial em issue pública foi herdada do planejamento; a credencial não foi consultada, testada nem reproduzida aqui. Gestão de segredos fica na análise correspondente do épico. |
| VULN-05 | Não comprovado como BOLA geral | Não se identificou RLS nas migrations; Terraform liga a aplicação ao master_username. Isso aumenta dependência dos controles da aplicação e privilégio potencial, mas não comprova acesso cruzado. Os testes horizontais executados passaram. |
| VULN-06 | Conclusão anterior substituída | A segregação SUPPORT/COMPLIANCE_ANALYST/SUPER_ADMIN não é requisito da nova matriz. SEC85-03 passa a avaliar a proibição de decisão manual de compliance, inclusive para Admin; resultados antigos não comprovam o novo requisito. |
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

# Para uma nova captura completa, preserve os snapshots anteriores em outro diretorio.
# Execute antes o diagnostico completo, o build e a regressao frontend acima/abaixo.
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\export-security-evidence.ps1 -Phase assessment -OutputDirectory .\docs\security\evidencias-85\nova-captura
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\export-security-evidence.ps1 -Phase integration-retest -OutputDirectory .\docs\security\evidencias-85\nova-captura
Copy-Item .\build\reports\security-assessment\endpoints.csv .\docs\security\evidencias-85\nova-captura\endpoints.csv
Copy-Item .\build\reports\security-assessment\endpoints-perfis.csv .\docs\security\evidencias-85\nova-captura\endpoints-perfis.csv
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\export-security-coverage.ps1 -EvidenceDirectory .\docs\security\evidencias-85\nova-captura
```

```powershell
# A partir de frontend/; dependências fixadas no package-lock.json.
npm ci --ignore-scripts --no-audit --no-fund
npm run test:coverage -- --maxWorkers=2 --reporter=default --reporter=junit --outputFile.junit=coverage/security-baseline.xml
```

O uso de ExecutionPolicy Bypass limita-se ao processo que executa o exportador e não altera a política global do Windows. Esse exportador não faz requisições às APIs.

**Resultado da revisão do PR, seção 3.7:** diagnóstico com 27 FAIL; a captura de 21 FAIL da seção 3.5 e a revalidação no HEAD da etapa externa da seção 3.6 são execuções distintas. Não use `ignoreFailures` para apresentar o diagnóstico como aprovado. A tarefa não faz parte de `build/check` nem do PIT; a suíte funcional permanece separada para distinguir regressão e diagnóstico. A integração dos testes corrigidos ao gate deve ser tratada na remediação.

Execuções filtradas sobrescrevem os XMLs daquela tarefa; para regenerar o conjunto completo de evidências, rode a tarefa sem `--tests` antes de exportar. Arquivos exportados de execuções antigas não devem ser confundidos com reteste de uma correção.

### 8.1.1. Reprodução específica do sandbox BlindPay

Com Docker/Java 25 e `BLINDPAY_API_KEY`/`BLINDPAY_INSTANCE_ID` já carregadas com segurança no mesmo processo, executar a partir de backend:

```powershell
.\scripts\run-blindpay-security-sandbox.ps1
```

O script usa a tarefa manual `blindpaySecuritySandboxTest`, prepara segredos exclusivos da aplicação efêmera e escolhe diretório novo. Não inserir credenciais ou IDs externos no comando, relatório ou versionamento. As variáveis opcionais `BLINDPAY_SANDBOX_CUSTOMER_ID` e `BLINDPAY_SANDBOX_BANK_ACCOUNT_ID` não são requisitos da preparação específica desta avaliação; o executor verifica a massa de demonstração e cria conta própria. A tarefa antiga `blindpaySandboxTest` conserva seus requisitos originais e não foi executada integralmente por engano. Procedimento completo e limitações no relatório complementar do backend.

### 8.1.2. Reprodução específica do sandbox Avenia

Arquivos de instrumentação adicionados:

| Arquivo | Finalidade |
|---|---|
| `AveniaSandboxPreflightTest.java` | Consultas iniciais de conta, acesso, metadados, limites e callbacks |
| `AveniaSecuritySandboxTest.java` | Aplicação HTTP local, banco efêmero, isolamento A/B, cotações e tickets reais |
| `AveniaSandboxSession.java` | Fronteira de rede, orçamento, assinatura pelo cliente existente e checkpoints sanitizados |
| `AveniaSandboxBoundaryTest.java` | Contraprovas locais para destino, extração de flags e classificação sanitizada de erros |
| `scripts/test-avenia-security-sandbox.ps1` | Segredos exclusivos do processo, execução manual, hashes e bloqueio de reenvio |
| `build.gradle` | Tarefa opt-in e exclusão explícita das suítes padrão/PIT |
| Relatório, JSONs e atualização da wiki | Resultados rastreáveis, condições, recomendações e limitações |

Pré-requisitos: Java 25, Docker funcionando, novas chaves locais protegidas pelo Git e acesso autorizado à conta sandbox esperada. Não configurar MFA novamente para chamadas assinadas.

Na branch de testes, em `backend/`, para **uma nova campanha explicitamente escolhida**:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/test-avenia-security-sandbox.ps1 -Phase preflight -Campaign revisao-20261114
# Conferir o preflight antes de prosseguir.
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/test-avenia-security-sandbox.ps1 -Phase assessment -Campaign revisao-20261114
```

Os comandos criam recursos novos de teste. Não executar novamente para apenas visualizar resultados; abrir os JSONs e este relatório. A campanha `initial` contém esta captura e bloqueia reenvios. `business` é uma continuação controlada usada na investigação, com recuperação explícita da subconta própria; não é necessária no fluxo final normal. O teto é 50 chamadas por campanha e os limites de tempo são 5 minutos para preflight e 15 para cada etapa posterior. Iniciar outra campanha não é autorização para ignorar operações anteriores incertas.

A tarefa `aveniaSecuritySandboxTest` exige opt-in e está excluída de `test`, `build`, CI padrão e PIT. O executor gera segredos próprios para JWT/TOTP/convites no processo, sem usar os compartilhados. Logs brutos e JUnit permanecem em `build/` ignorado e não foram exportados como evidência. As revisões finais do runner foram verificadas localmente; não se repetiram as operações remotas apenas para atualizar hashes.

Validação sem APIs reais: `gradlew.bat build --no-daemon --max-workers=1 --console=plain`, com opt-in falso, chaves vazias e bases de provedores em loopback. **Saída 0: 1.522 casos em 119 suítes, 1.516 PASS, zero falhas/erros e 6 ignorados.** Inclui os três testes locais da fronteira de sandbox/sanitização. Os testes externos são excluídos por tag, não contados como aprovações da regressão. Resultado e hashes em `validation.json`. A sintaxe PowerShell e a recusa de reenvio de campanha já executada também passaram, sem chamadas HTTP.

A conferência posterior das duas cotações sanitizadas também verificou tarifas não negativas, todas em BRL, soma menor que a entrada e resíduo de arredondamento até um centavo de BRL entre entrada líquida e destino × preço. Os detalhes estão em `validation.json`. É análise das respostas capturadas, sem chamada adicional, e não valida todos os cálculos defensivos do backend ou todas as moedas.

#### Recursos e recuperação Avenia

Foram deixados **uma subconta INDIVIDUAL própria, duas carteiras beneficiárias próprias (GNOSIS/POLYGON) e dois tickets PAID**. As carteiras usam endereços aleatórios não financiados. Nenhuma subconta/carteira preexistente da equipe foi modificada ou excluída. Subcontas são permanentes segundo a documentação do provedor.

IDs e referências para recuperação estão exclusivamente em `.env.avenia-security/assessment-resources-*.json`, `business-resources-*.json`, `assessment-ticket-*.json` e `business-ticket-*.json`, ignorados pelo Git. A versão final também grava intenções privadas antes de POSTs. Não publicar essa pasta. Os JSONs versionados usam aliases e não permitem ao revisor recuperar identificadores sensíveis.

O executor preserva tentativas anteriores, limita chamadas/polling e bloqueia reenvio após escrita cujo resultado não foi recebido. Após queda, revisar `RUNNING`/`INTERRUPTED`, os registros privados e o provedor antes de iniciar outra operação. Não inferir ausência de ticket pela expiração do token. `FINISHED` na avaliação e saída 1 do Gradle são compatíveis com asserções FAIL; não significam processo ainda em andamento.

### 8.2. Conferência manual recomendada

O navegador não substitui os testes de API: ocultar um botão não impede chamadas diretas. Para revisão pelo responsável, o caminho mais seguro é rodar os grupos acima e abrir `backend/build/reports/tests/securityAssessmentTest/index.html`, acompanhando estes casos:

1. TC-AUTH-02: primeira e segunda instâncias recebem o mesmo código; observar a ausência da rejeição na segunda.
2. TC-RBAC-01–09: comparar status HTTP, persistência e permissões anunciadas com a matriz revisada, exclusivamente na base efêmera. Incluir VIEWER como somente leitura, preservar a distinção de OWNER e OPERATOR e separar a conta Admin dos usuários empresariais. Não voltar a exigir segregação entre os níveis administrativos antigos.
3. TC-FIN-01 e TC-CONC-02: distinguir falta de TOTP da unicidade de execução; um controle não substitui o outro.
4. TC-INT-01/02: acompanhar a aceitação de assinatura válida e a rejeição do corpo adulterado, sem cadastrar webhooks em terceiros.
5. TC-API-05: verificar que os arquivos contêm apenas prefixos e que o retorno é de um cliente simulado.

Essas conferências pelo responsável ainda **não foram realizadas** e não são registradas como evidência manual executada. Não é necessário usar dados pessoais, configurar credenciais reais ou transferir dinheiro para reproduzir os testes.

## 9. Limitações e riscos residuais

1. **Camadas e provedores:** as capturas locais de 10–12/11 usaram mocks e não comprovam contratos ou efeitos reais de terceiros. A seção 3.6 separa as chamadas funcionais reais à BlindPay Development de 13/11. Avenia real não foi exercitada na etapa BlindPay, mas foi avaliada separadamente na seção 3.8. AWS e chatbot não foram exercitados nessas etapas externas. Não houve recebimento de webhook real, túnel ou alteração de dashboard.
2. **Concorrência:** plano local executado: cinco confirmações de oferta, cinco entregas do mesmo webhook, confirmação/recotação versus conciliação, consumo TOTP intercalado e expiração entre checagem/submissão. Não é prova formal de liquidação única distribuída, nem execução em múltiplas Lambdas. Na corrida de payout a evidência é a chave estável, não a alegação de uma única chamada HTTP ao provedor.
3. **Recotação:** a captura local até COMPLETED usa payout simulado. A complementação BlindPay identifica separadamente expiração provocada no banco efêmero, recotação real e eventual aceitação explícita, conforme os estados observados. Na seção 3.8, a confirmação Avenia gerou cotação fresca e ticket de sandbox real; não se induziu preço pior, expiração ou timeout no provedor. Não se afirma liquidação em produção.
4. **Valores/moedas/tarifas:** a matriz TC-FIN-13–19 cobre casos representativos, fronteiras de centavos/long, sinal de tarifas e persistência fiat/cripto. Os defeitos encontrados são FAIL, não ausência de teste. Novos ativos/regras de arredondamento ou limites de negócio demandam manutenção da matriz; não se afirma cobertura de todas as combinações matematicamente possíveis.
5. **Imutabilidade:** COMPLETED/FAILED/REFUNDED foram exercitados em confirmação, recotação, start e reconciliação com comparação integral do registro de execução. Isso não é uma proteção contra escrita SQL direta por um administrador de banco nem comprova comportamento interno de terceiros.
6. **Recuperação/sessão:** recuperação, logout no servidor e troca de senha via endpoint não existem no HEAD. Limpeza do cliente é coberta pelo Vitest; a alteração de senha foi simulada no banco isolado e demonstrou sessão antiga ainda aceita. A lacuna de revogação está registrada em SEC85-06.
7. **Uploads:** serviço e limites reais do parser HTTP exercitados. Antivírus, PDFs ativos e documentos pessoais não foram usados; o teste de documento truncado não comprova execução de conteúdo malicioso.
8. **SSRF/injeção/abuso:** UUID, filtros, autoridade/path de cliente, redirects com builder real, erros por operação, paginação, pool/fila e repetição limitada foram executados. Avenia seguiu redirect entre origens, SEC85-10. A revisão dos sinks não identificou fetch genérico de URL escolhida pelo usuário nos serviços examinados; não se declara ausência universal de SSRF/XSS/injeção. Nenhum stress/DoS foi feito; ausência de rate limiter local não prova ausência de controle externo. OBS85-03 registra a avaliação e a necessidade de política de limitação.
9. **Ignorados preexistentes:** `OnboardingIntegrationTest` possui dois casos desabilitados e `TransactionQueryIntegrationTest` quatro. Isso é configuração `@Disabled`, não falha do Docker nesta execução. Casos novos cobrem cadastro e isolamento do histórico, sem alegar reativação ou equivalência integral aos seis ignorados.
10. **Convites/publicação:** em 12/11 a equipe informou desenvolvimento do cadastro/convite de operadores; as permissões foram avaliadas por vínculos sintéticos. O HEAD usado para sandbox em 13/11 já contém código de convites, mas o fluxo permanece fora desta complementação por decisão de escopo, reservado para avaliação própria. Não se afirma que ele continua ausente nesse HEAD. Não houve commit/push/PR, correção de produção ou achado marcado como remediado por esta execução.

Os itens parciais das tabelas não recebem automaticamente CWE: ausência de funcionalidade, cobertura incompleta e limitação de ambiente não demonstram por si só uma fraqueza explorável. As fragilidades confirmadas têm classificação, evidência e reprodução na seção 6.

### 9.1. Pendências e limites da complementação Avenia

- **Webhooks reais:** recebimento, assinatura, falsificação, replay, duplicidade e ordem não executados; polling não os substitui.
- **Falha financeira/cancelamento:** não foram criados cenários financeiros terminais FAILED/CANCELED. O simulador documentado de bloqueio bancário exige payout PIX, fora da rota BRL/PIX → carteira do adapter atual. Não foi forçada uma corrida de cancelamento contra PIX automaticamente pago.
- **Resultado incerto e TOCTOU:** não houve timeout ou resposta perdida, nem falhas/tempo manipulados no provedor. Permanecem testes defensivos locais; esta etapa não prova recuperação real de um ticket incerto.
- **Validade e tarifas:** tokens frescos foram aceitos e a confirmação recotou pelo caminho de produção; não se induziu expiração/reuso externo nem preço pior no sandbox. Tarifas reais foram observadas com suas moedas; não se afirma validação universal de toda tarifa/ativo/combinação.
- **Onboarding:** não houve envio de documentos, KYC/KYB integral ou aprovação artificial. O sandbox liberou BRL na subconta nova; o vínculo local permaneceu PENDING. Isso não demonstra elegibilidade institucional em produção.
- **Fluxos excluídos:** cadastro/convites de operadores, BlindPay real, AWS, chatbot e produção não foram reavaliados.

Assim, a integração de saída Avenia está demonstrada nas rotas acima e dois achados locais ganharam reprodução com provedor real. O DoD integral da #85 continua parcial, em especial quanto a eventos externos e demais requisitos não demonstrados.

## 10. Checklist do DoD da #85

As caixas distinguem os casos do plano local das exigências integrais da issue. A revisão da seção 3.7 corrige testes e rastreabilidade: referências genéricas de componente e inventário do perfil padrão não comprovam todos os contratos/autorização por endpoint. A matriz nova admite PARCIAL e NAO_AVALIADO e separa cobertura de resultado do controle. Um FAIL executado não é um teste pendente. A cobertura externa permanece parcial, mesmo após a etapa BlindPay.

- [x] **Autenticação, autorização, sessão e isolamento avaliados no plano local.** R01/R02/R07: ciclo 2FA completo, JWT, sessão/revogação, três perfis × rotas administrativas, seis operações de KYC por propriedade, duas rotas de cotação, recursos financeiros e mass assignment; regressões de vínculo e frontend reexecutadas. Recuperação/logout ausentes registrados, sem inventar execução.
- [ ] **Endpoints/APIs — cobertura parcial.** Na branch do PR local: 59 operações com casos limitados de entrada/erro e barreiras de acesso; duas rotas de jobs condicionais inventariadas separadamente. Referências aos serviços/controllers não equivalem a todos os contratos de cada rota. Exposição de implantação/IAM e contexto HTTP migration não avaliados. Na etapa externa, os 71 endpoints pertencem a outro HEAD; as capturas não são somadas como cobertura integral.
- [x] **Fluxos existentes e controles contra abuso avaliados com dados sintéticos.** R04/R05/R08/R13: cadastro, empresas, representantes, compliance, beneficiários, cotações, transferências, repetição, validação, paginação, pool/fila e uploads; ausência de limiter local registrada em OBS85-03. Convite/cadastro de operadores ainda indisponível, fora da execução até sua entrega.
- [x] **Integridade financeira avaliada pela matriz finita.** R06/R07: valores negativos/zero/precisão/overflow/moedas/rotas/tarifas, validade/margem, oferta selecionada, empresa-beneficiário-cotação-execução, propriedades protegidas e persistência. Perda de precisão/tarifa negativa permanecem FAIL. Os testes não exigem explorar infinitas combinações monetárias.
- [x] **Idempotência, concorrência, TOCTOU e estados avaliados localmente.** R08/R09/R10: confirmação concorrente, webhook duplicado, recotação versus conciliação, TOTP e relógio controlado; regressão da máquina de estados, falha/reembolso/retries e imutabilidade terminal. SEC85-09 registra a falha encontrada; liquidação distribuída real não foi simulada como prova de produção.
- [ ] **Integrações e webhooks — cobertura externa parcial.** R11/R12/R14 têm a cobertura local documentada e a complementação BlindPay Development da seção 3.6. A integração de saída Avenia real foi complementada nas seções 3.8 e 5.7. Recebimento/autenticidade/replay/ordenação de webhooks externos e os demais limites da seção 9.1 continuam não demonstrados. Testes defensivos de assinatura, duplicidade, erros, timeouts e redirects não foram confundidos com chamadas funcionais bem-sucedidas. Não se implementou integração ausente apenas para marcar o checklist.
- [x] **Resultados, evidências e limitações registrados para cada caso executado.** Seções 3–5 e arquivos JSON sanitizados; casos individuais da regressão e do diagnóstico enumerados.
- [x] **Vulnerabilidades reproduzíveis registradas com passos seguros.** Seções 6 e 8; sem movimentação real nem exposição de segredos.

A complementação local da seção 3.5 permanece como captura própria. A seção 3.6 complementa a BlindPay e as seções 3.8 e 5.7 complementam as rotas autorizadas da Avenia; o encerramento integral da #85 ainda depende das pendências externas e da avaliação separada dos fluxos excluídos. Não se apresentam webhook real ou convites como testados nesta complementação. Corrigir os achados e comprovar sua remediação pertence à etapa de correção/reteste.
