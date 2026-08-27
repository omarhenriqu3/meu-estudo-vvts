# Laboratório 02 — Estratégias e Níveis de Teste na Prática
**Domínio:** Plataforma de E-commerce (Shop 20) - Função "Adicionar ao Carrinho"

---

## 1. Teste de Unidade (Unit Testing)

### 1.1 — Verificação de lógica atômica em componente/classe isolada

#### 1. Diagrama de Classes UML
```mermaid
classDiagram
    class CalculadoraDeDescontoCarrinhoTest {
        <<Classe de Teste (Unit Testing)>>
        +testarDescontoMaximoVipComCincoItens() void
        +testarCondicaoDeBordaQuantidadeCinco() void
        +testarExcecaoParaQuantidadeNegativa() void
    }

    class CalculadoraDeDescontoCarrinho {
        <<Classe Sob Teste (SUT)>>
        +calcularDescontoProgressivo(valorTotalItem, quantidade, perfilCliente) Double
    }

    CalculadoraDeDescontoCarrinhoTest ..> CalculadoraDeDescontoCarrinho : instancia e exercita
```

#### 2. Explicação Textual do Cenário
**Contexto:** No Shop20, usamos o Teste de Unidade para testar a classe CalculadoraCarrinho totalmente sozinha, sem conectar com o banco de dados. A classe de teste envia as informações (valor, quantidade, perfil VIP) direto para a calculadora e confere se o resultado final bate com o esperado. Isso serve para achar erros nas contas matemáticas ou bloquear dados errados (como uma quantidade negativa) logo no começo da programação, antes de juntar com o resto do sistema.

**Como a abordagem é aplicada:** o método `testarDescontoMaximoVipComCincoItens()` confirma que um cliente VIP com 5 itens recebe o percentual máximo de desconto; `testarCondicaoDeBordaQuantidadeCinco()` verifica o comportamento exatamente no limiar que muda a faixa de desconto; e `testarExcecaoParaQuantidadeNegativa()` garante que `calcularDescontoProgressivo()` rejeita uma quantidade inválida em vez de retornar um valor incorreto silenciosamente.

**Objetivo e Defeitos Revelados:** Garantir a corretude da lógica matemática. Revela cálculos incorretos e validar perfil do cliente antes de juntar com o resto do sistema.

---

## 2. Teste de Integração (Integration Testing)

### 2.1 — Integração Não Incremental (Big Bang)

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant Controller as CarrinhoController
    participant Service as CarrinhoService
    participant Estoque as GatewayEstoque
    participant Calculadora as CalculadoraCarrinho
    participant Repo as CarrinhoRepository

    Cliente->>Controller: adicionarItem(produtoId, quantidade)
    activate Controller

    Note over Controller,Repo: Integração Big Bang: Todos os componentes reais acionados juntos

    Controller->>Service: adicionarAoCarrinho(usuarioId, produtoId, quantidade)
    activate Service

    Service->>Estoque: verificarEstoque(produtoId, quantidade)
    activate Estoque
    Estoque-->>Service: true (Estoque confirmado)
    deactivate Estoque

    Service->>Calculadora: calcularDescontoProgressivo(valorTotalItem, quantidade, perfilCliente)
    activate Calculadora
    Calculadora-->>Service: Valor calculado
    deactivate Calculadora

    Service->>Repo: salvarItem(usuarioId, produtoId, quantidade, valorCalculado)
    activate Repo
    Repo-->>Service: true (Persistido no banco)
    deactivate Repo

    Service-->>Controller: Sucesso (Carrinho atualizado)
    deactivate Service

    Controller-->>Cliente: 200 OK (Item adicionado com sucesso)
    deactivate Controller
