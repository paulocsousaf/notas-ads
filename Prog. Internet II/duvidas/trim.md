O método `.trim()` é amplamente utilizado em várias linguagens de programação (como JavaScript, Java, C#, entre outras) para **remover os espaços em branco do início e do final de uma string** (texto).

Ele não altera os espaços que estão _no meio_ do texto, apenas os que ficam nas extremidades.

### Exemplo em JavaScript:

```javascript

const texto = "   Olá, mundo!   ";
const textoSemEspacos = texto.trim();
console.log(texto);           // Saída: "   Olá, mundo!   "
console.log(textoSemEspacos); // Saída: "Olá, mundo!"
```

### O que ele remove exatamente?

Normalmente, o `.trim()` remove:

- Espaços normais ( )
- Quebras de linha (`\n`)
- Tabulações (`\t`)
- Outros caracteres de espaço em branco definidos pela linguagem.

### Variações úteis

Em algumas linguagens (como JavaScript e PHP), existem também métodos específicos se você quiser remover espaços apenas de um lado:

- `.trimStart()` (ou `.trimLeft()`): Remove espaços apenas do **início** da string.
- `.trimEnd()` (ou `.trimRight()`): Remove espaços apenas do **final** da string.

**Onde isso é útil?** É extremamente útil ao limpar dados inseridos por usuários em formulários. Por exemplo, se alguém digitar seu nome com espaços extras sem querer, o `.trim()` garante que você salve no banco de dados apenas o nome em si