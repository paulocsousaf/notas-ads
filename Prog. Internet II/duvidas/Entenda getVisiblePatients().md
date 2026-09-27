# getVisiblePatients()

1. Cria  uma variável e adiciona o atributo searchTerm, sem espaços no começo e no final, e com todas as letras minusculas
2. Retorna o resultado do seguinte filter, aplicado na lista de pacientes dentro de state
	1. Verifica se o termo obtido de state.searchTerm contem naquele paciente especifico na qual o laço se encontra, o retorno será True ou False
	2. Verifica se o filtro está ativo, se sim: retorna o valor boleano do atributo ativo que pertence ao paciente em especifico. Se for falso: ele smplesmente retorna true. O filtro estando False, significa que podemos listar todos os pacientes
	*E se não tiver nenhum termo de busca


A função `getVisiblePatients()` no arquivo [`state.js`](file:///home/paulo/Projetos/01-projeto-base/01-projeto-base/public/js/state.js) é responsável por **calcular e retornar a lista de pacientes que devem aparecer na tela**, combinando os filtros aplicados pelo usuário. 

Ela atua como um dado derivado (ou calculado) a partir do estado principal. Em vez de guardar a lista filtrada permanentemente, ela pega a lista completa (`state.patients`) e a filtra no momento que é chamada, seguindo duas regras:

1. **Filtro de busca (`searchTerm`):** 
   Ela pega o texto da busca (removendo os espaços com `.trim()` e convertendo para letras minúsculas) e verifica se o nome do paciente inclui esse texto.
2. **Filtro de ativos (`onlyActive`):** 
   Se a variável `state.onlyActive` for `true`, ela só inclui na lista final os pacientes que tiverem a propriedade `active` verdadeira. Se for `false`, ela deixa passar todo mundo.

Ela retorna a lista contendo apenas os pacientes que passarem por essas duas condições.

Como diz o comentário no código, essa é uma **função pura**: ela não altera nada no estado do sistema (não tem "efeitos colaterais"), apenas lê os dados de entrada (`state`) e devolve uma nova lista filtrada como saída.

---
Com certeza! Vamos destrinchar a função linha por linha para entender exatamente o que acontece em cada passo:

```javascript
export function getVisiblePatients() {
```
* **O que faz:** Cria a função com o nome `getVisiblePatients`. 
* O termo `export` antes dela significa que esta função fica "pública", ou seja, pode ser importada e usada por outros arquivos do seu projeto.

```javascript
  const term = state.searchTerm.trim().toLowerCase();
```
* **O que faz:** Prepara o texto que o usuário digitou na busca antes de começarmos a comparar.
* Ela pega a variável `state.searchTerm` (o que está no campo de busca), usa o `.trim()` para remover espaços em branco acidentais no início e no fim, e o `.toLowerCase()` para transformar todas as letras em minúsculas. O resultado disso é guardado na variável `term`. *Fazer tudo virar minúsculo garante que buscar por "Maria", "maria" ou "MARIA" traga o mesmo resultado.*

```javascript
  return state.patients.filter((patient) => {
```
* **O que faz:** Inicia o processo de devolver (`return`) uma nova lista.
* Ela pega a lista completa de pacientes guardada no estado (`state.patients`) e aplica o método `.filter()`. 
* O `.filter()` é como uma peneira: ele vai olhar para um paciente de cada vez (representado pela palavra `patient` dentro dos parênteses) e vai rodar o bloco de código abaixo para decidir se esse paciente passa pela peneira ou não.

```javascript
    const matchesTerm = patient.name.toLowerCase().includes(term);
```
* **O que faz:** Testa se o nome do paciente atual da peneira "bate" com o que o usuário buscou.
* Ela pega o nome do paciente (`patient.name`), transforma em minúsculas (`toLowerCase()`) e checa se esse nome inclui (`includes()`) o texto preparado na variável `term`. O resultado disso será `true` (verdadeiro, o nome tem o texto) ou `false` (falso, não tem), e esse valor é guardado em `matchesTerm`.

```javascript
    const matchesStatus = state.onlyActive ? patient.active : true;
```
* **O que faz:** Testa se o status do paciente bate com o filtro de ativos.
* Essa linha usa um "if" encurtado chamado **operador ternário** (a parte com o `?` e `:`). Funciona assim:
  * "O filtro de mostrar apenas ativos (`state.onlyActive`) está ligado?"
  * Se **SIM** (`?`), olhe se este paciente está ativo de fato (`patient.active`).
  * Se **NÃO** (`:`), simplesmente considere como `true` (verdadeiro). Se o filtro não está ligado, nós aceitamos todos os pacientes, ativos ou não.
* O resultado de `true` ou `false` é guardado na variável `matchesStatus`.

```javascript
    return matchesTerm && matchesStatus;
```
* **O que faz:** É a decisão final da peneira para este paciente específico.
* O operador `&&` significa **"E"**. Ou seja, ele só vai devolver `true` (deixando o paciente passar para a lista visível) se `matchesTerm` for verdadeiro **E** `matchesStatus` também for verdadeiro. Se qualquer um dos dois falhar, ele devolve `false` e o paciente fica de fora da tela.

```javascript
  });
}
```
* **O que faz:** A primeira linha `});` fecha o bloco do `.filter()`. A segunda linha `}` fecha a função inteira `getVisiblePatients`.

---
Se o campo de busca não tiver nada (ou seja, `state.searchTerm` for uma string vazia `""`), **todos os pacientes vão passar na validação de nome**.

Veja como o código se comporta passo a passo nesse cenário:

1. Ao rodar `const term = state.searchTerm.trim().toLowerCase();`, a variável `term` recebe uma string vazia: `""`.
2. Quando a função for checar cada paciente na linha `patient.name.toLowerCase().includes(term)`, ela estará na verdade fazendo isso: `includes("")`.
3. Em JavaScript, se você perguntar se um texto qualquer inclui um texto vazio (uma string vazia `""`), **a resposta sempre será `true` (verdadeiro)**.

Portanto, a variável `matchesTerm` será `true` para absolutamente todos os pacientes da sua lista. 

Nesse caso, a busca por texto é ignorada com sucesso, e o que vai decidir se o paciente aparece ou não será apenas a segunda regra (se o botão de "mostrar apenas ativos" está ligado ou não). O sistema vai listar todo mundo (ou todos os ativos, dependendo do outro filtro).