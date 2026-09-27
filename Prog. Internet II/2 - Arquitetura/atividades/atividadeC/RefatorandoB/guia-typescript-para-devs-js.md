# Guia de TypeScript para Devs JavaScript

> **Para quem é este guia?**
> Para quem já sabe JavaScript, mas está aprendendo TypeScript e quer entender *por que* e *quando* usar cada recurso — sem a confusão das linguagens de POO clássica como Java.

---

## O que o TypeScript realmente é

O TypeScript é um **superconjunto** do JavaScript. Isso significa que:

- Todo arquivo `.js` válido é um arquivo `.ts` válido.
- Você não aprende uma nova linguagem; você aprende a **anotar** o JavaScript que já escreve.
- Quando compilado, todo código TypeScript se torna JavaScript puro. `type`, `interface` e anotações de tipo **desaparecem** em produção.

```
Seu .ts  ──► Compilador TypeScript (tsc) ──► .js (JavaScript puro)
         (análise de tipos em dev)          (zero overhead em produção)
```

---

## 1. Tipos Primitivos

Em JavaScript você já usa esses valores o tempo todo. No TypeScript, você apenas declara explicitamente qual tipo uma variável aceita.

```ts
// JavaScript
let nome = "Paulo";
let idade = 25;
let ativo = true;

// TypeScript — a diferença é só a anotação ': tipo'
let nome: string = "Paulo";
let idade: number = 25;
let ativo: boolean = true;
```

> [!TIP]
> Na maioria dos casos, você **não precisa** escrever o tipo. O TypeScript é inteligente o suficiente para **inferir** o tipo pelo valor:
> ```ts
> let nome = "Paulo";  // TypeScript já sabe que é string, sem você dizer nada
> nome = 42;           // ERRO: 'number' não pode ser atribuído a 'string'
> ```
> Reserve as anotações explícitas para quando a inferência não for suficiente (parâmetros de funções, retornos complexos).

---

## 2. Tipando Objetos com `type` e `interface`

Esta é a funcionalidade mais importante do TypeScript para o dia a dia. Serve para descrever o **formato (shape)** de um objeto.

### `type` — Para formatos de dados

```ts
// JS: você nunca sabia ao certo quais campos vinham do banco
function processar(row) {
  return row.patient_name; // Será que é 'patient_name'? 'patientName'? 'paciente'?
}

// TS: você declara o contrato uma vez, e o editor garante pra sempre
type MedicationRow = {
  id: number;
  patient_name: string;
  medication_name: string;
  dosage: string;
  route: string;
  scheduled_at: string;
  notes: string | null;
};

function processar(row: MedicationRow) {
  return row.patient_name; // Autocomplete completo. Erro imediato se errar o nome.
}
```

### `interface` — Para contratos que podem ser estendidos

```ts
interface Animal {
  nome: string;
  emitirSom(): void;
}

// Interfaces podem ser "estendidas" com herança
interface Cachorro extends Animal {
  raca: string;
}
```

### `type` vs `interface` — Quando usar cada um?

| | `type` | `interface` |
|:---|:---|:---|
| Formato de objetos simples | ✅ Preferido | ✅ Funciona |
| Uniões (`string \| null`) | ✅ Único que funciona | ❌ Não suporta |
| Extensão com herança | ✅ Com `&` | ✅ Com `extends` |
| Representar classes | ✅ Funciona | ✅ Preferido |

**Regra prática:** No contexto de APIs e bancos de dados, use **`type`**. Reserve `interface` para quando precisar de herança entre tipos.

---

## 3. Campos Opcionais e Valores Nulos

```ts
type Medicamento = {
  id: number;
  nome: string;
  observacoes?: string;     // '?' = campo pode não existir (undefined)
  reacao: string | null;    // pode ser string OU null
};

// Acessar campo que pode ser null/undefined sem travar o programa:
const med: Medicamento = { id: 1, nome: "Dipirona", reacao: null };

// ❌ PERIGO — lança erro se 'observacoes' for undefined
console.log(med.observacoes.toUpperCase());

// ✅ SEGURO — Optional Chaining: só executa se não for null/undefined
console.log(med.observacoes?.toUpperCase());

// ✅ SEGURO — Nullish Coalescing: valor padrão se for null/undefined
console.log(med.observacoes ?? "Sem observações");
```

---

## 4. Funções Tipadas

A tipagem de funções é onde o TypeScript realmente brilha, porque documenta e protege a "porta de entrada e saída" do seu código.