```

#### 2. Explicação Textual do Cenário
**Contexto:** Abordagem que junta todos os módulos desenvolvidos de uma só vez para verificar se funcionam em conjunto.

**Como a abordagem é aplicada:** O teste aciona o Controller passando o ID do produto e a quantidade. O fluxo percorre os 5 módulos reais (`CarrinhoController`, `CarrinhoService`, `GatewayEstoque`, `CalculadoraCarrinho`, `CarrinhoRepository`) em sequência, até chegar ao banco de dados. Se o item aparecer salvo corretamente no final, o teste passa.
**Objetivo e Defeitos Revelados:** Validar se os módulos conversam entre si. Revela falhas de injeção de dependência e erros de comunicação de rede.

---

### 2.2 — Integração Incremental Top-Down (Descendente) com uso de Stubs

#### 1. Diagrama de Classes UML
```mermaid
classDiagram
    class CarrinhoService {
        -IGatewayEstoque gatewayEstoque
        +adicionarAoCarrinho(usuarioId, produtoId, quantidade) Boolean
    }

    class IGatewayEstoque {
        <<interface>>
        +verificarEstoque(produtoId, quantidade) Boolean
    }

    class EstoqueGatewayStub {
        <<stub / simulador>>
        -Boolean repostaProgramada
        +verificarEstoque(produtoId, quantidade) Boolean
        +configurarParaSucesso() void
        +configurarParaProdutoEsgotado() void
        +configurarParaErroDeServico() void
    }

    class GatewayEstoqueReal {
        <<em desenvolvimento>>
        +verificarEstoque(produtoId, quantidade) Boolean
    }

    CarrinhoService --> IGatewayEstoque : utiliza
    EstoqueGatewayStub ..|> IGatewayEstoque : implementa
    GatewayEstoqueReal ..|> IGatewayEstoque : implementa
```

#### 2. Explicação Textual do Cenário
**Contexto:** A classe superior precisa consultar o estoque, mas a API real está indisponível.

**Como a abordagem é aplicada:** O `CarrinhoService` (topo) é testado utilizando um `EstoqueGatewayStub` (simulador) que substitui a camada inferior. O Stub é programado com três respostas fixas distintas: `configurarParaSucesso()`, para simular item disponível; `configurarParaProdutoEsgotado()`, para simular ausência de estoque; e `configurarParaErroDeServico()`, para simular uma falha de infraestrutura no gateway real (ex.: timeout), diferente de um simples "sem estoque".

**Objetivo e Defeitos Revelados:** Revela falhas na lógica condicional de alto nível ao tratar as diferentes respostas que a camada inferior poderia enviar — incluindo se `CarrinhoService` trata um erro de serviço da mesma forma (incorreta) que trataria um produto esgotado.

---

### 2.3 — Integração Incremental Bottom-Up (Ascendente) com uso de Drivers

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    participant Driver as CarrinhoDriver
    participant Estoque as GatewayEstoque
    participant Calc as CalculadoraCarrinho
    participant Repo as CarrinhoRepository

    Note over Driver,Repo: Nota: CarrinhoService ainda não existe. O Driver orquestra a integração.

    Driver->>Estoque: verificarEstoque(produtoId, quantidade)
    activate Estoque
    Estoque-->>Driver: true (Estoque confirmado)
    deactivate Estoque

    Driver->>Calc: calcularDescontoProgressivo(valorTotalItem, quantidade, perfilCliente)
    activate Calc
    Calc-->>Driver: Valor final calculado
    deactivate Calc

    Driver->>Repo: salvarItem(usuarioId, produtoId, quantidade, valorFinal)
    activate Repo
    Repo-->>Driver: true (Persistido no banco)
    deactivate Repo
```

#### 2. Explicação Textual do Cenário
**Contexto:** Neste cenário, a base do sistema (`GatewayEstoque`, `CalculadoraCarrinho` e `CarrinhoRepository`) já está pronta, mas o serviço principal que orquestra tudo (`CarrinhoService`) ainda não foi programado.

**Como a abordagem é aplicada:** Para testar se esses componentes da base conseguem trabalhar em conjunto, criamos um simulador temporário (o `CarrinhoDriver`). Esse Driver instancia os 3 módulos reais e faz as chamadas em sequência: primeiro ele verifica o estoque, depois calcula o preço e, por fim, salva o item no banco de dados.

**Objetivo e Defeitos Revelados:** Validar que os módulos de base funcionam corretamente em conjunto antes de existir um controlador que os una. Revela erros de interface entre `GatewayEstoque`, `CalculadoraCarrinho` e `CarrinhoRepository` — por exemplo, o formato do valor retornado por `calcularDescontoProgressivo()` não sendo o que `salvarItem()` espera receber.

---

