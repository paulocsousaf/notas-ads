# Anotações de Estudo: Rota DELETE, Boas Práticas REST, Status Codes e Arquitetura em Camadas

Este documento consolida os conceitos arquiteturais, decisões de design, padrões de status HTTP e práticas com TypeScript abordados durante a concepção, análise crítica e refinamento da rota de exclusão/suspensão de prescrições (**DELETE `/api/medications/:id`**) no projeto **Painel de Medicação**.

---

## 1. Por que um Controller NUNCA deve chamar outro Controller?

Durante o desenvolvimento inicial, surgiu a dúvida: *seria uma boa ideia chamar `getById(req, res)` dentro da função `suspend(req, res)` para verificar se o ID existe antes de deletar?*

A resposta técnica e arquitetural é **não**. Existem três motivos fundamentais:

```
[ Requisição DELETE ]
         │
         ▼
┌──────────────────┐
│ suspend(req, res)│
└────────┬─────────┘
         │ Chama getById(req, res)? ──> ❌ ERRO: Envia res.status(200).json(...)
         │                                       Fecha a conexão HTTP prematuramente!
         ▼
res.status(204).send() ──> 💥 Crash: ERR_HTTP_HEADERS_SENT
```

### A. Conflito no Ciclo de Resposta HTTP (`ERR_HTTP_HEADERS_SENT`)
* Controladores do Express operam diretamente no objeto `res` (Response).
* A função `getById` finaliza o ciclo HTTP enviando uma resposta ao cliente (`res.status(200).json(...)`, `res.status(400)...` ou `res.status(404)...`).
* Se a função `suspend` chamar `getById` e depois tentar enviar `res.status(204).send()`, o Node.js lançará a exceção:  
  `Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client`.
* O protocolo HTTP não permite responder duas vezes para a mesma requisição.

### B. Retorno de Valor em JS vs. Envio de Resposta HTTP
* `getById(req, res)` manipula o fluxo de rede via `res`, mas não tem uma declaração `return item` em JavaScript.
* Chamar `getById` não entrega o objeto do banco para a variável local do chamador, tornando variáveis como `if (!existing)` indefinidas (`ReferenceError`).

### C. Violação do Padrão em Camadas
* Em uma arquitetura em camadas, a comunicação flui **verticalmente**:
  $$\text{Route} \longrightarrow \text{Controller} \longrightarrow \text{Service} \longrightarrow \text{Database}$$
* Controllers são **adaptadores de transporte** (porteiros). Eles não devem depender de outros controllers lateralmente. Toda consulta ou regra de dados deve ser solicitada à camada de **Service**.

---

## 2. Otimização de Persistência no DELETE: Evitando Consultas Redundantes

### O Antipadrão do "SELECT antes do DELETE"
Um vício comum é buscar o registro no banco antes de apagá-lo:

```typescript
// ⚠️ Desnecessário quando o driver já informa o resultado da ação
const existing = db.prepare(`SELECT id FROM medication_orders WHERE id = ?`).get(id);
if (!existing) {
  throw new Error("Prescrição não encontrada");
}
db.prepare(`DELETE FROM medication_orders WHERE id = ?`).run(id);
```

### A Abordagem Otimizada com `result.changes`
Drivers modernos de banco de dados (como o `better-sqlite3`) retornam metadados imediatos após uma operação de escrita:
* `result.changes`: quantidade de linhas afetadas pela instrução.

Se executarmos o `DELETE` diretamente:
* Se `result.changes > 0`: o registro existia e foi removido.
* Se `result.changes === 0`: o registro não existia.

```typescript
// ✅ Solução limpa, atômica e com metade dos acessos ao banco de dados:
function deleteMedication(id: number): boolean {
  const result = db.prepare(`DELETE FROM medication_orders WHERE id = ?`).run(id);
  return result.changes > 0;
}
```

---

## 3. Tratamento Semântico de Erros: 404 (Not Found) vs. 500 (Internal Server Error)

Uma API profissional deve separar claramente **erros de negócio previsíveis** de **falhas imprevistas de infraestrutura**.

```
                        Operação DELETE
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
       ID não encontrado               Falha no Banco de Dados
    (Fluxo previsto / Esperado)        (Disco cheio, SQLite travado)
               │                               │
               ▼                               ▼
         Status HTTP 404                Status HTTP 500
      "Prescrição não encontrada"        "Erro interno do servidor"
```

### Por que não lançar `throw new Error()` para 404?
1. **Controle de fluxo por exceção é um antipadrão:** Tentar deletar um ID que não existe não é um colapso do sistema; é uma busca sem resultado. O Service deve expressar isso com um retorno booleano (`false`) ou `null`.
2. **Confusão de Status:** Se o Service lança um `Error` genérico e o Controller o captura num `catch` geral respondendo `500`, a API informa que o servidor quebrou, quando na verdade o cliente apenas passou um ID inexistente.