```ts
// JS — qualquer coisa entra, qualquer coisa sai, silêncio total
function somar(a, b) {
  return a + b;
}
somar("2", 3); // Retorna "23" (concatenação). Sem aviso!

// TS — você define o contrato da função
function somar(a: number, b: number): number {
  return a + b;
}
somar("2", 3); // ERRO imediato: Argument of type 'string' not assignable to 'number'
```

### Tipando Arrow Functions

```ts
// Parâmetros tipados, retorno inferido
const dobrar = (n: number) => n * 2;

// Parâmetros e retorno explícitos
const dividir = (a: number, b: number): number => a / b;

// Função que não retorna nada usa 'void'
const logar = (mensagem: string): void => {
  console.log(mensagem);
};
```

### Tipando callbacks (muito útil no Express)

```ts
import { Request, Response } from "express";

// Sem tipo: você nunca sabe o que tem dentro de req e res
app.get("/", (req, res) => {
  res.sattus(200).json({}); // Erro de digitação silencioso no JS!
});

// Com tipo: o editor avisa imediatamente que 'sattus' não existe
const handler = (req: Request, res: Response): void => {
  res.status(200).json({});
};
```

---

## 5. Narrowing — Refinando Tipos em Tempo de Execução

Quando um valor pode ter mais de um tipo (ex: `string | null`), o TypeScript exige que você **verifique qual é** antes de usá-lo. Isso é chamado de *narrowing*.

```ts
type Resultado = string | null;

function processar(valor: Resultado) {
  // TypeScript não deixa chamar .toUpperCase() sem verificar antes
  
  // ✅ Narrowing com 'if'
  if (valor === null) {
    return "Sem valor";
  }
  // Aqui dentro, TypeScript já sabe que 'valor' é string
  return valor.toUpperCase();
}
```

```ts
// Narrowing com 'typeof' — para tipos primitivos
function formatar(entrada: string | number) {
  if (typeof entrada === "string") {
    return entrada.trim();   // TypeScript sabe que é string aqui
  }
  return entrada.toFixed(2); // TypeScript sabe que é number aqui
}
```

---

## 6. `unknown` vs `any` — O vilão e o herói

### `any` — O tipo que desativa o TypeScript (evite)

```ts
let dado: any = "Paulo";
dado = 42;             // Ok
dado = { a: 1 };       // Ok
dado.metodoInventado(); // Ok pra o TS, quebra em tempo de execução!
```

Usar `any` é dizer ao TypeScript: *"Pode parar de me avisar sobre este valor."* Derrota o propósito da ferramenta.

### `unknown` — A alternativa segura

```ts
function processar(dado: unknown) {
  dado.toUpperCase(); // ERRO: 'dado' pode ser qualquer coisa

  // ✅ Você é obrigado a verificar antes de usar
  if (typeof dado === "string") {
    dado.toUpperCase(); // Agora o TS sabe que é string
  }
}
```

**Regra prática:** Se você está recebendo dados de uma fonte externa (JSON de API, `req.body`, `catch(error)`) e não sabe o tipo, use `unknown`. Nunca `any`.

---

## 7. Asserção de Tipo (`as`) — Use com responsabilidade

Às vezes você sabe mais do que o TypeScript. O `as` diz: *"Confie em mim, eu sei que este valor é deste tipo."*

```ts
// O método .all() do SQLite retorna 'unknown[]' por padrão
const rows = db.prepare("SELECT * FROM medicamentos").all();

// Com 'as', você assume a responsabilidade pelo tipo
const medicamentos = rows as MedicationRow[];
medicamentos.map(row => row.patient_name); // ✅ Agora funciona com autocomplete
```

> [!WARNING]
> O `as` **não faz conversão de dados** em tempo de execução. Se os dados reais não correspondem ao tipo declarado, o programa vai quebrar de formas inesperadas. Use quando você tem **certeza** do formato dos dados (ex: dados vindos diretamente do seu próprio banco com schema conhecido).

---

## 8. Generics — Funções que funcionam com qualquer tipo

Generics permitem criar funções, tipos e classes que **são genéricos** e funcionam com múltiplos tipos, mantendo a segurança.

```ts
// Sem generics: você perde a informação do tipo
function primeiro(lista: any[]): any {
  return lista[0];
}
const x = primeiro([1, 2, 3]); // x é 'any' — TypeScript não sabe que é número

// Com generics: o tipo é preservado
function primeiro<T>(lista: T[]): T | undefined {
  return lista[0];
}
const x = primeiro([1, 2, 3]);       // x é 'number' ✅
const y = primeiro(["a", "b", "c"]); // y é 'string' ✅
const z = primeiro([]);              // z é 'undefined' ✅
```

