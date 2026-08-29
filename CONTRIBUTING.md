# Fluxo de Desenvolvimento

Este documento descreve o **fluxo completo de desenvolvimento** adotado nos repositórios de **front-end** e **back-end** do projeto.
Todo o processo utiliza **reusable workflows** do GitHub Actions, pre-commit, pre-push, linting, testes e boas práticas de desenvolvimento.

Para garantir que sua contribuição seja aprovada rapidamente, siga os seguintes passos:

## 1. Selecionar Task

- Acesse o [**board do projeto**](https://github.com/orgs/AGES-V-Stable/projects/4) no GitHub.
- Escolha uma tarefa disponível na coluna **To Do**.
- Clique em **Assign** e atribua a si mesmo (e a quem mais estiver fazendo a task com você).
- Arraste a task para a coluna **In Progress**.

> ⚠️ **IMPORTANTE:** Apenas atribua uma tarefa ao seu nome e a coloque em progresso quando de fato tiver disponibilidade para trabalhar nela.

## 2. Criar Feature-Branch

Você pode criar a sua feature-branch de duas maneiras, mas ela **sempre** deve seguir a seguinte convenção de nomenclatura (em **INGLÊS**):

```bash
<tipo>/<issue-id>-<descricao-em-kebab-case>
```

**Tipos de branch:**
- `feat`: nova funcionalidade
- `fix`: correção de bug
- `docs`: documentação
- `style`: formatação
- `refactor`: refatoração
- `test`: testes
- `chore`: manutenção

**Exemplo:**
`feat/123-login-screen`

### Opção A: Pela interface do GitHub (Recomendado)
No card da sua Issue, no campo `Development` (na barra lateral direita), clique em `Create a branch`. Isso é recomendado pois vincula automaticamente a sua branch à Issue no board.

### Opção B: Pelo Terminal (Localmente)
Se preferir criar via linha de comando, garanta que sua base está atualizada antes de criar a nova ramificação:
```bash
git checkout main                   # Garante que está na branch main
git pull origin main                # Atualiza sua branch main local
git checkout -b <nome-da-branch>    # Cria a branch e troca para ela
```

## 3. Acessar e Sincronizar o Repositório

### Baixando a Branch (Caso tenha usado a Opção A)
Como você criou a branch pela interface do GitHub, o seu computador local ainda não sabe que ela existe. Para baixar e acessar sua branch, execute:

```bash
git fetch origin                      # Baixa as informações mais recentes do GitHub
git checkout <nome-da-sua-branch>     # Entra na branch da sua tarefa
```
> *Se você usou a Opção B, você já está na branch e pode pular este passo).*

### Sincronizando o Repositório

Como o nosso fluxo é rápido, a branch `main` receberá novos códigos de outros colegas frequentemente. Isso fará com que a sua branch fique desatualizada e pode gerar quebras na pipeline. Portanto, **SEMPRE** antes de testar seu código e abrir um PR, atualize sua branch puxando as novidades da main com o comando **rebase** (que mantém nosso histórico limpo e linear):

```bash
git fetch origin main           # Busca as últimas atualizações da main remota
git rebase origin/main          # Coloca o seu código "no topo" da main mais recente
```

> ℹ️ **DICA:** Se ocorrerem conflitos durante o rebase, o terminal avisará. Basta abrir o VS Code, aceitar as mudanças corretas, rodar `git add .` e, em seguida, continuar o processo com `git rebase --continue`. **NUNCA** use `git commit` no meio de um rebase!

## 4. Configurar Ambiente Local

Com a branch sincronizada, suba o ambiente para desenvolver e testar sua feature localmente.

### Back-end (Spring Boot + PostgreSQL)

O banco de dados roda em um container **Docker**, enquanto a aplicação **Spring Boot** é executada localmente:

```bash
docker compose up -d      # Sobe o container do PostgreSQL (conforme o docker-compose.yml do projeto)
```

Em seguida, execute a aplicação Spring Boot pela sua IDE (Run/Debug) ou pelo wrapper de build do projeto (Maven ou Gradle), conforme configurado no repositório.

### Front-end (React)

```bash
npm install       # Instala as dependências
npm run dev        # Inicia o projeto em modo de desenvolvimento
```

