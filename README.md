#FLYWAY

### Parte 1: O que é o Flyway e como ele funciona?

O **Flyway** é uma ferramenta de código aberto para **versionamento de banco de dados** (ou *database migration*). A ideia principal é tratar as alterações no esquema do seu banco de dados (criar tabelas, adicionar colunas, etc.) da mesma forma que você trata seu código-fonte: com controle de versão.

Imagine o problema:
*   **Desenvolvedor A** adiciona uma coluna `email` na tabela `users`.
*   **Desenvolvedor B** adiciona uma tabela `products`.
*   Como garantir que o banco de dados de produção, de testes e de todos os outros desenvolvedores esteja sempre na mesma "versão" e que todas as alterações sejam aplicadas na ordem correta?

O Flyway resolve isso de forma elegante.

#### Como ele funciona (Os Conceitos-Chave)

1.  **Scripts de Migração (Migrations):**
    *   Você escreve as alterações do seu banco de dados em scripts SQL.
    *   Esses scripts seguem uma **convenção de nomenclatura rigorosa** que é a chave de tudo:
        `V<VERSÃO>__<DESCRIÇÃO>.sql`
        *   `V`: Indica que é uma migração versionada (Versioned).
        *   `<VERSÃO>`: Um número de versão, como `1`, `1.1`, `20231027103000`. As versões são ordenadas.
        *   `__` (dois underscores): Separam a versão da descrição.
        *   `<DESCRIÇÃO>`: Um texto descritivo sobre o que o script faz. Ex: `Create_user_table`.
    *   **Exemplos:**
        *   `V1__Create_user_table.sql`
        *   `V2__Add_email_to_user_table.sql`
        *   `V3__Create_products_table.sql`

2.  **A Tabela de Histórico (`flyway_schema_history`):**
    *   Na primeira vez que o Flyway roda em um banco de dados, ele cria uma tabela de metadados chamada `flyway_schema_history`.
    *   Esta tabela é o "cérebro" do Flyway. Ele armazena quais migrações já foram aplicadas, quando foram aplicadas, por quem e se a execução foi bem-sucedida.
    *   **Isso é fundamental:** Ao rodar novamente, o Flyway consulta essa tabela para saber quais scripts ele ainda precisa executar. Ele nunca executa um script que já está na tabela de histórico.

#### O Fluxo de Execução do Flyway

Quando você aciona o comando `migrate`:
1.  **Conexão:** O Flyway se conecta ao banco de dados.
2.  **Lock:** Ele aplica um "lock" (bloqueio) no banco para garantir que nenhuma outra instância do Flyway rode ao mesmo tempo, evitando conflitos.
3.  **Verificação:** Ele verifica se a tabela `flyway_schema_history` existe. Se não, ele a cria.
4.  **Scan:** Ele escaneia o classpath (ou o diretório que você configurar) em busca de scripts de migração (ex: `V1__...`, `V2__...`).
5.  **Comparação:** Ele compara a lista de scripts encontrados com os registros na tabela `flyway_schema_history`.
6.  **Migração:** Para cada script que ainda **não foi aplicado**, o Flyway o executa em ordem de versão (do menor para o maior).
7.  **Atualização:** Após cada execução bem-sucedida, ele insere um novo registro na `flyway_schema_history`. Se um script falhar, a migração inteira para, e a tabela de histórico não é atualizada para aquele script.
8.  **Unlock:** Ao final, ele remove o "lock" do banco.

---

### Parte 2: Usando o Flyway com Spring Boot e Maven

A integração com Spring Boot é incrivelmente simples, pois ele oferece auto-configuração para o Flyway.

#### Passo 1: Adicionar as Dependências no `pom.xml` (Maven)

Você só precisa de duas dependências principais no seu arquivo `pom.xml`:

1.  **Flyway Core:** A biblioteca principal do Flyway.
2.  **Spring Data (JPA ou JDBC):** O Flyway precisa de uma fonte de dados (`DataSource`) para se conectar ao banco. A dependência do Spring Data já provê isso.