Você não precisa criar seus próprios generics no começo. O importante é **reconhecê-los** ao usar bibliotecas. Por exemplo, no `better-sqlite3`:

```ts
// O '.get<T>()' é um generic do better-sqlite3
const row = db.prepare("SELECT * FROM ...").get<MedicationRow>(id);
// row é 'MedicationRow | undefined' automaticamente, sem precisar do 'as'
```

---

## 9. Classes no TypeScript — Quando e Como

> [!IMPORTANT]
> Classes **não são obrigatórias** no TypeScript. Use-as apenas quando fizer sentido, não por hábito de Java/C#.

### Quando usar classes

```ts
// ✅ BOM USO: Quando há estado interno que precisa ser protegido e validado
class CarrinhoDeCompras {
  private itens: Item[] = [];  // 'private' — só acessível dentro da classe

  adicionar(item: Item): void {
    if (item.preco <= 0) {
      throw new Error("Preço inválido");
    }
    this.itens.push(item);
  }

  get total(): number {
    return this.itens.reduce((soma, i) => soma + i.preco, 0);
  }
}
```

```ts
// ✅ BOM USO: Erros customizados com herança
class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    message: string
  ) {
    super(message);
    this.name = "AppError";
  }
}

// Agora você pode lançar erros com código HTTP:
throw new AppError(404, "Receita não encontrada");

// E capturar com 'instanceof':
if (error instanceof AppError) {
  res.status(error.statusCode).json({ error: error.message });
}
```

### Quando NÃO usar classes (prefira funções/objetos)

```ts
// ❌ DESNECESSÁRIO: Classe sem estado só para agrupar métodos
class MedicationService {
  findAll() { ... }
  findById(id: number) { ... }
}
export const service = new MedicationService();

// ✅ MAIS IDIOMÁTICO NO NODE.JS: Objeto simples ou funções exportadas
export const medicationService = {
  findAll(): MedicationResponse[] { ... },
  findById(id: number): MedicationResponse | null { ... },
};
```

---

## 10. Estrutura Recomendada de Tipos no Projeto

Para projetos como o seu `painel-medicacao`, uma boa organização é:

```
src/
├── types/
│   └── medication.types.ts   # Todos os 'type' e 'interface' do domínio
├── services/
│   └── medication.service.ts  # Lógica de negócio (funções/objeto)
├── controllers/
│   └── medication.controller.ts
├── routes/
│   └── medication.route.ts
└── database.ts
```

**`src/types/medication.types.ts`**
```ts
// Representa uma linha bruta do banco (snake_case)
export type MedicationRow = {
  id: number;
  patient_name: string;
  medication_name: string;
  dosage: string;
  route: string;
  scheduled_at: string;
  notes: string | null;
};

// Representa o formato da resposta JSON da API (camelCase)
export type MedicationResponse = {
  id: number;
  patientName: string;
  medicationName: string;
  dosage: string;
  route: string;
  scheduledAt: string;
  notes: string | null;
};

// Representa o body de uma requisição POST
export type CreateMedicationBody = {
  patientName: string;
  medicationName: string;
  dosage: string;
  route: string;
  scheduledAt: string;
  notes?: string;
};
```

---

## 11. Os 5 Mitos sobre TypeScript

| Mito ❌ | Verdade ✅ |
|:---|:---|
| "TypeScript exige usar classes" | TypeScript não exige nada além de tipagem. Classes são opcionais. |
| "TypeScript é mais lento que JS" | TypeScript só existe em desenvolvimento. Em produção é JS puro, idêntico. |
| "Devo anotar o tipo de tudo" | A inferência cuida da maioria dos casos. Anote só onde necessário. |
| "TypeScript pega todos os erros" | Ele pega erros de *tipo*, não de lógica. Testes ainda são necessários. |
| "Preciso reescrever tudo de JS para TS" | Você pode migrar gradualmente. Todo `.js` já é um `.ts` válido. |

---

## Resumo Final: O Mental Model

```
TypeScript = JavaScript + Descrição do Formato dos Dados

type/interface  →  Descrevem o FORMATO de dados (desaparecem em produção)
funções         →  Implementam o COMPORTAMENTO (sem estado → sem classe)
class           →  Quando há ESTADO INTERNO + COMPORTAMENTO (use com critério)
any             →  Foge da tipagem (evite sempre que possível)
unknown         →  Dado de fonte externa que precisa ser verificado antes de usar
as              →  "Eu sei mais que você, TypeScript" (use com responsabilidade)
```
