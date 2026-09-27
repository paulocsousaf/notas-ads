## Banco de Dados

- O banco usa snak_case e o código usa CamelCase
- mapeamento entre modelo de dados e modelo de resposta.

## Persistência no Projeto

- src/server.ts  **→**  src/database.ts  **→**  prontuario.db (arquivo físico no disco)
- src/database.ts --> *Conexão com o banco*

## SQLite

É importante entender a diferença do SQLite para outros bancos:

- O SQLite é uma **biblioteca embutida** (_embedded_), não um servidor.
- Ele abre (ou cria) o arquivo `prontuario.db` e lê/escreve **direto nele**.

|            | Postgres/MySQL                                                | SQLite                                    |
| ---------- | ------------------------------------------------------------- | ----------------------------------------- |
| instalação | Instala servidor, cria porta, configura usuário               | Já vem tudo embutido na biblioteca        |
| Servidor   | Roda em um processo separado (daemon), escutando em uma porta | Não tem servidor. Acessa o arquivo direto |
| Conexão    | String com host, porta, usuário, senha, database              | Caminho do arquivo                        |


## `better-sqlite3`

- Nesse momento não usaremos ORM
- `better-sqlite3` você passa o SQL e manda ela executar
- Biblioteca síncrona, devolve o dado direto. Sem promises, sem await e sem async
- A escolha de usar uma biblioteca síncrona é para diminuir o ruido de entendimento
- Lembrando que o SQLite é um Banco local, por isso as operações serem síncronas não faz mal
```js
db.pragma('forengn_key = ON')
```
### Quatro métodos

- prepare() -> O banco analisa o comando e guarda o plano
#### all()
```js
// all: retorna um array de linhas
const rows = db.prepare('SELECT id, name FROM patients ORDER BY name').all()
```

#### get()
```js
// Para um registro em específico
// Devolve UMA linha ou undefined
const row = db.prepare('SELECT * FROM patients WHERE id = ?').get(42)
```

#### run()

```js
// Para INSET, UPDATE, DELETE
const result = db
	.preapre('INSERT INTO patients (name) VALUES(?)')
	.run('Joana');
result.lastInsertRowid;
```
### Veja como a biblioteca funciona
db -->  prepare -->  SQL -->  Método (all, get ou run)

```js
	db.prepare("SELECT * FROM patients").all()
```

| método | retorno                |
| ------ | ---------------------- |
| .all() | Array de Objetos       |
| .get() | Um objeto ou undefined |
| .run() | info da operação       |
|        |                        |
#### Listar todos os pacientes
```js
const rows = db
	.prepare('SELECT * FROM patients')
	.all()
	
// [{id: 1, name: 'Ana'}....]
```

#### Buscar um pacientes

```js
const row = db.prepare('SELECT * FROM patients WHERE id = ?').ge(10)
// ? é um placeholder, o valor real vem como argumento de get
```
#### Modificar ou Inserir
```js
const result = db.prepare(
  "INSERT INTO patients (name, birth_date, national_id) VALUES (?, ?, ?)"
).run("João Silva", "1990-05-15", "700012345678999");

console.log(result.lastInsertRowid); // → 9  (o id gerado pelo AUTOINCREMENT)
console.log(result.changes);         // → 1  (quantas linhas foram afetadas)
```
- `.run()` retorna `{ lastInsertRowid, changes }` — use para confirmar o que aconteceu