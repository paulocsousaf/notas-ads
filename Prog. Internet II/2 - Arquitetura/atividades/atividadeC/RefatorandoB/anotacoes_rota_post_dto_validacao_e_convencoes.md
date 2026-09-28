# Anotações de Estudo: Rota POST, DTOs, Validação e Divisão de Responsabilidades

Este documento consolida os conceitos arquiteturais, padrões de design, boas práticas de tipagem com TypeScript e convenções de versionamento discutidos durante o desenvolvimento da rota de criação de prescrições (**POST `/api/medications`**) no projeto **Painel de Medicação**.

---

## 1. Controller vs. Service: De quem é a responsabilidade de tratar os dados?

Na arquitetura em camadas, **ambas** as camadas tratam dados, mas em **níveis de abstração e contextos totalmente diferentes**.

```
[ Cliente HTTP ]
       │
       ▼ (JSON Bruto / req.body)
┌──────────────┐  Validação Sintática / de Formato
│  Controller  │  (Campos obrigatórios, tipos primitivos, regex de data)
└──────┬───────┘  Retorna HTTP 400 Bad Request se inválido
       │
       ▼ (Dados tipados / DTO)
┌──────────────┐  Validação Semântica / Regras de Negócio
│   Service    │  (Invariantes, consistência, conflitos de horário)
└──────┬───────┘  Retorna entidade ou erro de domínio
       │
       ▼ (Query SQL parametrizada)
[ Banco de Dados ]
```

### A. O papel do Controller (Porteiro / Tradutor HTTP)
* **Contexto:** Conhece o protocolo HTTP (`req`, `res`, headers, status codes, query params, cookies).
* **Responsabilidade com os dados:**
  1. Extrair os dados da requisição (`req.body`).
  2. Executar a **validação sintática**: verificar se os campos obrigatórios vieram preenchidos (`isBlank`) e se atendem ao formato básico (ex: regex para ISO 8601).
  3. Se houver falha de formato, barrar imediatamente respondendo com status **`400 Bad Request`**.
  4. Repassar dados limpos e tipados para o Service.
* **O que NÃO deve fazer:** Não deve executar queries de banco de dados nem aplicar regras de negócio da clínica/hospital.

### B. O papel do Service (Cérebro do Negócio)
* **Contexto:** É agnóstico ao meio de transporte. Não sabe o que é HTTP, `req` ou `res`.
* **Responsabilidade com os dados:**
  1. Receber dados puros (objetos tipados pelo TypeScript).
  2. Executar a **validação semântica / regras de negócio** (ex: dosagem compatível, checagem de interações medicamentosas, validação se o paciente já tem medicação no mesmo horário).
  3. Preparar e sanitizar os dados para persistência (ex: converter texto vazio em `null` para campos opcionais).
  4. Orquestrar a inserção no banco de dados e devolver a entidade criada já no formato de domínio.
* **O que NÃO deve fazer:** Não deve manipular status HTTP (`res.status(201)`) nem depender de objetos do Express.

---

## 2. O Service deve "confiar cegamente" no Controller?

**Não.** Embora o Controller funcione como primeiro filtro, o Service não deve ser negligente:

1. **Autonomia do Domínio:** Se no futuro a função `medicationService.create()` for chamada por um teste unitário, por uma tarefa em segundo plano (*cron job*) ou por uma fila de mensagens, ela deve continuar funcionando de maneira íntegra sem depender de um Controller HTTP.
2. **Defesa em Profundidade:**
   * O Controller garante que a requisição web não traga lixo.
   * O Service confia no **contrato de tipos** (definido pelo TypeScript), mas cuida ativamente da integridade e formatação final dos dados para o armazenamento.

---

## 3. Por que tipar em TypeScript se os tipos somem após compilar?

O TypeScript utiliza **Type Erasure** (apagamento de tipos): ao compilar para JavaScript puro executado pelo Node.js, todas as interfaces e `type` são removidos. 

Ainda assim, a tipagem estática é indispensável por quatro razões:

1. **Prevenção de Erros de Digitação (*Typos*):** Evita que o desenvolvedor passe `pacientName` no Controller e tente ler `patientName` no Service. O TypeScript acusa o erro com linha vermelha no editor antes de rodar o código.
2. **Autocomplete Inteligente (*IntelliSense*):** Ao digitar `data.`, a IDE exibe imediatamente as propriedades disponíveis, acelerando o desenvolvimento.
3. **Refatoração Segura:** Se um novo campo obrigatório for adicionado ao sistema, o compilador lista todos os pontos do projeto que precisam ser atualizados.
4. **Documentação Viva e Contrato:** A assinatura `create(data: CreateMedicationDTO)` comunica explicitamente tudo o que o método necessita para operar.