## 5. Desenvolvimento

- Implemente POR COMPLETO a funcionalidade descrita na task
- Siga as boas práticas de código do projeto (clean code, SOLID)
- Mantenha o código organizado e legível

## 6. Verificações: Lint, Testes e Build

**Obrigatório:** toda feature deve ter testes desenvolvidos (unitários e/ou de integração), de acordo com o que for aplicável ao módulo alterado.

⚠️ Não serão aceitos PRs sem linting, formatação e testes bem-sucedidos (correspondentes aos módulos desenvolvidos).

**Front-end (React):**
```bash
npm run lint       # Verifica o código
npm run build      # Garante que o projeto builda sem erros
```

**Back-end (Spring Boot):**
- Execute o lint/formatação e os testes unitários e/ou de integração conforme configurado no projeto (Maven ou Gradle).
- Garanta que a aplicação builda sem erros antes de abrir o PR.

**Requisitos mínimos:**
- ✅ Todos os testes devem passar
- ✅ Sempre que possível, inclua evidências dos testes executados (e da cobertura, quando aplicável)

## 7. Commit
Use o padrão de **Conventional Commits:**

```text
<tipo>[escopo opcional]: <descricao curta>
```

**Exemplo:**
```text
feat: add login screen
fix: adjust e-mail validation
```

## 8. Push

```bash
git push origin <nome-da-branch>
```
O push passa automaticamente pelo **pre-push** (validações adicionais).

## 9. Verificar CI (GitHub Actions)

1. Acesse a aba **Actions** do repositório
2. Confirme que os workflows executaram com sucesso. O CI automatizado engloba:
   - **Validação e Lint:** Verifica pull request rules e estilos de código (ex: `npm run lint` no front-end, ou o padrão de formatação configurado no back-end).
   - **Testes (Unitários/Integração):** Garante a execução completa sem quebrar regras.
   - **Build:** Compila as aplicações (ex: `npm run build` no front-end e o build do Spring Boot no back-end), garantindo um artefato viável.

## 10. Abrir Pull Request (PR)

- Crie um PR da sua branch para main (Trunk-Based Development)
- Título claro e descritivo
- Descrição do que foi feito (use bullet points se houver muitas alterações)
- Inclua **evidências obrigatórias**:

| Tipo de repositório | Evidências obrigatórias |
|:---|:---|
| **Front-end** | 📸 Prints da tela/feature (pode necessitar de logs dependendo da task) |
| **Back-end** | 📄 Logs ou outputs da API |
| **Todos** | ✅ Evidência de que os testes passaram (screenshot/relatório, com cobertura quando aplicável) |

- Após criar o PR, mova o card no board para **Ready for Review**

## 11. Solicitar Review

- Atribua um revisor (qualquer AGES III)
- Avise no Discord, no canal `#pull-requests` marcando os AGES III (@AGESIII)

## 12. Code Review
Os revisores (AGES III) irão analisar o PR.

### ✅ Aprovado

- O AGES III move o card para **Done**
- O AGES III realiza o merge na `main`

### ❌ Reprovado

- Comentários serão deixados diretamente no PR
- O card volta para **In Progress**
- Você será notificado no Discord (canal `#review`)

**Se reprovado:**
- Corrija **todos** os pontos levantados
- Repita o fluxo a partir do passo 6 (Verificações: Lint, Testes e Build) até a aprovação (é chato, eu sei 😓)

## 13. Pós-Merge (Integração e Deploy Contínuo)

Após o merge na `main`, a infraestrutura de Actions cuida do processo de adoção dessa nova versão com workflows em background:
- **GitLab Sync:** O código mais atualizado do GitHub é sincronizado e espelhado automaticamente para os repositórios da AGES (GitLab).
- **Tag & Release:** Se aplicável, é gerada via automação uma Tag e os devidos Release Notes para a versão concluída.
- **Deploy:** O workflow envia as adições para implantação em seu respectivo ambiente como **staging** ou **production**.

## Pronto! 🥳
Seguindo este documento, sua contribuição será integrada com qualidade, rapidez e sem surpresas.

> ***P.S.:** Qualquer dúvida, favor acionar os AGES III 😊*