### 2.4 — Teste de Fumaça (Smoke Testing)

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    participant Pipeline as Pipeline CI/CD
    participant Runner as Runner de Smoke Test
    participant Auth as Autenticação
    participant Produto as Servico de Produto
    participant Controller as CarrinhoController
    participant Service as CarrinhoService
    participant Repo as CarrinhoRepository

    Pipeline->>Runner: dispararSmokeTest()
    activate Runner

    Note over Runner,Repo: Execução rápida do caminho crítico (Caminho Feliz)

    Runner->>Auth: healthCheck / realizarLogin()
    Auth-->>Runner: 200 OK (Token gerado)

    Runner->>Produto: consultarProduto(produtoId)
    Produto-->>Runner: 200 OK (Catálogo online)

    Runner->>Controller: POST /carrinho/adicionar
    activate Controller
    
    Controller->>Service: adicionarAoCarrinho(usuarioId, produtoId, quantidade)
    activate Service
    
    Service->>Repo: salvarItem(dados)
    activate Repo
    Repo-->>Service: Sucesso (Banco de dados respondendo)
    deactivate Repo
    
    Service-->>Controller: Integração OK
    deactivate Service
    
    Controller-->>Runner: 200 OK (Carrinho operante)
    deactivate Controller

    alt Todas as verificações passaram
        Runner-->>Pipeline: Build Aprovado (Avançar pipeline)
    else Falha em qualquer etapa (ex: Erro 500, Timeout)
        Runner-->>Pipeline: Build Rejeitado (Abortar implantação)
    end
    deactivate Runner
```

#### 2. Explicação Textual do Cenário
**Contexto:** O teste de fumaça serve para responder a uma pergunta fundamental: "O sistema básico está respirando?". Se o usuário consegue fazer o mínimo esperado, como adicionar um item ao carrinho, as integrações principais estão saudáveis e a versão é considerada "estável".

**Como a abordagem é aplicada:** Um script automatizado (Runner) dispara requisições superficiais e rápidas no caminho principal (login, consulta e adição ao carrinho) e valida conexões com as APIs e o banco, sem focar em cálculos ou cenários de erro.
**Objetivo e Defeitos Revelados:** Detecta defeitos bloqueantes instantâneos, como portas de firewall fechadas ou senhas de banco incorretas.

---

### 2.5 — Teste de Regressão

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor Dev as Desenvolvedor
    participant Repo as Repositório Git
    participant CI as Pipeline CI/CD
    participant Suite as Suíte de Testes
    participant Service as CarrinhoService
    participant Calc as CalculadoraCarrinho

    Dev->>Repo: push (código com a nova regra de cupom)
    Repo->>CI: aciona o gatilho (webhook)
    activate CI

    CI->>Suite: executarTodosOsTestes()
    activate Suite

    Note over Suite,Calc: Execução do Teste Antigo (Regra original de 15%)
    
    Suite->>Service: executarTesteDescontoVIPOriginal()
    activate Service
    
    Service->>Calc: calcularDesconto(perfilVIP, semCupom)
    activate Calc
    
    Calc-->>Service: Valor inesperado (Nova regra quebrou a lógica antiga)
    deactivate Calc
    
    Service-->>Suite: Asserção falhou (AssertionError)
    deactivate Service
    
    Note over Suite,CI: Defeito de Regressão Detectado!

    Suite-->>CI: Resultado: FAILED
    deactivate Suite
    
    CI-->>Dev: Alerta de Falha: O build quebrou!
    deactivate CI
```

#### 2. Explicação Textual do Cenário
**Contexto:** Um novo recurso (uma regra de cupom de desconto, que passa a ser um parâmetro adicional de `CalculadoraCarrinho`) foi adicionado ao projeto e pode ter quebrado o que já funcionava no cálculo do carrinho.

**Como a abordagem é aplicada:** Um servidor de integração contínua (CI/CD) agrupa todos os testes criados anteriormente em uma Suíte de Testes e os reexecuta automaticamente ao menor sinal de alteração no código-fonte, inclusive `executarTesteDescontoVIPOriginal()`, que chama `calcularDescontoProgressivo()` exatamente como nas seções anteriores, sem informar cupom, para confirmar que o comportamento antigo continua igual.

