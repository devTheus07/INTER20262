# HardSoft — Sistema de Ordem de Serviço

Sistema de gestão de ordens de serviço para a assistência técnica **Matheus Técnico de Informática**
(manutenção de computadores e notebooks).

Projeto acadêmico — FATEC São José do Rio Preto.
Grupo **HardSoft**: Matheus Vilela, Davi Vieira e Isabelle Miraglia.

## Arquitetura: o banco de dados é independente do Java

Toda a regra de negócio vive no **SQL Server**, escrita no mesmo padrão dos scripts das aulas
(`create table`, views com `join`, procedures com `begin try`/`begin tran`, triggers com
`inserted`/`deleted` e functions escalares e de tabela). O Java (Spring Boot) não cria tabelas
(`spring.jpa.hibernate.ddl-auto=none`) e não tem regra de negócio: ele apenas **chama o banco**.

| Operação no sistema | Quem faz |
|---|---|
| Criar o banco e as tabelas | Script `banco/01_HardSoft_Tabelas.sql` |
| Gravar (cadastrar, editar, excluir) | **Stored procedures** (`sp_CadCliente`, `sp_AbrirOS`, `sp_UsarPecaServico`...) |
| Ler (listas, detalhes, histórico) | **Views** (`v_Cliente`, `v_OS`, `v_PecaEstoque`...) e **functions** (`fc_OSsPorCliente`, `fc_PecasEstoqueMinimo`) |
| Baixa/devolução de estoque, valor total da OS, data de saída, trava de status, soft delete, contato principal, logs | **Triggers** (`tg_BaixaEstoque`, `tg_DevolveEstoque`, `tg_CalcTotalOS_*`, `tg_StatusOS`, `tg_ExcluiCliente`...) |

No Java, a estrutura continua a mesma (controllers → services → repositories):

```java
// LEITURA: os repositories consultam views e functions (consultas nativas)
@Query(value = COLUNAS_OS + "FROM v_OS WHERE Cod_OS = :id", nativeQuery = true)
Optional<OrdemServicoView> buscarView(@Param("id") Long id);

@Query(value = COLUNAS_OS + "FROM dbo.fc_OSsPorCliente(:idCliente) ORDER BY Cod_OS", nativeQuery = true)
List<OrdemServicoView> listarPorClienteFunction(@Param("idCliente") Long idCliente);

// GRAVAÇÃO: o ProcedureRepository executa a procedure (EXEC sp_AbrirOS @idCliente = ?, ...)
Long id = procedures.executarRetornandoId("sp_AbrirOS", "idCliente", dto.getIdCliente(), ...);
```

As interfaces da pasta `views/` representam uma linha de cada view, e os DTOs têm um
método `fromView(...)` ao lado do `fromEntity(...)`.

Quando uma regra é violada, a procedure dispara `throw 50002` (regra de negócio) ou
`throw 50001` (registro não encontrado); no `begin catch` ela desfaz a transação (`rollback`)
e devolve o erro, e o Java mostra a mensagem do banco na tela (HTTP 422 ou 404).

O cadastro de cliente é feito em partes: `sp_CadCliente` grava o cliente e, em seguida, o Java
chama `sp_CadEmailCliente` e `sp_CadTelefoneCliente` para cada contato da lista.

## Stack

- **Banco:** SQL Server 2012 ou superior (usa `THROW`)
- **Backend:** Java 17, Spring Boot 3.5, Spring Data JPA (consultas nativas), Spring JDBC (procedures), Validation, Lombok, Maven
- **Frontend:** HTML5, CSS3 e JavaScript puro, servido pelo próprio Spring Boot

## Como rodar

### 1. Criar o banco (SQL Server Management Studio)

Abra e execute os scripts da pasta `banco/`, **nesta ordem**:

```
01_HardSoft_Tabelas.sql      cria o banco HardSoftDB e as tabelas
02_HardSoft_Views.sql        views (todas as leituras)
03_HardSoft_Functions.sql    functions
04_HardSoft_Procedures.sql   procedures (todas as gravações)
05_HardSoft_Triggers.sql     triggers (estoque, valor da OS, status, soft delete, logs)
06_HardSoft_DadosTeste.sql   dados de teste (opcional)
```

> Execute-os em um servidor onde o banco **HardSoftDB ainda não existe**.

### 2. Configurar a conexão

Em `src/main/resources/application.properties`, ajuste servidor, usuário e senha do SQL Server.

### 3. Subir a aplicação

```bash
mvnw.cmd spring-boot:run      # Windows
./mvnw spring-boot:run        # Linux/Mac
```

Acesse `http://localhost:8080`. Logins de teste (criados pelo script 06):

- `matheus@hardsoft.com` / `senha123`
- `ana@hardsoft.com` / `senha456`

## Regras de negócio (no banco)

- Cliente precisa de pelo menos 1 e-mail e 1 telefone; o banco garante exatamente 1 principal de cada
  (os triggers `tg_EmailPrincipal` e `tg_TelefonePrincipal`).
- CPF com 11 dígitos ou CNPJ com 14 dígitos, sem repetir.
- Excluir cliente = **soft delete** (fica INATIVO). Bloqueado se ele tiver OS concluída ou entregue.
- O setup (equipamento) da OS precisa pertencer ao cliente da OS.
- Não é possível usar mais peças do que há em estoque; a baixa e a devolução são automáticas.
- Valor total da OS = mão de obra + peças, recalculado automaticamente.
- OS concluída, entregue ou cancelada não muda mais de status; cancelar devolve as peças ao estoque.
- Peças, setups e tipos de serviço não podem ser excluídos (preservam o histórico).

## Estrutura do projeto

```
sistema/
├── banco/                     ← scripts SQL (01 a 06)
├── pom.xml
└── src/main/
    ├── java/com/hardsoft/sistema/
    │   ├── controllers/       ← endpoints REST
    │   ├── services/          ← regras de uso: leem pelos repositories, gravam pelas procedures
    │   ├── repositories/      ← consultas nas views/functions + ProcedureRepository (procedures)
    │   ├── views/             ← formato das linhas das views
    │   ├── entities/          ← mapeamento das tabelas (modelo de classes)
    │   ├── dtos/              ← dados de entrada e saída da API
    │   └── exceptions/        ← tradução dos erros do banco para HTTP
    └── resources/
        ├── application.properties
        └── static/            ← frontend (login, dashboard e telas)
```

## Endpoints da API

- `/api/auth/login`
- `/api/admins`
- `/api/clientes` (+ `/inativos` e `PATCH /{id}/reativar`)
- `/api/setups` (+ `/cliente/{id}`)
- `/api/pecas` (+ `?nome=` e `/estoque-baixo?limite=`)
- `/api/tipos-servico`
- `/api/ordens-servico` (+ `?status=`, `/cliente/{id}` e `PATCH /{id}/status`)
- `/api/servicos` (+ `/ordem-servico/{id}`)
- `/api/pecas-servico` (+ `/servico/{id}`)

## Autenticação

Versão acadêmica: a procedure `sp_LoginAdmin` confere e-mail e senha (texto puro) e o navegador
guarda a sessão em `sessionStorage`. Em produção, o recomendado seria Spring Security com senha
criptografada (BCrypt).