```xml
<dependencies>
    <!-- Dependência para JPA e Hibernate (já inclui o JDBC) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Dependência do Flyway -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>

    <!-- Driver do seu banco de dados (ex: PostgreSQL) -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

#### Passo 2: Configurar o Banco de Dados no `application.properties`

Configure a conexão com seu banco de dados no arquivo `src/main/resources/application.properties` (ou `.yml`).

```properties
# Configurações do DataSource (conexão com o banco)
spring.datasource.url=jdbc:postgresql://localhost:5432/meu_banco
spring.datasource.username=postgres
spring.datasource.password=sua_senha

# --- Configurações IMPORTANTES para o Flyway e Hibernate ---

# Habilita o Flyway (já é true por padrão se a dependência estiver no classpath)
spring.flyway.enabled=true

# DICA CRÍTICA: Desligue o gerenciamento de schema do Hibernate/JPA!
# O Flyway agora é o responsável por criar e alterar tabelas.
# Se você deixar 'create' ou 'update', Hibernate e Flyway entrarão em conflito.
# 'validate' é uma boa opção: o Hibernate vai verificar se o schema do banco
# corresponde às entidades JPA, mas não tentará alterá-lo.
spring.jpa.hibernate.ddl-auto=validate
```
**Atenção:** A propriedade `spring.jpa.hibernate.ddl-auto` é a causa número 1 de problemas para iniciantes. Defini-la como `validate` ou `none` é essencial para que apenas o Flyway gerencie o banco.

#### Passo 3: Criar os Scripts de Migração SQL

Por padrão, o Spring Boot procura os scripts de migração na pasta `src/main/resources/db/migration`.

1.  Crie essa estrutura de pastas.
2.  Adicione seus scripts SQL lá, seguindo a convenção de nomenclatura.

**Exemplo:**

`src/main/resources/db/migration/V1__Create_author_table.sql`:
```sql
CREATE TABLE author (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

`src/main/resources/db/migration/V2__Create_book_table.sql`:
```sql
CREATE TABLE book (
    id BIGINT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    author_id BIGINT,
    FOREIGN KEY (author_id) REFERENCES author(id)
);
```

#### Como Tudo Funciona Junto

Quando você inicia sua aplicação Spring Boot:
1.  A auto-configuração do Spring detecta a dependência do `flyway-core`.
2.  **Antes** de o Hibernate/JPA ser inicializado, o Spring Boot configura e executa o Flyway.
3.  O Flyway executa o fluxo que descrevi na Parte 1: conecta-se, verifica a tabela de histórico e aplica as migrações pendentes (`V1` e `V2`, no nosso exemplo).
4.  Só depois que o banco de dados está na versão correta, o Hibernate/JPA é inicializado. Como você configurou `ddl-auto=validate`, ele apenas verificará se as entidades `@Entity` correspondem às tabelas `author` e `book`, que o Flyway acabou de criar.

Pronto! Sua aplicação sempre iniciará com o banco de dados na versão correta.

---

### Parte 3: Usando o Plugin do Maven do Flyway

Às vezes, você quer rodar as migrações **fora** do ciclo de vida da aplicação. Por exemplo, em um pipeline de CI/CD ou para que um DBA possa preparar o banco de produção antes do deploy da aplicação. Para isso, usamos o plugin do Maven.

#### Passo 1: Configurar o Plugin no `pom.xml`

Adicione a seção `<plugin>` dentro de `<build><plugins>` no seu `pom.xml`.

```xml
<build>
    <plugins>
        <!-- Outros plugins como o spring-boot-maven-plugin -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- Plugin do Flyway -->
        <plugin>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-maven-plugin</artifactId>
            <version>9.16.0</version> <!-- Use uma versão recente -->
            <configuration>
                <!-- Pega a URL, usuário e senha do application.properties -->
                <!-- ou você pode definir aqui diretamente -->
                <url>${spring.datasource.url}</url>
                <user>${spring.datasource.username}</user>
                <password>${spring.datasource.password}</password>
                <!-- Opcional: Local dos scripts (o padrão já é o correto) -->
                <locations>
                    <location>filesystem:src/main/resources/db/migration</location>
                </locations>
            </configuration>
        </plugin>
    </plugins>
</build>
```
*Dica: Para o plugin ler as propriedades do `application.properties`, você pode precisar de uma configuração adicional, mas para simplificar, o exemplo acima mostra como configurá-las diretamente.*

#### Passo 2: Executar Comandos do Flyway via Maven

Agora, você pode usar a linha de comando para gerenciar seu banco:

*   **Aplicar migrações:**
    ```sh
    mvn flyway:migrate
    ```

*   **Verificar o status das migrações (o que foi aplicado, o que está pendente):**
    ```sh
    mvn flyway:info
    ```

*   **Validar se os scripts no projeto correspondem ao que foi aplicado (checksum):**
    ```sh
    mvn flyway:validate
    ```

*   **Limpar o banco de dados (APAGA TUDO! Cuidado, útil apenas em ambiente de desenvolvimento):**
    ```sh
    mvn flyway:clean
    ```

### Resumo e Boas Práticas

1.  **Flyway Versiona seu Banco:** Trate seu schema como código.
2.  **Integração com Spring Boot é Automática:** Adicione a dependência, crie os scripts na pasta correta e configure o `ddl-auto=validate`.
3.  **Plugin do Maven para Controle Manual:** Use `mvn flyway:migrate` para controlar as migrações fora da inicialização da aplicação, ideal para automação e pipelines.
4.  **Regra de Ouro:** **Nunca, jamais, edite um script de migração que já foi aplicado** em qualquer ambiente. Se precisar corrigir algo, crie uma *nova* migração (ex: `V4__Fix_column_type_in_book_table.sql`). O Flyway valida a integridade dos scripts antigos através de um checksum.


# Mais informações sobre Flyway

### 1. Tipos Avançados de Migração (Além do `V`)

Você já conhece as migrações versionadas (`V...`), mas existem outras que resolvem problemas específicos:

#### a) Migrações Repetíveis (`R__...`.sql)

*   **O que são?** São migrações que rodam **toda vez que o seu conteúdo (checksum) muda**. Elas não têm um número de versão e são sempre aplicadas por último, após todas as migrações versionadas pendentes.
*   **Quando usar?** Perfeitas para gerenciar objetos do banco que não têm um "estado" versionado, como:
    *   **Views:** Se você altera a definição de uma `VIEW`, basta editar o arquivo SQL. Na próxima vez que o Flyway rodar, ele detectará a mudança e recriará a `VIEW`.
    *   **Stored Procedures, Functions, Triggers:** Mesma lógica. Em vez de criar `V2__alter_procedure`, `V3__alter_procedure_again`, você simplesmente edita um único arquivo `R__My_Stored_Procedure.sql`.
*   **Exemplo:** `R__Create_or_update_customer_summary_view.sql`

#### b) Migrações de "Desfazer" (Undo Migrations - `U__...`.sql)

*   **O que são?** São scripts SQL que revertem o efeito de uma migração versionada correspondente. O nome do arquivo deve corresponder à versão que ele desfaz.
*   **Exemplo:** Para reverter `V1__Create_user_table.sql`, você criaria `U1__Drop_user_table.sql`.
*   **Como usar?** Através do comando `flyway:undo` (no Maven).
*   **⚠️ Cuidado!** A comunidade e os próprios criadores do Flyway recomendam cautela com Undo migrations. A filosofia predominante é a de **"Roll Forward"**: se você cometeu um erro na `V2`, não desfaça a `V2`. Em vez disso, crie uma `V3__Fix_mistake_from_V2.sql`. Isso mantém um histórico linear e auditável de todas as alterações, exatamente como o Git. Migrações de "undo" podem ser complexas e arriscadas, especialmente se envolverem perda de dados.

***

### 2. Estratégias e Boas Práticas para Times

#### a) O Problema do Conflito de Versão

Imagine que você e um colega estão trabalhando em branches diferentes. Ambos criam uma migração com o mesmo número: `V3__My_feature.sql` e `V3__Colleagues_feature.sql`. Quando vocês tentarem fazer o merge, o Flyway vai falhar, pois não pode haver duas migrações com a mesma versão.

**A Solução: Nomenclatura Baseada em Timestamp**

A melhor prática para equipes é usar um timestamp como número de versão. Isso garante que as versões sejam sempre únicas e cronológicas.

*   **Formato:** `V<YYYYMMDDHHMMSS>__Descricao.sql`
*   **Exemplo:**
    *   `V20231027103000__Create_user_table.sql`
    *   `V20231027110000__Add_email_to_user_table.sql`

Isso resolve 99% dos problemas de merge de migrações.

***

### 3. Gerenciando um Banco de Dados Existente

O que fazer se você quer começar a usar o Flyway em um projeto que já tem um banco de dados em produção há anos? Você não pode simplesmente criar a `V1` e mandar rodar, pois ele tentaria criar tabelas que já existem.

**O Comando `baseline` é a Resposta.**

O `baseline` diz ao Flyway: "Ignore todas as migrações até esta versão e finja que elas já foram aplicadas".

*   **Fluxo de trabalho:**
    1.  Defina uma versão de linha de base. Por exemplo, `1.0`.
    2.  Execute o comando: `mvn flyway:baseline -Dflyway.baselineVersion=1.0`
    3.  Isso criará a tabela `flyway_schema_history` e inserirá uma primeira linha com a versão `1.0`, marcando-a como bem-sucedida.
    4.  Agora, o Flyway só aplicará migrações com versão **maior** que `1.0` (ex: `V1.1__...` ou `V2__...`).

Você pode configurar isso também no `application.properties`:

```properties
# Diz ao Flyway para criar um baseline na versão 1 na primeira execução
spring.flyway.baseline-on-migrate=true
spring.flyway.baseline-version=1
```

### 4. Configurações Avançadas no Spring Boot (`application.properties`)

#### a) Múltiplos Locais para Scripts (`locations`)

Por padrão, o Flyway procura em `classpath:db/migration`. Você pode mudar ou adicionar locais, o que é útil para organizar scripts por módulo ou tipo.

```properties
# Procura scripts em duas pastas diferentes
spring.flyway.locations=classpath:db/migration,classpath:db/procedures
```

#### b) Placeholders

Isso é extremamente útil para tornar seus scripts SQL dinâmicos e adaptáveis a diferentes ambientes (desenvolvimento, homologação, produção).

*   **No seu SQL:**
    `V5__Create_app_user.sql`:
    ```sql
    CREATE USER ${appName}_user WITH PASSWORD '${appName}_password';
    GRANT ALL PRIVILEGES ON DATABASE my_db TO ${appName}_user;
    ```

*   **No seu `application.properties`:**
    ```properties
    spring.flyway.placeholders.appName=minha_app
    spring.flyway.placeholders.appName_password=senha_secreta_aqui
    ```
    O Flyway substituirá as variáveis `${...}` antes de executar o SQL.

#### c) Callbacks

O Flyway permite que você execute scripts SQL ou código Java em pontos específicos do ciclo de vida da migração (ex: `beforeMigrate`, `afterEachMigrate`, `afterMigrate`).

*   **Uso com SQL:** Crie um arquivo SQL com um nome específico no seu diretório de migrações.
    *   `beforeMigrate.sql`: Roda uma vez antes de todo o processo de migração.
    *   `afterMigrate.sql`: Roda uma vez depois de todo o processo.
    *   `beforeEachMigrate.sql`: Roda antes de cada migração individual.
*   **Quando usar?** Ótimo para tarefas como desabilitar triggers antes de migrar, limpar caches ou popular dados de teste em ambientes de desenvolvimento.

### 5. Comandos Úteis (e Perigosos)

*   `mvn flyway:repair`
    *   **O que faz?** Reconstroi a tabela `flyway_schema_history`. É útil se uma migração falhou e deixou a tabela de metadados em um estado inconsistente. Ele também realinha os checksums caso você tenha (erroneamente) editado um script já aplicado e precise "consertar" o histórico. **Use com extremo cuidado.**

*   `mvn flyway:clean`
    *   **O que faz?** **APAGA TODOS OS OBJETOS** (tabelas, views, etc.) no schema configurado.
    *   **⚠️ Perigo!** Este comando é destrutivo. É fantástico para recriar um banco de dados limpo do zero em seu ambiente de desenvolvimento local, mas **NUNCA** deve ser habilitado ou executado em ambientes de produção. O Spring Boot, por segurança, desabilita este comando por padrão.

### Tabela Resumo dos Prefixos de Migração

| Prefixo | Nome | Descrição |
| :--- | :--- | :--- |
| **V** | Versioned | Migração padrão, baseada em versão, executa apenas uma vez. |
| **U** | Undo | Desfaz uma migração versionada. Usado com `flyway:undo`. |
| **R** | Repeatable | Executa toda vez que seu conteúdo (checksum) muda. |
| **B** | Baseline | Usado para introduzir o Flyway em um DB existente. |

# Exemplo de uso do UNDO


### ⚠️ Aviso Importante: A Filosofia "Roll-Forward" vs. "Undo"

A **prática mais comum e recomendada** no mundo do versionamento de banco de dados é o **"Roll-Forward"** (avançar).

*   **Problema:** Você aplicou a migração `V2__Add_email_column.sql`, mas percebeu que o tipo de dado estava errado (era `VARCHAR(100)` em vez de `VARCHAR(255)`).
*   **Solução Roll-Forward (Recomendada):** Crie uma nova migração, `V3__Alter_email_column_type.sql`. Isso mantém um histórico claro e auditável de todas as mudanças que ocorreram no banco.
*   **Solução Undo (Menos Comum):** Execute o `undo` para reverter a `V2` e depois corrija o script `V2` e aplique-o novamente. Isso "apaga" o erro do histórico, o que pode ser problemático para rastreabilidade e em ambientes de produção.

O `undo` é mais útil em **ambientes de desenvolvimento local** para correções rápidas, ou em cenários de **rollback de emergência** muito bem planejados em produção.

---

### Como Usar o `Undo` - Passo a Passo

O conceito é simples: para cada migração versionada (`V...`), você pode criar um script `undo` (`U...`) correspondente que reverte completamente suas alterações.

#### Pré-requisito: Habilitar o Comando

Por segurança, o `undo` pode estar desabilitado. Você precisa garantir que ele esteja ativo.

No `pom.xml` (para o plugin Maven):
```xml
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    ...
    <configuration>
        ...
        <!-- Pode ser necessário em algumas configurações/versões -->
        <!-- Por padrão, o undoFromMigration já é habilitado nas versões mais recentes -->
        <undoFromMigration>true</undoFromMigration> 
    </configuration>
</plugin>
```

No `application.properties` (para o Spring Boot):
```properties
# Habilita o comando undo para ser usado via API/código (não via linha de comando)
# Por padrão, já é habilitado, mas é bom saber que existe
# spring.flyway.undo-on-error=false # Não é o que queremos aqui, isso é outra coisa
```
Na prática, o mais comum é usar o `undo` via plugin do Maven, então a configuração no `pom.xml` é mais relevante.

### Exemplo 1: Simples (Criar e Remover Tabela)

Vamos supor que sua estrutura de arquivos é:
`src/main/resources/db/migration/`

**Passo 1: Criar a migração versionada (`V`)**

Crie o arquivo `V1__Create_client_table.sql`:
```sql
CREATE TABLE client (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Passo 2: Criar a migração de `undo` (`U`)**

Crie o arquivo `U1__Create_client_table.sql`. Note que **a versão e a descrição devem ser idênticas** ao arquivo `V`, apenas o prefixo muda.

```sql
DROP TABLE client;
```

**Passo 3: Executar a Migração**

Na linha de comando, aplique a migração `V1`:
```sh
mvn flyway:migrate
```

*   **O que acontece?**
    *   A tabela `client` é criada no banco.
    *   Uma linha é adicionada em `flyway_schema_history` para a versão `1`.

**Passo 4: Executar o `Undo`**

Agora, vamos desfazer a última migração aplicada.
```sh
mvn flyway:undo
```

*   **O que acontece?**
    *   O Flyway olha para a última migração na tabela `flyway_schema_history` (que é a `V1`).
    *   Ele procura pelo script `U1__Create_client_table.sql`.
    *   Ele executa o script `U1`, que executa o `DROP TABLE client;`.
    *   Ele **remove a linha correspondente à versão `1`** da tabela `flyway_schema_history`.

Agora seu banco de dados está exatamente no estado em que estava antes da `V1` ser aplicada.

### Exemplo 2: Mais Complexo (Alterar Tabela e Perda de Dados)

Este exemplo ilustra por que o `undo` pode ser perigoso.

**Passo 1: Scripts `V` e `U`**

`V2__Add_phone_to_client.sql`:
```sql
ALTER TABLE client ADD COLUMN phone_number VARCHAR(20);

-- Vamos popular o dado para alguns clientes existentes
UPDATE client SET phone_number = '99999-0001' WHERE id = 1;
UPDATE client SET phone_number = '99999-0002' WHERE id = 2;
```

`U2__Add_phone_to_client.sql`:
```sql
ALTER TABLE client DROP COLUMN phone_number;
```

**Passo 2: Migrar e Desfazer**

1.  Execute `mvn flyway:migrate`.
    *   A coluna `phone_number` é adicionada e os dados são populados.
    *   Uma linha para a versão `2` é adicionada ao histórico.

2.  Execute `mvn flyway:undo`.
    *   O Flyway executa o script `U2`.
    *   A coluna `phone_number` é **removida**.
    *   **Todos os números de telefone que foram adicionados são permanentemente perdidos!**
    *   A linha da versão `2` é removida do histórico.

Este é o ponto crítico: **operações de `undo` podem e frequentemente causam perda de dados**. Por isso, a abordagem "Roll-Forward" é mais segura, pois permite planejar a migração de dados de forma mais controlada.

### Comandos `Undo` via Maven

*   `mvn flyway:undo`
    *   Desfaz a **última** migração aplicada.

*   `mvn flyway:undo -Dflyway.target=<versão>`
    *   Desfaz todas as migrações até atingir a versão alvo. Por exemplo, se você está na `V5` e executa com `target=2`, ele irá desfazer a `V5`, `V4` e `V3`.

### Resumo: Quando e Como Usar `Undo`

| Cenário | Recomendação | Exemplo de Comando |
| :--- | :--- | :--- |
| **Desenvolvimento Local** | **Aceitável.** Útil para testar e corrigir rapidamente um script de migração que você acabou de escrever, sem ter que limpar todo o banco. | `mvn flyway:undo` |
| **Ambiente de Testes/Homologação** | **Use com Cautela.** Pode ser usado para reverter uma feature que causou problemas, mas a equipe precisa estar ciente. Roll-forward ainda é preferível. | `mvn flyway:undo` |
| **Ambiente de Produção** | **Altamente Desaconselhado / Apenas para Emergências.** Use somente como parte de um plano de rollback bem definido e testado. A perda de dados é um risco real. | `mvn flyway:undo` |

Em 95% dos casos, a melhor resposta para um erro em uma migração já aplicada é criar uma nova migração que corrija o problema.