**Objetivo e Defeitos Revelados:** Revela efeitos colaterais indesejados onde código novo corrompe funcionalidades antigas.

---

## 3. Teste de Validação (Validation Testing)

### 3.1 - Critérios de Aceitação 

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor PO as Product Owner / Cliente
    participant UI as Interface Web (Front-end)
    participant API as API do E-commerce
    participant BD as Banco de Dados

    Note over PO,BD: Execução do Teste de Aceitação (UAT) - Regras de Negócio

    PO->>UI: Tenta adicionar 11 itens (O limite de negócio é 10)
    activate UI
    
    UI->>API: POST /carrinho/adicionar (quantidade: 11)
    activate API
    
    API->>BD: consultarRegrasEEstoque(produtoId)
    activate BD
    BD-->>API: Dados retornados
    deactivate BD
    
    Note over API: API verifica que 11 excede o critério de aceitação
    
    API-->>UI: Erro 422 (Unprocessable Entity)
    deactivate API
    
    UI-->>PO: Exibe alerta visual: "Limite máximo de 10 itens atingido."
    deactivate UI
    
    Note over PO,UI: PO valida o cenário: O sistema bloqueou a ação corretamente, cumprindo o critério de aceitação.
```
#### 2. Explicação Textual do Cenário
**Contexto:** Nesse teste, busca-se validar se o sistema realmente atende às necessidades e expectativas do usuário e aos requisitos definidos para o sistema. Considerando a quantidade de itens que o usúario deseja adicionar ao carrinho.

**Como a abordagem é aplicada:** Considerando que o usuário esteja logado e tente adicionar 11 itens ao carrinho, o sistema deverá consultar e aplicar a regra de negócio que estabelece o limite máximo de 10 itens, por meio da função `consultarRegrasEEstoque(produtoId)`. Como resultado, a ação deverá ser bloqueada, retornando o status **422 — Unprocessable Entity**, e o sistema deverá exibir uma mensagem de alerta ao usuário informando que o limite máximo de itens do carrinho foi atingido.

**Objetivo e Defeitos Revelados:** Validar que o sistema respeita uma regra de negócio definida pelo cliente, do ponto de vista do usuário final — não da implementação interna. Revela regras de negócio mal implementadas ou ausentes (ex.: o limite de 10 itens não sendo verificado) e mensagens de erro pouco claras para o usuário.

### 3.2 Teste Alfa

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor QA as Voluntário Interno / QA
    participant UI as Interface Web (Staging)
    participant API as API Shop20
    participant BD as Banco de Dados
    participant Tracker as Bug Tracker (Jira)

    Note over QA, Tracker: Sessão de Teste Alfa (Exploratório em Staging)

    QA->>UI: Realiza fluxo atípico (ex: duplo clique em "Pagar")
    activate UI
    
    UI->>API: POST /carrinho/checkout (Chamada 1)
    UI->>API: POST /carrinho/checkout (Chamada 2 simultânea)
    activate API
    
    API->>BD: debitarEstoque()
    activate BD
    BD-->>API: Conflito de concorrência (Race Condition)
    deactivate BD
    
    API-->>UI: Retorna Erro 500 (Internal Server Error)
    deactivate API
    
    UI-->>QA: Exibe código de erro técnico quebrando a tela (Falha)
    deactivate UI
    
    Note over QA, UI: Comportamento inesperado detectado pelo testador
    
    QA->>Tracker: Registrar Bug: "Duplo clique no checkout exibe Erro 500"
    activate Tracker
    Tracker-->>QA: Ticket [SHOP-992] criado para a equipe de desenvolvimento
    deactivate Tracker
```
#### 2. Explicação Textual do Cenário
**Contexto:** O Teste Alfa ocorre quando a nova versão do Shop20 está finalizada do ponto de vista do desenvolvimento, mas ainda não é segura o suficiente para ser liberada aos clientes reais. Ele é realizado no ambiente de Staging (Homologação), que atua como uma réplica exata do ambiente de Produção. Para simular a experiência real, o teste é conduzido pela Equipe de QA que orquestra e coleta as métricas em conjunto com funcionários voluntários de outros setores (como RH e Atendimento).

