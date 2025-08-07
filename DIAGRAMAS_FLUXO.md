# Diagramas de Fluxo do Sistema de Ligações

## 🔄 Fluxo Principal da API

```mermaid
graph TD
    A[POST /create_call] --> B{Validação de Parâmetros}
    B -->|Erro| C[Retorna 400]
    B -->|OK| D{Tipo de Ligação?}
    
    D -->|Teams| E[Fluxo Teams]
    D -->|WhatsApp| F[Fluxo WhatsApp] 
    D -->|Ambos| G[Executa Ambos em Paralelo]
    
    E --> H[Gera Áudio TTS]
    H --> I[Obtém Token Azure]
    I --> J{Email é Array?}
    
    J -->|Sim| K[Múltiplas Ligações]
    J -->|Não| L{Escalonamento?}
    
    L -->|Não| M[Ligação Simples]
    L -->|Sim| N[Ligação com Escalonamento]
    
    K --> O[ThreadPoolExecutor]
    O --> P[Resultado Consolidado]
    
    M --> Q[Cria Chamada Teams]
    Q --> R[Aguarda Estabelecer]
    R --> S[Reproduz Áudio]
    S --> T[Encerra Chamada]
    
    N --> Q
    T -->|Não Atendeu| U[Consulta BD Escalonamento]
    U --> V[Próximo da Lista]
    V --> Q
    T -->|Atendeu| W[Sucesso]
    
    F --> X[Chama API WhatsApp]
    X --> Y[Resultado WhatsApp]
    
    G --> E
    G --> F
    
    P --> Z[Resposta JSON]
    W --> Z
    Y --> Z
```

## 🎯 Fluxo de Escalonamento Detalhado

```mermaid
graph TD
    A[Chamada Principal] --> B{Atendeu?}
    B -->|Sim| C[FIM - Sucesso]
    B -->|Não| D{Escalonamento Ativo?}
    D -->|Não| E[FIM - Falha]
    D -->|Sim| F[Determina Turno Atual]
    
    F --> G[Consulta BD: Data + Turno + Área]
    G --> H[Filtra: Disponivel + TR]
    H --> I[Remove Usuário Original]
    I --> J{Lista Vazia?}
    
    J -->|Sim| K[FIM - Nenhum Disponível]
    J -->|Não| L[Próximo da Lista]
    
    L --> M[Tenta Ligação]
    M --> N{Atendeu?}
    N -->|Sim| O[FIM - Sucesso Escalonamento]
    N -->|Não| P{Mais Usuários?}
    
    P -->|Sim| L
    P -->|Não| Q[FIM - Ninguém Atendeu]
```

## 📱 Fluxo Múltiplas Ligações

```mermaid
graph TD
    A[Lista de Emails] --> B[ThreadPoolExecutor]
    B --> C[Thread 1: Email 1]
    B --> D[Thread 2: Email 2] 
    B --> E[Thread N: Email N]
    
    C --> F[Fluxo Completo Ligação]
    D --> G[Fluxo Completo Ligação]
    E --> H[Fluxo Completo Ligação]
    
    F --> I[Resultado 1]
    G --> J[Resultado 2]
    H --> K[Resultado N]
    
    I --> L[Aguarda Todas as Threads]
    J --> L
    K --> L
    
    L --> M[Consolida Resultados]
    M --> N{Alguma Atendeu?}
    N -->|Sim| O[Sucesso Global]
    N -->|Não| P[Falha Global]
```

## 🔐 Fluxo de Autenticação e Recursos

```mermaid
graph TD
    A[Início Ligação] --> B[Gera Áudio TTS]
    B --> C[Google Cloud Credentials]
    C --> D[Text-to-Speech API]
    D --> E[Upload para Storage]
    E --> F[URL Assinada 1h]
    
    F --> G[Obtém Token Azure]
    G --> H[Azure AD OAuth2]
    H --> I[Token Bearer]
    
    I --> J[Microsoft Graph API]
    J --> K[Cria Chamada]
    K --> L[Configura Mídia]
    L --> M[Estabelece Conexão]
    M --> N[Reproduz Áudio]
```

## 📊 Estados da Chamada Teams

```mermaid
stateDiagram-v2
    [*] --> Criando
    Criando --> Estabelecendo: create_call()
    Estabelecendo --> Estabelecida: conexão OK
    Estabelecendo --> Falhou: timeout/erro
    Estabelecida --> Reproduzindo: play_prompt()
    Reproduzindo --> Finalizando: áudio termina
    Finalizando --> Finalizada: end_call()
    Falhou --> [*]
    Finalizada --> [*]
```
