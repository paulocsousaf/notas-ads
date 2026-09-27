**renderApp**(getState())

## A render App pode fazer 3 coisas

### Renderizar tela de ERROR

Se state.errorMensage === true -> Renderiza error
finaliza a função renderApp()

### Renderizar tela de CARREGAMENTO

Se state.isLoanding === true -> Renderizar tela de carregamento
finaliza função renderApp()

### Renderizar Lista de pacientes e contador de pacientes

Caso nenhuma das opçoes a cima:

Renderiza lista de pacientes
Renderiza contador