**Como a abordagem é aplicada:** Como o objetivo é simular o uso real do produto, a abordagem é de teste exploratório e livre. Por exemplo, por impaciência ou erro humano, um voluntário pode realizar um duplo clique no botão "Pagar". Esse comportamento imprevisto dispara duas requisições simultâneas de DebitarEstoque. O banco de dados acusa um erro de concorrência, retornando o status 500 (Internal Server Error) e exibindo uma mensagem técnica que quebra a tela do usuário.

**Objetivos e Defeitos revelados** Identificar comportamentos inesperados, problemas de usabilidade e bugs não mapeados (como condições de corrida ou erros não tratados na interface) antes que o sistema seja exposto a clientes externos, atuando como um filtro de qualidade de negócio.

### 3.3 Teste Beta 

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor Cliente as Usuário Beta (Cliente Real)
    participant App as App Shop20 (Produção)
    participant API as API de Produção
    participant Telemetria as Monitoramento Passivo (Sentry)
    participant Feedback as Canal de Feedback Ativo

    Note over Cliente, Feedback: Teste Beta: Ocorre no mundo real, em seus próprios dispositivos, sem supervisão da equipe de QA.

    Cliente->>App: Acessa novo Checkout (Feature Flag ativada)
    activate App
    
    App->>API: POST /pagamento/processar
    activate API
    
    Note over Cliente, API: Simulação de problema real (Ex: Queda de sinal 4G do cliente)
    
    API-->>App: Timeout (Tempo de requisição esgotado)
    deactivate API
    
    App-->>Cliente: Botão de "Confirmar" trava carregando
    
    Note over App, Telemetria: Coleta Passiva (Invisível ao usuário)
    App->>Telemetria: Envia log de timeout silenciosamente
    
    Note over Cliente, Feedback: Coleta Ativa (Interação direta do usuário)
    Cliente->>App: Balança o celular (Shake to Report)
    App->>Feedback: Abre tela de reporte com print automático
    activate Feedback
    
    Cliente->>Feedback: Digita: "O botão de Pix travou na rua" e envia
    Feedback-->>Cliente: "Obrigado! Seu feedback ajuda o Shop20 a melhorar."
    deactivate Feedback
    
    deactivate App
```
#### 2. Explicação Textual do Cenário
**Contexto:** A nova versão do sistema, como o novo Checkout Unificado do Marketplace, já passou pelo Teste Alfa e foi implantada no ambiente de Produção. No entanto, ela ainda não é disponibilizada para todos os usuários. Para controlar essa liberação, utilizamos a técnica de Feature Flag (chave de ativação). O novo código já está presente no sistema, mas a funcionalidade é ativada inicialmente para uma pequena parcela dos clientes, por exemplo, 5% dos clientes mais ativos ou membros VIP do Shop20. Dessa forma, o teste é realizado por clientes reais, utilizando dispositivos, condições de rede e dados reais, diferentemente do Teste Alfa, que ocorre em um ambiente controlado.

**Como a abordagem é aplicada** O Cliente acessa o novo Checkout pelo App Shop20, que está liberado por uma Feature Flag. Ao realizar o pagamento, o App envia uma requisição para a API de Produção. Durante o processo, ocorre uma queda de conexão, causando um Timeout. O problema é registrado automaticamente pelo Monitoramento Passivo (Sentry). Além disso, o Cliente pode utilizar o Canal de Feedback Ativo para relatar o problema, permitindo que a equipe identifique falhas que acontecem em condições reais de uso. 

**Objetivo e Defeitos Revelados:** Validar o comportamento do software no mundo real. Revela com facilidade defeitos de fragmentação de dispositivos (compatibilidade com celulares antigos), falhas em redes instáveis e comportamentos de uso que a equipe interna jamais conseguiria prever.

## 4. Teste de Sistema

### 4.1 — Teste de Recuperação

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor Cliente as Aplicação Cliente
    participant Controller as CarrinhoController
    participant Master as CarrinhoRepository (nó primário)
    participant Chaos as Simulador de Falha
    participant Sentinel as Failover (Sentinel)
    participant Slave as CarrinhoRepository (nó réplica)

    Cliente->>Controller: POST /carrinho/adicionar
    activate Controller
    
    Controller->>Master: salvarItem(usuarioId, produtoId, quantidade)
    activate Master
    
    Note over Master, Chaos: Simulador injeta falha crítica
    Chaos--xMaster: Mata o processo abruptamente (CRASH)
    deactivate Master
    
    Controller--xController: Erro: Connection Timeout
    
    Note over Sentinel, Slave: Mecanismo de Recuperação Automática
    Sentinel->>Master: ping()
    Sentinel-->>Sentinel: Detecta nó inoperante (Fail)
    Sentinel->>Slave: promoverParaMaster()
    activate Slave
    Slave-->>Sentinel: Status: Novo Master Ativo
    
    Note over Controller, Slave: Controller implementa padrão de tolerância a falhas (Retry)
    Controller->>Sentinel: ondeEstaOMaster()?
    Sentinel-->>Controller: Retorna endereço do Novo Master (Antigo Slave)
    
    Controller->>Slave: tentarNovamente: salvarItem(usuarioId, produtoId, quantidade)
    Slave-->>Controller: 200 OK (Item persistido)
    
    Controller-->>Cliente: 200 OK (Sucesso transparente)
    deactivate Controller
    deactivate Slave
```

