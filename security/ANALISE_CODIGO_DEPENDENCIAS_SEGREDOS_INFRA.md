# Relatório Consolidado — Issue #84: Análise de Código, Dependências, Segredos e Infraestrutura

- **Issue:** [`AGES-V-Stable/cicd#84`](https://github.com/AGES-V-Stable/cicd/issues/84) (Epic `#80`)
- **Data:** 2026-11-12
- **Coordenação (Tech Lead):** Gemini 4 Hydrogen (High)
- **Especialistas (Swarm):** `backend-sec` (Hydrogen Medium), `frontend-sec` (3.8 Flash Medium), `devsecops-cicd` (Hydrogen Medium)
- **Referências:** OWASP ASVS 5.0.0, OWASP SAMM 2.0, OWASP CI/CD Security Cheat Sheet, NIST SP 800-218 (SSDF 1.1), CWE, CIS AWS Benchmarks, AWS Well-Architected Security Pillar.

---

## 1. Resumo Executivo

A auditoria multidisciplinar cobriu os 3 repositórios (`backend`, `frontend`, `cicd`), configurações do **SonarCloud**, dependências (**Gradle** e **npm**), pipelines **GitHub Actions**, módulos **Terraform** (`staging`, `production`, `global`) e varredura de segredos no código e histórico Git.

**Principais conclusões:**
1. **Qualidade & SonarCloud:** O projeto `AGES-V-Stable_backend` encontra-se com **Quality Gate em `ERROR`** (`security_rating: 4.0`, cobertura `87.7%`) devido a **2 vulnerabilidades abertas** (`java:S4502` Crítica em `InternalJobSecurityConfiguration.java:30` e `githubactions:S8233` Major em `.github/workflows/deploy.yml:11`). No frontend, há divergência entre a chave em `sonar-project.properties` (`ages-v-stable_frontend`, 404 na API) e `AGES-V-Stable_frontend` (sem análise de cobertura/alert_status retornada).
2. **Vulnerabilidades Críticas e Altas de Aplicação:** Confirmado **bypass de 2FA (TOTP)** na confirmação de ofertas (`/v1/quotes/offers/{offerId}/confirm` e `requote/confirm`), estado de **anti-replay/lockout TOTP e rate-limit** armazenados em `ConcurrentHashMap` em memória volátil no Lambda, **RBAC administrativo colapsado** em `ROLE_ADMIN`, ausência de **PostgreSQL RLS** com conexão Lambda como `vstable_admin` (`rds_superuser`), e omissão de `TWO_FACTOR_ENCRYPTION_KEY`/`AI_SUPPORT_API_KEY` em `LambdaSecretsEnvironmentPostProcessor` e Terraform `provider_secrets`.
3. **Dependências & Supply Chain:** Frontend possui **0 vulnerabilidades** em dependências de produção (`npm audit --omit=dev`), mas 13 moderadas em `devDependencies` mascaradas por `--audit-level=high` e **checks comentados no `deploy.yml`**. Backend possui *version skew* (`spring-security-core:7.0.5` vs `7.1.1`) e nenhum plugin SCA/SBOM no Gradle. Pipelines não pinam Actions por SHA completo (exceto `configure-aws-credentials`) e imagens ECR usam `--provenance=false` sem SBOM nem gate de CVE.
4. **Infraestrutura AWS (IaC):** RDS configurado sem backups (`backup_retention_period = 0`, `skip_final_snapshot = true`, `multi_az = false`), `production/variables.tf` com default de `avenia_base_url` apontando para sandbox, CloudFront sem *Response Headers Policy* (CSP/HSTS/XFO) e API Gateway HTTP sem throttling, access logs ou AWS WAF.
5. **Segredos no Git:** Nenhum segredo real ou chave privada foi encontrado nos arquivos ou histórico Git dos 3 repositórios (permanece válida apenas a exposição prévia em metadado da Issue `#74`, registrada para rotação).

---

## 2. SonarCloud (Configuração, Cobertura e Achados)

| Repositório | Config Versionada | Métricas Públicas (SonarCloud API) | Achados / Discrepâncias |
| :--- | :--- | :--- | :--- |
| **Backend** | `build.gradle` (`org.sonarqube` `7.3.1.8318`, key `AGES-V-Stable_backend`, `jacocoTestReport.xml`) | **Quality Gate:** `ERROR`<br>**Coverage:** `87.7%`<br>**Security Rating:** `4.0`<br>**Vulnerabilities:** `2` (`0` Hotspots, `0` Bugs) | 1. **`java:S4502` (CRITICAL):** `InternalJobSecurityConfiguration.java:30` desabilita CSRF para `POST /events` (`@Profile("migration \| seed")`).<br>2. **`githubactions:S8233` (MAJOR):** `.github/workflows/deploy.yml:11` define `id-token: write` no nível do workflow em vez de por job.<br>3. `sonar.qualitygate.wait=true` não está configurado no build/CI. |
| **Frontend** | `sonar-project.properties` (key `ages-v-stable_frontend`, `coverage/lcov.info`) | **Key `AGES-V-Stable_frontend`:** `0` Bugs, `0` Vulnerabilities, `0` Hotspots, `security_rating: 1.0` (sem `coverage`/`alert_status` retornados). **Key `ages-v-stable_frontend`:** `404` | Divergência de `projectKey` em `sonar-project.properties` vs organização/key consultada e ausência de `sonar.qualitygate.wait=true`. |

---

## 3. Dependências Diretas e Transitivas (SCA)

### 3.1. Frontend (`npm`)
- **Execução:** `npm audit --omit=dev` -> **0 vulnerabilidades** em runtime de produção.
- **Achado (Médio):** `npm audit` completo apontou **13 vulnerabilidades moderadas** em dependências de desenvolvimento (`@vitest/mocker` GHSA-qxg6-89mg-rqv8 / GHSA-xp26-3454-mjgp; `minimatch` GHSA-g3hv-75wg-m249). O script `package.json:17` usa `npm audit --audit-level=high`, mascarando severidades moderadas.

### 3.2. Backend (Gradle)
- **Versões resolvidas (`runtimeClasspath`):** Spring Boot `4.1.1`, JJWT `0.12.6`, PostgreSQL JDBC `42.7.13`, Flyway `12.4.0`, PDFBox `3.0.8`, AWS SDK BOM `2.55.14`, Tomcat `11.0.24`, Netty `4.2.17.Final`, SnakeYAML `2.6`, Jackson 3 (`3.1.5`) + Jackson 2 (`2.21.5`).
- **Achado 1 (Médio — Dependency Skew):** `build.gradle:52` fixa `spring-security-core:7.0.5` enquanto `spring-security-web` e `spring-security-config` resolvem para `7.1.1`.
- **Achado 2 (Médio — Ausência de SCA/SBOM):** Nenhuma task de verificação (`./gradlew tasks --group="verification"`) executa `dependency-check-gradle` ou geração de SBOM.

---

## 4. Detecção de Segredos e Tratamento de Exposições

- **Varredura realizada:** Inspeção em `git ls-files`, arquivos `.env*`, histórico Git (`git log -G` / `-p`) nos 3 repositórios para padrões de chaves (`AKIA`, `AIza`, `sk-`, `ghp_`, `glpat-`, `PRIVATE KEY`, `.pem`, `.tfvars`, `.p12`, `.pfx`, `.jks`).
- **Resultado no código/Git:** **Limpo** nos 3 repositórios (valores encontrados em workflows de teste são placeholders sintéticos `test-only-*`; `tfvars.example` contém apenas placeholders).
- **Exposição prévia registrada (`VULN-04` do doc #83):** Credencial e URL de staging expostas no texto da [Issue `#74`](https://github.com/AGES-V-Stable/cicd/issues/74). **Recomendação:** rotação imediata da credencial e sanitização do histórico da issue (sem reproduzir valores secretos).
- **Lacuna de proteção local:** `.gitignore` do frontend não possui regras explícitas para variações como `.env.production` ou `.env.staging`.

---

## 5. Revisão Contextual de Código Sensível

| ID | Severidade | Componente / Arquivo | Achado & Condição de Exploração | Impacto | Recomendação Preliminar |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **APP-01** | **Crítica** | `TransferExecutionController.java` L29-47, L63-78; `TransferExecutionService.java` L115-146 | `POST /v1/quotes/offers/{offerId}/confirm` e `requote/confirm` executam remessas sem exigir/validar TOTP (`TwoFactorAuthService.validateTransferTotp`). | Execução de transferências reais sem 2º fator por qualquer sessão `TRANSFER_CREATE` comprometida. | Exigir e validar TOTP antes da confirmação/requote. |
| **APP-02** | **Alta** | `TwoFactorAuthService.java` L39-41; `SupportChatService.java` L44 | Anti-replay TOTP, lockout (5 tentativas) e rate-limit (15 req/min) armazenados em `ConcurrentHashMap` em memória volátil. | Ineficaz em AWS Lambda concorrente ou após *cold start* (replay em 90s e bypass de brute-force/rate-limit). | Persistir estado em PostgreSQL (ou DynamoDB/Redis). |
| **APP-03** | **Alta** | `LambdaSecretsEnvironmentPostProcessor.java` L24-27; `TwoFactorSecretConverter.java` L29-35 | `REQUIRED_KEYS`/`OPTIONAL_KEYS` omite `TWO_FACTOR_ENCRYPTION_KEY` e `AI_SUPPORT_API_KEY`. | Falha de inicialização no profile `lambda` ou ausência de chave AES-256-GCM / IA. | Incluir ambas as chaves no post-processor e no Secrets Manager IaC. |
| **APP-04** | **Alta** | `AdministratorEntity.java` L48-51, L78-80 | Todos os `AdminAccessLevel` (`SUPER_ADMIN`, `COMPLIANCE_ANALYST`, `SUPPORT`) recebem apenas `ROLE_ADMIN`. | `SUPPORT` pode aprovar compliance (`PATCH .../compliance-status`) ou excluir empresas (`DELETE /v1/companies/{id}`). | Mapear authorities por `AdminAccessLevel` e restringir endpoints. |
| **APP-05** | **Alta** | Migrations `V1`–`V11`; `CompanyAccess` | Ausência total de `ENABLE ROW LEVEL SECURITY` / políticas RLS no PostgreSQL. | Qualquer falha em filtro JPA resulta em leitura/alteração cross-tenant (IDOR/BOLA). | Habilitar RLS nas tabelas de negócio combinado a role DB não-superuser. |
| **APP-06** | **Alta** | Frontend `src/shared/http/authToken.ts:18`; `src/services/quotes.ts:56` | JWT de sessão armazenado em `localStorage`. | Qualquer XSS ou script de terceiro malicioso sequestra a sessão. | Migrar para cookie `HttpOnly; Secure; SameSite=Strict`. |
| **APP-07** | **Média** | `SecurityConfiguration.java` L78, L81 | Swagger (`/swagger-ui/**`, `/v3/api-docs/**`) exposto publicamente sem bloqueio em prod; rota órfã `/user/exists`. | Enumeração completa de contratos internos da API. | Desabilitar Swagger em `production` e remover rota órfã. |
| **APP-08** | **Média** | `GlobalExceptionHandler.java` L27-32, L70-82, L140-174; `AuthController.java` L50 | Retorno direto de `ex.getMessage()` ao cliente; `AuthRequestDTO` sem `@Valid`. | Vazamento de mensagens internas/integrações Avenia/BlindPay. | Retornar mensagens genéricas sanitizadas e aplicar `@Valid`. |
| **APP-09** | **Média** | Frontend `storedAccount.ts:18-20`, `AppRoutes.tsx:81`, `quotes.ts:6,56,67`, `index.html`, `App.tsx` | Guardrail admin via claim client-side; `quotes.ts` ignora `httpClient.ts`; ausência de `<meta>` CSP e `ErrorBoundary`. | Exposição de UI admin via tamper local, quebra de padrão HTTP e maior superfície XSS. | Validar role exclusivamente via backend `/me`, unificar `quotes.ts` em `httpClient.ts` e adicionar CSP/ErrorBoundary. |

> **Complemento (2026-11-14):** Os achados `APP-10` a `APP-13` da revisão complementar do fluxo de upload de documentos, imagens e metadados estão detalhados na [Seção 8](#8-auditoria-complementar--fluxo-de-upload-de-m%C3%ADdia-documentos-e-metadados-2026-11-14).

> **Controles positivos verificados:** `TwoFactorSecretConverter` (AES-256-GCM IV 12B `SecureRandom` + tag 128b), `BlindPayWebhookVerifier` (HMAC-SHA256 tempo constante + janela 5 min), `JwtTokenUtils` (HS512, bloqueio de `challengeToken`), `@JsonIgnore`/`@ToString.Exclude` em `twoFactorSecret`, e **0 sinks de XSS** (`dangerouslySetInnerHTML`, `innerHTML`, `eval`) no frontend.

---

## 6. Infraestrutura Terraform, AWS e Pipelines CI/CD

### 6.1. Recursos AWS (Baseado em IaC Versionada — `cicd` `940f9d1`)
- **RDS & Banco (`modules/database/main.tf`, `production/main.tf`):** `multi_az = false`, `backup_retention_period = 0`, `delete_automated_backups = true`, `skip_final_snapshot = true`, `performance_insights_enabled = false`. Role restrita foi removida (`removed { from = module.database_role }`), fazendo o Lambda conectar como `vstable_admin` (`rds_superuser`).
- **Secrets Manager & Lambda (`modules/api/main.tf`, `variables.tf`):** `provider_secrets` não aceita `TWO_FACTOR_ENCRYPTION_KEY` nem `AI_SUPPORT_API_KEY`; Lambda `backend` sem `reserved_concurrent_executions` (exaustão de conexões no `db.t4g.micro`); `production/variables.tf:99` mantém default de `avenia_base_url` apontando para `api.sandbox.avenia.io:10952`.
- **API Gateway (`modules/api/main.tf`):** Stage `$default` sem `throttling_burst_limit`/`throttling_rate_limit`, sem `access_log_settings` e sem AWS WAF.
- **S3 / CloudFront (`modules/frontend/main.tf`):** Positivo: S3 privado com OAC (`sigv4`), `AES256` e `TLSv1.2_2021`. Negativo: sem `response_headers_policy_id` (faltam CSP, HSTS, `X-Frame-Options`, `X-Content-Type-Options`), sem access logs e sem WAF.
- **VPC / Rede (`modules/network/main.tf`):** 1 NAT Gateway apenas (SPOF zonal); egress `0.0.0.0/0` protocolo `-1` na SG do Lambda; sem VPC Flow Logs nem VPC Endpoints.
- **IAM & State:** Estado Terraform local (sem backend S3 remoto criptografado + locking). Em `production`, `backend_deploy_role` ainda concede acesso ao `seed_lambda_arn`. Positivo: Trust policies OIDC usam `sub` com IDs numéricos e `ref` separado (`refs/heads/main` staging vs `refs/tags/*` prod).

### 6.2. Pipelines GitHub Actions, Integridade de Artefatos, SBOM e ECR
- **Pinagem por SHA e Refs Mutáveis:** Apenas `aws-actions/configure-aws-credentials@e125382...` usa SHA completo. Todas as demais Actions (`checkout`, `setup-java`, `setup-node`, `upload-artifact`, `sonarqube-scan-action`) usam tags mutáveis (`@v4/@v6/@v7/@v8`), e `gitlab-sync.yml` referencia `reusable-gitlab-sync.yml@main` com `secrets: inherit`.
- **Workflow `backend/deploy.yml`:** Define `id-token: write` no nível do workflow (`githubactions:S8233`), porém **já atualiza e invoca `MIGRATION_LAMBDA_FUNCTION_NAME`** (atualização confirmada em relação ao doc `#83`), e restringe `seed` a `github.event_name == 'push'`.
- **Workflow `frontend/deploy.yml`:** Linhas 47-53 mantêm comentados `npm run audit`, `format:check`, `lint` e `test:coverage` antes do deploy.
- **ECR, Containers & SBOM:** `registry/main.tf` ativa `IMMUTABLE`, `AES256` e `scan_on_push = true` (positivo), mas não há lifecycle policy nem *gate* no deploy para bloquear CVEs críticas do ECR. O `buildx` utiliza `--provenance=false`, **não gera SBOM** (CycloneDX/SPDX) nem assina imagens (Cosign). `Dockerfile` e `Dockerfile.lambda` usam tags base mutáveis (`gradle:9-jdk25`, `eclipse-temurin:25-jre`), executam sem `USER` (`root`) e `Dockerfile.lambda` mantém `curl` instalado após baixar `rds-global-bundle.pem` sem verificação de checksum.
- **GitLab Sync & IaC CI:** `gitlab-sync.yml` dispara também em `pull_request` com push forçado default; repositório `cicd` não possui pipeline de `terraform plan`/scan IaC (`tfsec`/`checkov`/`trivy`).

---

## 7. Limitações de Acesso e Cobertura Registradas

1. **Conta AWS:** Não há acesso à conta AWS nem `aws` CLI/credenciais configuradas; a avaliação de IAM, API Gateway, Lambda, RDS, S3/CloudFront e Secrets Manager baseou-se integralmente no Terraform e configurações versionadas (`cicd@940f9d1`).
2. **GitHub Settings UI:** Não foi possível auditar configurações de *Branch Protection Rules*, *Environments* com aprovadores manuais e inventário de *Secrets* no painel do GitHub apenas pelos arquivos locais.
3. **Ferramentas Locais:** `terraform`, `trivy`, `tfsec`, `checkov`, `gitleaks` e plugins SCA Gradle não estavam instalados no ambiente local; a varredura de segredos e IaC foi feita via scripts Git/regex e inspeção direta, e a análise CVE de dependências Java não contou com consulta automática offline à base NVD.
4. **Execução de Testes Locais:** Auditoria realizada em modo estritamente somente leitura; no frontend, 2 suites de teste (`navigation.test.tsx`, `architecture.test.ts`) sofreram timeout de 15s sob overhead de coverage V8 no Windows (778 testes passaram). No backend, não foi executada a suíte completa de testes de integração (`./gradlew test`).

---

## 8. Auditoria Complementar — Fluxo de Upload de Mídia, Documentos e Metadados (2026-11-14)

Em 14/11/2026, durante a revisão de integração do módulo Rust `vstable-media-sanitizer` (branch `feat/bonus-rust-pdf-image-sanitizer`, commit `903ba67` / PR `#96`), foi conduzida uma inspeção complementar ponta a ponta no fluxo de upload de imagens e documentos (`frontend` em `origin/main` `4994656` / `ia-chatbot` `36feef9` -> `backend` em `origin/main` `46ccb12` / PR `#96` -> provedores Avenia e BlindPay).

### 8.1. Distinção em relação aos achados já registrados

- **Já documentado anteriormente (`SEC85-05` na [#85](TESTES_SEGURANCA_APLICACAO_APIS.md) e `P86-19` na [#86](PENTEST_CONTROLADO_E_ANALISE_DE_IMPACTO.md)):** O método `DocumentFormat.detect` em `BlindPayComplianceService.java` (linhas 682-718) valida apenas o prefixo inicial de *magic bytes* (`%PDF`, 8 bytes PNG e `FF D8 FF` JPEG), permitindo o encaminhamento de arquivos truncados ou sem validação estrutural completa para o gateway.
- **Já coberto na Task Bônus Rust (`PR #96`):** Sanitização estrutural e limites de memória para **PDF Injection** (`/JavaScript`, `/JS`, `/OpenAction`, `/Launch`, assinaturas poliglotas `<script`, `<html`, `<!doctype`, `<?php`, `PK\x03\x04`) em `security/media-sanitizer/src/pdf.rs` e **Decompression Bombs / Pixel Floods** em `security/media-sanitizer/src/image_bomb.rs`.

### 8.2. Novos achados identificados nesta revisão (`APP-10`–`APP-13`)

| ID | Severidade | Repositório / Arquivo | Achado & Condição no Código | Impacto | Recomendação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **APP-10** | **Média / Alta** | `backend` (`feat/bonus-rust-pdf-image-sanitizer` `903ba67`): `security/media-sanitizer/src/image_bomb.rs` L207, L310-334, L599-675; `BlindPayComplianceService.java` L128-158, L682-718 | O módulo `image_bomb.rs` valida CRCs, dimensões e bombas de descompressão, mas não aplica a detecção de scripts/poliglotas (`<script`, `<svg`, `<html`, `<!doctype`, `<?php`, `javascript:`) usada em `pdf.rs` sobre payloads textuais de imagem: `tEXt` cai no ramo `_ => {}` (L207); `zTXt` (`verify_png_ztxt`, L599-626) e `iTXt` (`verify_png_itxt`, L628-675) validam tamanho/ratio e descartam o texto; `COM` (`0xFE`) e `APP0`–`APP15` (`0xE0`–`0xEF`, EXIF/XMP) no JPEG checam apenas comprimento do segmento (L310-334). Além disso, `BlindPayComplianceService.uploadDocument` ainda não invoca `vstable-media-sanitizer`. | Arquivos poliglotas ou imagens com scripts embutidos em metadados (EXIF/XMP/PNG chunks) passam pela checagem de prefixo Java e, mesmo após integrar o sanitizador atual, não teriam seus metadados textuais inspecionados contra injeção de script. | Reaproveitar inspeção de assinaturas ativas/poliglotas nos chunks PNG (`tEXt`, `zTXt`, `iTXt`) e segmentos JPEG (`COM`, `APP0`–`APP15`) no `image_bomb.rs` (ou reencodar/strip metadados), e acoplar `vstable-media-sanitizer` em `BlindPayComplianceService.uploadDocument`. |
| **APP-11** | **Baixa / Média** | `frontend` (`origin/main` `4994656`): `src/features/onboarding/pages/Register/ComplianceStep.tsx` L80-87 | Na função `addFiles`, a extração da extensão sofreu substituição acidental do separador `'.'` pelo alias do diretório: `const ext = '@/features/onboarding/pages/Register' + (file.name.split('@/features/onboarding/pages/Register').pop()?.toLowerCase() ?? '')`. Assim, `ALLOWED_EXTENSIONS.includes(ext)` avalia sempre `false`, restando apenas `ALLOWED_MIME_TYPES.includes(file.type)`. | Bypass trivial no cliente apenas ajustando o MIME type informado pelo navegador; controle de extensão totalmente inoperante na UI de compliance Avenia. | Corrigir extração para `.split('.')` (`'.' + ...`) e manter a defesa efetiva no servidor/storage. |
| **APP-12** | **Média** *(Arquitetura)* | `frontend` (`origin/main` `4994656`): `src/features/onboarding/api/compliance.ts` L41-51 (`uploadFileToS3`) | No KYC da Avenia, o frontend obtém uma URL pré-assinada via `POST /v1/onboarding/{id}/compliance/documento` e envia o arquivo binário via `PUT` diretamente do navegador para o bucket S3 da Avenia, sem trafegar pelo backend V-Stable. | Qualquer validação de magic bytes, estrutura ou sanitização Rust implementada apenas no backend Java (`BlindPayComplianceService`) é contornada no fluxo de documentos da Avenia; além disso, um atacante autenticado pode enviar arquivos arbitrários para a URL pré-assinada diretamente fora do frontend. | Avaliar proxy de upload pelo backend V-Stable (quando compatível com limites do API Gateway/Lambda) ou validar/registrar explicitamente o risco residual e restrições de `Content-Type`/tamanho na política da URL pré-assinada da Avenia. |
| **APP-13** | **Média** | `backend` (`origin/main` `46ccb12`): `BlindPayKybRequest.java` L37, L49-56, L63, L108-114; `BlindPayComplianceService.java` L152-158, L407-414 | Os campos de documento em `BlindPayKybRequest` aceitam qualquer string compatível com `HTTPS_URL = "https://\\S+"` e `rfiValue` aceita qualquer `url.startsWith("https://")` (até 2048 chars), sem restringir o host ao domínio de storage retornado por `POST /v1/blindpay/compliance/documents` (`response.fileUrl()`). | Usuários autenticados podem submeter URLs externas arbitrárias (`https://host-externo/...`) nos cadastros KYB ou respostas de RFI encaminhados pelo backend à BlindPay (abuso de origem não confiável / links maliciosos armazenados no fluxo de compliance). | Restringir as URLs aceitas para documentos KYB/RFI por allowlist explícita de host do storage da BlindPay (ou vincular/validar contra os `fileUrl` emitidos pela própria sessão/empresa). |

### 8.3. Limitações desta auditoria complementar

- Revisão realizada exclusivamente por inspeção estática e leitura de código nos repositórios locais (`backend`, `frontend`) — sem execução de upload real contra S3/Avenia/BlindPay e sem testes dinâmicos de XSS armazenado ou exploração em buckets externos.
- Os achados `APP-10` a `APP-13` foram apenas **documentados nesta wiki** por decisão de escopo (sem alteração de código de aplicação nesta etapa), permanecendo abertos para tratamento na etapa de remediação (**Issue #87**).
