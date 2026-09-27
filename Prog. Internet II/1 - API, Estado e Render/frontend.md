# Frontend

## Aquecimento

- [x] Lista de pacientes vinda de um json
- [x] Busca que filtra enquanto digita 
- [x] Cartão responsivo: 1 coluna no celular, 3 no monitor

### Por que a tela mente?

- Existe diferença do Estado e do DOM
- O estado mudo e a tela é apenas consequência do estado 

EVENTO --> AÇÃO -->  ESTADO -->  RENDER -->  TELA

Evento: ação do usuário. Click, input 
Ação: única que pode mudar o estado
Estado: fonte da verdade
Render: desenha o estado, não decide nada
Tela: resultado do estado desenhado 

## Estado
### Estrutura do frontend

Isto não é “arquitetura”. É só arrumação
#### app.js

O maestro: 
- escuta eventos
- chama ações
- manda redesenhar
- **não guarda dado**
#### state.js

Livro de registro: 
- guarda o estado
- **não conhece o DOM**

#### render.
Desenhista: 
- recebe o que deve aparecer na tela
- Não decide nada
- Não ordena
- Não calcula
#### api.js

Porta única:
- Único arquivo autorizado a chamar fetch
- Ninguém mais sabe que existe HTTP
## Derivado
- A lista filtrada não está no estado. E isso é uma decisão, não um esquecimento.
- A lista filtrada é consequência de `patients` + `searchTerm`  
- Não guardamos consequência no estado! Guardar consequência no estado é criar duas verdades

## CSS

- Mobile primeiro 
- Analogia da mala: primeira você arruma a mala pequena, para depois organizar a grande
- a versão mais restrita é a que recebeu mais atenção.