### Prevenindo Requisições Penduradas (*Hang / Timeout*)
No bloco `catch`, evite condições que impeçam o envio da resposta:

```typescript
// ❌ Cuidado: Se o erro não for instância de Error, o cliente fica esperando até o timeout!
} catch (error) {
  if (error instanceof Error) {
    res.status(500).json({ error: error.message });
  }
}

// ✅ Resposta garantida com logging interno e mensagem segura ao cliente:
} catch (error) {
  console.error("Erro inesperado no servidor:", error);
  return res.status(500).json({ error: "Erro interno do servidor" });
}
```

> **Princípio de Segurança da Informação:** Erros 500 devem logar a stack trace completa internamente (`console.error`), mas devolver uma mensagem neutra ao cliente (`"Erro interno do servidor"`), sem vazar detalhes da infraestrutura ou queries SQL.

---

## 4. Boas Práticas REST: Devolver ou não conteúdo no DELETE?

### O controller não retornar nada no corpo é uma boa prática?
**Sim, é a convenção mais recomendada pela RFC 9110 (HTTP Semantics).**

Se a entidade foi excluída do banco de dados, ela não existe mais. Logo, não há necessidade nem coerência semântica em representá-la na resposta. O próprio status code já atesta o sucesso da ação.

### Tabela de Status Codes no Método DELETE

| Status Code | Significado | Quando utilizar? |
| :--- | :--- | :--- |
| **`204 No Content`** | Sucesso (Sem corpo) | **Padrão ouro:** O recurso foi excluído e nenhum dado precisa ser trafegado de volta na rede. |
| **`200 OK`** | Sucesso (Com corpo) | Quando o cliente precisa de dados de volta (ex.: retornar o objeto deletado para permitir um recurso de **"Desfazer" / Undo** no frontend). |
| **`202 Accepted`** | Aceito / Em processamento | Deleções assíncronas (ex.: solicitação enviada para uma fila de processamento em segundo plano). |
| **`400 Bad Request`** | Parâmetro inválido | O ID fornecido não é válido (ex.: texto onde se esperava número positivo). |
| **`404 Not Found`** | Recurso não encontrado | O ID solicitado para exclusão não existe no banco. |
| **`409 Conflict`** | Conflito de integridade | O item não pode ser excluído porque possui vínculos ativos (ex.: chave estrangeira). |
| **`500 Internal Error`** | Erro de infraestrutura | Falha inesperada no banco ou na execução do código. |

---

## 5. Tipagem Explícita no Service com TypeScript

Explicitar os tipos de retorno nas funções da camada Service é fundamental em arquiteturas em camadas:

```typescript
// Tipos de Domínio em types/medications.types.ts
export type Medication = {
  id: number;
  patientName: string;
  medicationName: string;
  dosage: string;
  route: string;
  scheduledAt: string;
  notes: string | null;
};
```

### Benefícios Práticos:
1. **Contrato Rígido entre Camadas:** Se a estrutura do banco mudar, o compilador alerta no Service antes que o erro chegue ao Controller ou ao Frontend.
2. **Prevenção de Bugs com `null`:** Ao tipar `findById(id): Medication | null`, o TypeScript obriga o desenvolvedor no Controller a fazer a checagem `if (!medication)` antes de acessar propriedades.
3. **Autonomia de Testes:** Facilita a criação de mocks para testes unitários isolados do Service.

```typescript
// Assinaturas ideais no Service:
function findAll(): Medication[];
function findById(id: number): Medication | null;
function create(data: createMedication): Medication;
function deleteMedication(id: number): boolean;
```

---

## 6. Consistência do Contrato da API (`error` vs. `erro`)

APIs devem manter um **contrato previsível**. Misturar chaves em diferentes idiomas ou formatos quebra os interceptors de requisição do frontend:

```typescript
// ❌ Inconsistente (ora inglês, ora português)
return res.status(400).json({ error: "id inválido" });
return res.status(400).json({ erro: "id inválido" });

// ✅ Padronizado em toda a aplicação
return res.status(400).json({ error: "id inválido" });
return res.status(404).json({ error: "Prescrição não encontrada" });
return res.status(500).json({ error: "Erro interno do servidor" });
```

---

## 7. Síntese do Fluxo Refinado

```
[ DELETE /api/medications/:id ]
              │
              ▼
    [ Controller: suspend ]
              │
              ├─ Valida formato do ID ──(Inválido)──> 400 Bad Request
              │
              ▼
    [ Service: deleteMedication ]
              │
              ├─ Executa DELETE com SQLite
              │
              ├───> result.changes === 0 ─────────────> 404 Not Found
              │
              ├───> result.changes > 0 ──────────────> 204 No Content
              │
              └───> Erro no Banco (Exceção) ──────────> 500 Internal Server Error
```