#### 2. Explicação Textual do Cenário
**Contexto:** No Shop20, o `CarrinhoRepository` é implementado sobre um cluster de banco de dados com nó primário e nó réplica, identificados como `CarrinhoRepository (nó primário)` e `CarrinhoRepository (nó réplica)`. Durante um pico de acessos, como a Black Friday, o nó primário pode sofrer uma falha crítica. Para garantir alta disponibilidade, o nó réplica pode assumir seu lugar por meio do `Failover (Sentinel)`. O objetivo do Teste de Recuperação é verificar se o Shop20 consegue continuar funcionando após essa falha sem intervenção humana.

**Como a abordagem é aplicada:** Durante o teste, o Simulador de Falha provoca propositalmente um CRASH no `CarrinhoRepository (nó primário)` enquanto o `CarrinhoController` está chamando `salvarItem()`. O `CarrinhoController` recebe um `Connection Timeout`. Em seguida, o `Failover (Sentinel)` verifica o nó primário, identifica que está inoperante e promove o `CarrinhoRepository (nó réplica)` a novo primário. O `CarrinhoController` consulta o Sentinel para descobrir o novo endereço e refaz a chamada `salvarItem()`, desta vez contra o nó réplica promovido.

**Objetivo e Defeitos Revelados:** Validar se o Shop20 consegue se recuperar automaticamente após a falha do banco principal, mantendo a operação do usuário de adicionar item ao carrinho. Revela problemas como perda de dados no `CarrinhoRepository`, falhas no processo de Failover, indisponibilidade prolongada e erros no mecanismo de Retry.

---

### 4.2 — Teste de Segurança

#### 1. Diagrama de Sequência UML
```mermaid
sequenceDiagram
    autonumber
    actor Atacante as Atacante (Usuário Malicioso)
    participant Controller as CarrinhoController
    participant Auth as ServicoAutenticacao (JWT)
    participant Service as CarrinhoService
    participant Repo as CarrinhoRepository

    Note over Atacante, Controller: Ataque combinado: IDOR (Tentando acessar o Carrinho da Vítima) e Preço Adulterado (R$ 0,01)

    Atacante->>Controller: POST /api/carrinho/{vitimaId}/item (Preço: 0.01)
    activate Controller
    
    Controller->>Auth: extrairUsuarioDoToken(Header JWT)
    activate Auth
    Auth-->>Controller: usuarioId = AtacanteID
    deactivate Auth
    
    Note over Controller: Validação 1: O dono do Token (Atacante) é o dono do Carrinho ({vitimaId})?
    
    alt Acesso Negado (Bloqueio de IDOR)
        Controller-->>Atacante: 403 Forbidden (Acesso não autorizado)
    else Carrinho pertence ao Atacante (Passou no IDOR)
        Controller->>Service: adicionarAoCarrinho(usuarioId, produtoId, precoAdulterado)
        activate Service
        
        Service->>Repo: consultarPrecoReal(produtoId)
        activate Repo
        Repo-->>Service: precoReal = R$ 5000.00
        deactivate Repo
        
        Note over Service: Validação 2: Preço do Payload (0.01) == Preço do CarrinhoRepository (5000.00)?
        
        Service-->>Controller: ExcecaoSeguranca: Divergência de preço detectada!
        deactivate Service
        
        Controller-->>Atacante: 400 Bad Request (Requisição manipulada)
    end
    deactivate Controller
```

