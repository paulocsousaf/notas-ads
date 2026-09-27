- Comesse extraindo as regras de negócio
- Quando chega no service, o percurso é voltar pelo controller ? Ou já manda pra o cliente?
- Qual a diferença de Express para express? '"express"' has no exported member named 'express'. Did you mean 'Express'?ts(2724)
- Qual a diferença de importa com chaves e importar sem chaves

- O server deve continuar existindo. É ele que inicializa o servidor

```
Service --> Router --> Controller --> Service
```

### Server

- O server deve conhecer as rotas!
### Route
- O route deve utilizar o roteador do Express (`express.Router()`) e repassar a requisição para o **controller**:

### Controller

- O controller sabe que existe HTTP 

### Service
```
Service --> Controller --> Router --> CLIENTE
```
- O service não sabe o que é HTTP, nem req e res e nem verbos HTTP. Nele só contém as regras de negócios e acesso a camada de dados
- O Service só recebe dados puros.