> **Importante:** A validação em tempo de execução (`validateInput`, `isBlank`) protege o sistema contra o **usuário externo**. Os tipos do TypeScript protegem o sistema contra o **próprio programador**.

---

## 4. O Padrão DTO (*Data Transfer Object*)

### O que é?
Um **DTO** é um objeto simples cujo único propósito é **transportar dados entre processos ou camadas**, sem comportamentos, métodos ou regras complexas embutidas.

### Por que usar o sufixo `DTO`?
Ajuda a distinguir claramente os diferentes momentos em que o mesmo conceito circula na aplicação:

| Tipo | Onde atua | Características |
| :--- | :--- | :--- |
| `medicationRow` | Banco de Dados (SQLite) | Reflete a tabela: `snake_case` (`patient_name`), tipos do SQLite, possui `id`. |
| `CreateMedicationDTO` | Entrada do Service | Dados necessários **apenas para criar**: campos em `camelCase`, **sem `id`** (pois o ID ainda não existe). |
| `Medication` | Resposta da API | Representação completa para o cliente: campos em `camelCase`, possui `id`. |

*(Nota: O sufixo `DTO` é uma convenção de mercado consagrada. Nomes como `CreateMedicationInput` ou `CreateMedicationPayload` também são válidos).*

---

## 5. Tratamento de Campos Opcionais no Banco de Dados

Ao persistir um campo opcional (como `notes`), é necessário garantir que a ausência de valor seja representada adequadamente no banco relacional.

### A linha analisada:
```typescript
const notesFormatted = data.notes && data.notes.trim() !== "" ? data.notes.trim() : null;
```

### Decomposição:
1. **`data.notes &&` (Proteção contra `undefined`/`null`):**
   * Impede a execução de `.trim()` sobre valores nulos ou indefinidos, o que causaria um `TypeError: Cannot read properties of undefined`.
2. **`data.notes.trim() !== ""` (Proteção contra espaços em branco):**
   * Se o usuário digitou apenas espaços (`"   "`), o `.trim()` resulta em string vazia `""`. A condição avalia para falso.
3. **`? data.notes.trim() : null` (Operador Ternário):**
   * Se houver texto real, armazena o texto limpo sem espaços excedentes nas bordas.
   * Se não houver texto válido, armazena `null`.

### Alternativa moderna com *Optional Chaining*:
```typescript
const notesFormatted = data.notes?.trim() || null;
```

### Por que salvar `null` e não `""`?
* No SQLite ([schema.sql](file:///home/paulo/Projetos/arquitetura-camadas/painel-medicacao/database/schema.sql)), a coluna `notes` não possui `NOT NULL`.
* Em bancos relacionais, a convenção padrão para a ausência de dado é `NULL`.
* Facilita a vida do frontend: checagens simples como `if (medication.notes)` funcionam de forma previsível.

---

## 6. Conversão de Nomenclatura: `snake_case` vs. `camelCase`

* O banco de dados armazena colunas em `snake_case` (`patient_name`, `scheduled_at`).
* A API e o JavaScript trabalham em `camelCase` (`patientName`, `scheduledAt`).
* **Responsabilidade:** A função de mapeamento (`toMedicationJson`) pertence à **camada de Service**, pois é ela quem conversa diretamente com o banco de dados e produz a representação de negócio.
* O Controller não precisa chamar funções de conversão; ele apenas repassa a resposta já padronizada devolvida pelo Service.

---

## 7. Boas Práticas de Git: `feat` vs. `refactor`

Ao registrar mudanças com *Conventional Commits*:

* **Use `feat`:** Quando a alteração introduz um **novo comportamento ou recurso observável** (ex: a rota `POST /api/medications` foi criada agora e antes não existia).
  ```text
  feat(medications): implementar criacao de prescricao (POST /)
  ```
* **Use `refactor`:** Quando você **reorganiza internamente o código sem alterar o comportamento externo** (ex: o `POST` já existia no `server.ts` e você apenas distribuiu o código entre `controller` e `service`).
  ```text
  refactor(medications): modularizar criacao de prescricao em camadas
  ```

---

## 8. Checklist de Code Review para Novas Rotas

Antes de finalizar qualquer endpoint em camadas:
- [ ] O Controller valida a presença e o formato dos parâmetros obrigatórios e responde `400` em caso de erro?
- [ ] O Controller trata exceções com bloco `try/catch` e responde com `500` em caso de falha inesperada?
- [ ] O Service recebe dados tipados sem depender de objetos HTTP (`req`, `res`)?
- [ ] O retorno do Service foi convertido para a nomenclatura de domínio (`camelCase`)?
- [ ] Foram verificados e removidos imports acidentais adicionados pelo editor (ex: `import { create } from "domain"`)?
