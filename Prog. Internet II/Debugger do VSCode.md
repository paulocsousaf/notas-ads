Usar o Debugger é uma boa prática e pode te ajudar a entender o códig, agora só resta entender como faz essa merda

- 🔴 **Breakpoint (Ponto de Parada):** É uma marcação vermelha que você coloca ao lado do número da linha do código (basta clicar no espaço à esquerda do número). Quando o programa chegar nessa linha, ele vai "congelar".
- ⏯️ **Continue / Play (F5):** Retoma a execução do programa até ele encontrar o próximo Breakpoint ou terminar.
- ⏭️ **Step Over (F10):** Executa a linha atual e pula para a **próxima linha do arquivo atual**. Se a linha atual chamar uma função, ele executa a função inteira de uma vez sem entrar nela.
- ⬇️ **Step Into (F11):** Se a linha atual chamar uma função, ele "entra" dentro dessa função para que você possa debugar o que acontece lá dentro, linha por linha.
- ⬆️ **Step Out (Shift+F11):** Se você entrou em uma função usando o _Step Into_ e já viu o que queria, esse botão executa o resto da função atual de uma vez e volta para o arquivo que a chamou.
- 🔄 **Restart (Ctrl+Shift+F5):** Reinicia a aplicação e a sessão de debug.
- ⏹️ **Stop (Shift+F5):** Encerra a sessão de debug.


- Aonde o código js está sendo executado: no Servidor ou no Navegador
- `"type": "node"` e `"type": "chrome"
```json

	"configurantions": 
		[
			{
				"tye: node"
			},
		
			{
				"type: chrome"
			}
		]

```

- Usar o próprio DevTools do Chrome para debuggar o frontend
### Type node

- Código Js roda no SO via Node
- Usado para debugar rotas no Express, conexão com banco e lógica de negócios

### Type chrome


- Debugar o Frontend dentro do navegador Chrome
- Aqui o código Js está rodando dentro do Navegador e não no node

## Tipos de Depuração: Launch e Attach

- Launch: inicia o programa e tem o controle do processo
- Attach: se conecta com o programa, que já foi iniciado
## Arquivo launch.json

- Configura com o debugger vai rodar

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Program",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}\\app.js"
    }
  ]
}

```

```js
const a = 10

	const b = 20
console.log(a + b)
```