#### 2. Explicação Textual do Cenário
**Contexto:** No Shop20, o Teste de Segurança verifica se o sistema consegue impedir que um usuário mal-intencionado manipule informações do checkout. Nesse cenário, o `Atacante (Usuário Malicioso)` altera a requisição para tentar acessar o carrinho de outra pessoa por meio de um ataque IDOR e também modifica o preço do produto para R$ 0,01, caracterizando Parameter Tampering.

**Como a abordagem é aplicada:** O `Atacante (Usuário Malicioso)` envia uma requisição para o `CarrinhoController` com o `vitimaId` e o preço adulterado. O Controller consulta o `ServicoAutenticacao (JWT)` para identificar o usuário pelo Token. Caso o usuário tente acessar o carrinho de outra pessoa, o sistema bloqueia a requisição com 403 Forbidden. Caso o carrinho pertença ao próprio atacante, a requisição segue para o `CarrinhoService`, que consulta o preço verdadeiro no `CarrinhoRepository` por meio de `consultarPrecoReal(produtoId)`. Ao identificar que o preço enviado é diferente do preço armazenado, o sistema bloqueia a operação e retorna 400 Bad Request.

**Objetivo e Defeitos Revelados:** Validar se o Shop20 protege os dados do usuário e não confia em informações manipuladas pelo cliente. Revela falhas de controle de acesso (IDOR) e problemas de manipulação de parâmetros, como permitir que o preço enviado pelo front-end seja utilizado diretamente no processamento da compra.

---

### 4.3 — Teste de Estresse (Stress Testing)

#### 1. Diagrama de Arquitetura

```mermaid
flowchart TD
    %% Nós do Gerador de Carga
    subgraph Geradores de Carga [Nuvem de Injeção de Carga - k6/JMeter]
        G1((Nó 1))
        G2((Nó 2))
        G3((Nó 3))
    end

    %% Componentes do Sistema
    LB{Load Balancer}
    
    subgraph Cluster API [Cluster Shop20 - Kubernetes]
        API1(CarrinhoController - Pod 1)
        API2(CarrinhoController - Pod 2)
        API3(CarrinhoController - Pod 3<br/>! CPU 100% !)
    end

    Cache[(Cache de Sessão<br/>Redis)]
    DB[(CarrinhoRepository<br/>! Fila de Conexões Cheia !)]
    
    %% Observabilidade
    Monitor[[Ferramenta de Observabilidade<br/>Grafana / Datadog]]
    
    %% Alerta de Ruptura
    Alerta>PONTO DE RUPTURA:<br/>Tempos de resposta > 10s<br/>Taxa de Erro 503 > 60%]

    %% Relações de Tráfego
    G1 & G2 & G3 == "Injeção de 15.000 RPS<br/>(Simulação Black Friday)" === LB
    
    LB --> API1
    LB --> API2
    LB --> API3

    API1 & API2 & API3 --> Cache
    API1 & API2 & API3 -- "Tentativa de escrita (Gargalo)" --> DB

    %% Relações de Monitoramento
    API1 & API2 & API3 -. "Métricas: Latência, CPU, RAM" .-> Monitor
    DB -. "Alerta: Esgotamento de Pool de Conexões" .-> Monitor
    
    Monitor -.-> Alerta
```

#### 2. Explicação Textual do Cenário
**Contexto:** No Shop20, o Teste de Estresse verifica o comportamento e os limites da arquitetura sob condições extremas de tráfego, simulando um evento de pico como a Black Friday. Nesse cenário, o objetivo não é confirmar se o sistema funciona em condições normais, mas sim submetê-lo a uma carga absurdamente alta (saltando de 500 para 15.000 requisições por segundo) para descobrir qual é o seu exato ponto de ruptura. Cada pod do `Cluster Shop20` roda uma instância do `CarrinhoController`, interagindo com o `CarrinhoRepository` sob extrema pressão de infraestrutura.

**Como a abordagem é aplicada:** Os `Geradores de Carga` em nuvem (utilizando ferramentas como k6 ou JMeter) disparam uma tempestade de acessos simultâneos contra o `Load Balancer`, que distribui o tráfego entre os pods do `CarrinhoController`. Durante o ataque, a `Ferramenta de Observabilidade` (Grafana ou Datadog) monitora métricas vitais como latência, CPU e memória RAM. O teste força a infraestrutura até que um componente crítico ceda — seja a CPU do `CarrinhoController` batendo 100% ou o `CarrinhoRepository` lotando sua fila de conexões —, registrando o momento em que a aplicação colapsa e começa a retornar lentidão severa (tempo de resposta > 10s) ou o erro `503 Service Unavailable`.

**Objetivo e Defeitos Revelados:** Identificar o limite máximo absoluto de capacidade do Shop20 e observar como o sistema reage à própria falha (se morre graciosamente ou corrompe dados). Revela gargalos severos de escalabilidade que não aparecem em testes normais, como esgotamento do pool de conexões do `CarrinhoRepository`, lentidão nos gatilhos de autoscaling dos pods e vazamento de memória sob pressão.

---

### 4.4 — Teste de Desempenho (Performance Testing)

#### 1. Diagrama de Sequência UML

```mermaid
sequenceDiagram
    autonumber
    actor Injetor as Ferramenta de Carga (k6/JMeter)
    participant Controller as CarrinhoController
    participant Service as CarrinhoService
    participant Repo as CarrinhoRepository
    
    Note over Injetor, Repo: Injeção de Carga Normal: 500 Usuários Virtuais simultâneos (VUs)
    
    loop Durante 30 minutos (Carga Contínua)
        Injetor->>Controller: POST /carrinho/adicionar
        activate Controller
        
        Controller->>Service: adicionarAoCarrinho(usuarioId, produtoId, quantidade)
        activate Service
        
        Service->>Repo: salvarItem(usuarioId, produtoId, quantidade)
        activate Repo
        Repo-->>Service: 200 OK (ms consumidos)
        deactivate Repo
        
        Service-->>Controller: 200 OK (ms consumidos)
        deactivate Service
        
        Controller-->>Injetor: 200 OK (Registro de Latência)
        deactivate Controller
    end
    
    Note over Injetor, Repo: META (SLA): Tempo de Resposta (Percentil 95) < 200ms | Taxa de Erro < 0.1%
```

#### 2. Explicação Textual do Cenário
**Contexto:** No Shop20, o Teste de Desempenho avalia se o sistema atende aos Acordos de Nível de Serviço (SLA) estabelecidos para o uso cotidiano. Diferente do Teste de Estresse (que busca quebrar a aplicação), este cenário simula uma carga normal e representativa — por exemplo, 500 usuários simultâneos adicionando produtos ao carrinho de forma constante, fluindo através do `CarrinhoController`, `CarrinhoService` e `CarrinhoRepository`. A meta de qualidade arquitetural é rigorosa: 95% das requisições (p95) devem ser respondidas em menos de 200 milissegundos, com uma taxa de erro máxima aceitável de 0,1%.

**Como a abordagem é aplicada:** A `Ferramenta de Carga` (como k6, Gatling ou JMeter) é estruturada para enviar um fluxo contínuo de requisições em loop durante um período prolongado. A requisição atravessa o `CarrinhoController`, passa pelo `CarrinhoService` e realiza a operação `salvarItem()` no `CarrinhoRepository`. A cada ciclo, a ferramenta de medição não verifica se a resposta está "certa" ou "errada" no sentido de negócio, mas atua como um cronômetro de precisão, agregando as latências e construindo gráficos de percentis em tempo real para avaliar se o sistema sofre degradação ao longo do tempo.

**Objetivo e Defeitos Revelados:** Avaliar a velocidade, a responsividade e a estabilidade do Shop20 nas operações normais. Revela gargalos de desempenho "silenciosos" (problemas que não derrubam o sistema, mas causam lentidão e frustração no usuário), como queries ineficientes no `CarrinhoRepository` por falta de índices, payloads JSON desnecessariamente pesados na rede ou alto tempo de processamento na camada de negócio do `CarrinhoService`.
