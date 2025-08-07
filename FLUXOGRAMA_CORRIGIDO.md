# 🎯 Fluxograma Visual Corrigido - Sistema de Ligações

## 📊 Fluxo Principal da API - Visão Geral

```mermaid
flowchart TD
    Start([🚀 POST create_call]) --> ValidateParams{🔍 Validação<br/>Parâmetros}
    ValidateParams -->|❌ Erro| Error400[⚠️ Retorna 400<br/>Bad Request]
    ValidateParams -->|✅ OK| ParseCall{📞 Tipo de Call?}
    
    ParseCall -->|Teams| TeamsFlow[🟦 Fluxo Teams]
    ParseCall -->|Whatsapp| WhatsFlow[🟢 Fluxo WhatsApp]
    ParseCall -->|Ambos| BothFlow[🟣 Executa Ambos<br/>em Paralelo]
    
    TeamsFlow --> EmailCheck{📧 Email é Array?}
    EmailCheck -->|Sim| MultiCall[🔄 Múltiplas Ligações]
    EmailCheck -->|Não| EscalationCheck{⬆️ Escalation?}
    
    EscalationCheck -->|Não| SingleCall[📱 Ligação Simples]
    EscalationCheck -->|Sim| EscalationFlow[📈 Ligação com<br/>Escalonamento]
    
    MultiCall --> ThreadPool[🧵 ThreadPoolExecutor<br/>max workers 5]
    ThreadPool --> ConsolidateResults[📋 Consolida<br/>Resultados]
    
    SingleCall --> CreateTeamsCall[🔗 Cria Chamada<br/>Microsoft Graph]
    EscalationFlow --> CreateTeamsCall
    
    WhatsFlow --> CallWhatsAPI[📲 Chama API<br/>WhatsApp]
    
    BothFlow --> TeamsFlow
    BothFlow --> WhatsFlow
    
    CreateTeamsCall --> WaitEstablish[⏰ Aguarda<br/>Estabelecer]
    WaitEstablish --> PlayAudio[🔊 Reproduz<br/>Áudio TTS]
    PlayAudio --> EndCall[📴 Encerra<br/>Chamada]
    
    CallWhatsAPI --> WhatsResult[📊 Resultado<br/>WhatsApp]
    
    ConsolidateResults --> FinalResponse[📤 Resposta Final]
    EndCall --> FinalResponse
    WhatsResult --> FinalResponse
    
    FinalResponse --> Success[✅ 200 Success]
    FinalResponse --> Failed[❌ 500 Failed]
    
    Error400 --> End([🏁 Fim])
    Success --> End
    Failed --> End
    
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    class Start,End startEnd
    class TeamsFlow,WhatsFlow,BothFlow,MultiCall,SingleCall,EscalationFlow process
    class ValidateParams,ParseCall,EmailCheck,EscalationCheck decision
    class Success success
    class Error400,Failed error
```

## 🔄 Fluxo Detalhado de Escalonamento

```mermaid
flowchart TD
    StartEsc([🎯 Início Escalonamento]) --> PrimaryCall[📞 Chamada Principal<br/>para Email Primário]
    PrimaryCall --> CheckAnswer{📋 Atendeu?}
    CheckAnswer -->|✅ Sim| SuccessEnd[🎉 FIM Sucesso]
    CheckAnswer -->|❌ Não| CheckEscalation{⬆️ Escalation<br/>Ativo?}
    
    CheckEscalation -->|❌ Não| FailEnd[❌ FIM Falha]
    CheckEscalation -->|✅ Sim| GetTurno[🕒 Determina<br/>Turno Atual]
    
    GetTurno --> QueryDB[(🗄️ Consulta BD<br/>Data Turno Área)]
    QueryDB --> FilterStatus[🔍 Filtra<br/>Disponivel TR]
    FilterStatus --> RemoveOriginal[🚫 Remove Usuário<br/>Original da Lista]
    
    RemoveOriginal --> CheckEmpty{📝 Lista Vazia?}
    CheckEmpty -->|✅ Sim| NoUsersEnd[❌ FIM Nenhum<br/>Usuário Disponível]
    CheckEmpty -->|❌ Não| NextUser[👤 Próximo da Lista]
    
    NextUser --> TryCall[📞 Tenta Ligação]
    TryCall --> CheckCallAnswer{📋 Atendeu?}
    CheckCallAnswer -->|✅ Sim| EscSuccessEnd[🎉 FIM Sucesso<br/>Escalonamento]
    CheckCallAnswer -->|❌ Não| MoreUsers{👥 Mais Usuários<br/>na Lista?}
    
    MoreUsers -->|✅ Sim| NextUser
    MoreUsers -->|❌ Não| NobodyEnd[❌ FIM Ninguém<br/>Atendeu]
    
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef database fill:#e0f2f1,stroke:#00695c,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:3px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:3px
    
    class StartEsc startEnd
    class PrimaryCall,GetTurno,FilterStatus,RemoveOriginal,NextUser,TryCall process
    class CheckAnswer,CheckEscalation,CheckEmpty,CheckCallAnswer,MoreUsers decision
    class QueryDB database
    class SuccessEnd,EscSuccessEnd success
    class FailEnd,NoUsersEnd,NobodyEnd error
```

## 🧵 Fluxo de Múltiplas Ligações Paralelas

```mermaid
flowchart TD
    StartMulti([🎯 Múltiplas Ligações]) --> EmailArray[📧 Lista de Emails<br/>Lista com todos emails]
    EmailArray --> CreateExecutor[🧵 ThreadPoolExecutor<br/>max workers 5]
    
    CreateExecutor --> SubmitTasks[📤 Submete Tasks<br/>para cada email]
    SubmitTasks --> Thread1[🧵 Thread 1<br/>process single call email1]
    SubmitTasks --> Thread2[🧵 Thread 2<br/>process single call email2]
    SubmitTasks --> Thread3[🧵 Thread 3<br/>process single call email3]
    SubmitTasks --> ThreadN[🧵 Thread N<br/>process single call emailN]
    
    Thread1 --> Flow1[🔄 Fluxo Completo<br/>make teams call with retries]
    Thread2 --> Flow2[🔄 Fluxo Completo<br/>make teams call with retries]
    Thread3 --> Flow3[🔄 Fluxo Completo<br/>make teams call with retries]
    ThreadN --> FlowN[🔄 Fluxo Completo<br/>make teams call with retries]
    
    Flow1 --> Result1[📊 Resultado 1<br/>email status answered]
    Flow2 --> Result2[📊 Resultado 2<br/>email status answered]
    Flow3 --> Result3[📊 Resultado 3<br/>email status answered]
    FlowN --> ResultN[📊 Resultado N<br/>email status answered]
    
    Result1 --> WaitAll[⏰ as completed<br/>Aguarda Todas as Threads]
    Result2 --> WaitAll
    Result3 --> WaitAll
    ResultN --> WaitAll
    
    WaitAll --> Consolidate[📋 Consolida Resultados<br/>successful calls lista]
    Consolidate --> CheckAny{🔍 Alguma<br/>Atendeu?}
    
    CheckAny -->|✅ Sim| GlobalSuccess[🎉 Sucesso Global<br/>status success]
    CheckAny -->|❌ Não| GlobalFail[❌ Falha Global<br/>status failed]
    
    GlobalSuccess --> FinalResult[📤 Resposta Final<br/>total calls successful calls answered by]
    GlobalFail --> FinalResult
    
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef thread fill:#e8eaf6,stroke:#3f51b5,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    class StartMulti,FinalResult startEnd
    class EmailArray,CreateExecutor,SubmitTasks,Consolidate process
    class Thread1,Thread2,Thread3,ThreadN,Flow1,Flow2,Flow3,FlowN,Result1,Result2,Result3,ResultN,WaitAll thread
    class CheckAny decision
    class GlobalSuccess success
    class GlobalFail error
```

## 🎵 Fluxo TTS e Chamada Teams Detalhado

```mermaid
flowchart TD
    StartTTS([🎯 Início Ligação Teams]) --> GenerateAudio[🎤 Gera Áudio TTS<br/>Google Cloud]
    GenerateAudio --> CalcDuration[⏱️ Calcula Duração<br/>frames dividido rate]
    CalcDuration --> UploadGCS[☁️ Upload para<br/>Google Cloud Storage]
    UploadGCS --> SignedURL[🔗 Gera URL Assinada<br/>válida por 1 hora]
    
    SignedURL --> GetToken[🔑 Obtém Token<br/>Azure OAuth2]
    GetToken --> RetryLoop{🔄 Loop de Tentativas<br/>attempt menor que retries}
    
    RetryLoop -->|Sim| GetUserId[👤 Busca User ID<br/>Microsoft Graph]
    RetryLoop -->|Não| MaxRetries[❌ Máximo de<br/>Tentativas Atingido]
    
    GetUserId --> CreateCall[📞 Cria Chamada<br/>Microsoft Graph API]
    CreateCall --> WaitEstablish[⏰ Aguarda Estabelecer<br/>max 20 tentativas]
    
    WaitEstablish --> CheckEstablished{📋 Estabelecida?}
    CheckEstablished -->|❌ Não| TryAgain[⏭️ Próxima Tentativa<br/>sleep 10 segundos]
    CheckEstablished -->|✅ Sim| PlayPrompt[🔊 Reproduz Áudio<br/>playPrompt]
    
    PlayPrompt --> WaitAudio[⏰ Aguarda Áudio<br/>sleep duration mais 5]
    WaitAudio --> EndCall[📴 Encerra Chamada<br/>DELETE call]
    
    EndCall --> SuccessResult[✅ Sucesso<br/>callId audioUrl answered true]
    
    TryAgain --> RetryLoop
    MaxRetries --> FailResult[❌ Falha<br/>status failed answered false]
    
    CreateCall -.->|Erro| HandleError[⚠️ Trata Erro<br/>Encerra chamada se criada]
    HandleError --> TryAgain
    
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    classDef gcp fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef azure fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:3px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:3px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    
    class StartTTS startEnd
    class GenerateAudio,CalcDuration,UploadGCS,SignedURL gcp
    class GetToken,GetUserId,CreateCall,PlayPrompt,EndCall azure
    class RetryLoop,CheckEstablished decision
    class SuccessResult success
    class MaxRetries,FailResult,HandleError error
    class WaitEstablish,WaitAudio,TryAgain process
```

## 🏗️ Arquitetura do Sistema

```mermaid
graph TB
    subgraph "🌐 Cliente"
        Client[📱 Aplicação Cliente<br/>curl Postman App]
    end
    
    subgraph "🔒 SSL HTTPS"
        SSL[🔐 Certificados SSL<br/>cer key]
    end
    
    subgraph "🐍 Servidor Flask"
        Flask[🌶️ Flask App<br/>Port 7000]
        Talisman[🛡️ Talisman<br/>Security Headers]
        ProxyFix[🔧 ProxyFix<br/>X-Forwarded Headers]
    end
    
    subgraph "☁️ Google Cloud"
        TTS[🎤 Text-to-Speech API<br/>pt-BR-Chirp3-HD-Orus]
        Storage[📦 Cloud Storage<br/>Bucket de Áudios]
    end
    
    subgraph "🔵 Microsoft Azure"
        AzureAD[🔑 Azure AD<br/>OAuth2 Token]
        GraphAPI[📊 Microsoft Graph<br/>Communications API]
        Teams[👥 Microsoft Teams<br/>Chamadas]
    end
    
    subgraph "📱 WhatsApp"
        WhatsAPI[📲 WhatsApp API<br/><IP>:7000]
        Zenvia[🌐 Zenvia Provider<br/>WhatsApp Gateway]
    end
    
    subgraph "🗄️ Database"
        PostgreSQL[(🐘 PostgreSQL<br/>escalas atividade)]
    end
    
    Client -->|HTTPS POST| SSL
    SSL --> Flask
    Flask --> Talisman
    Flask --> ProxyFix
    
    Flask -->|TTS Request| TTS
    TTS -->|Audio Binary| Storage
    Storage -->|Signed URL| Flask
    
    Flask -->|OAuth2| AzureAD
    AzureAD -->|Bearer Token| Flask
    Flask -->|Create Call| GraphAPI
    GraphAPI -->|Call Control| Teams
    
    Flask -->|HTTP POST| WhatsAPI
    WhatsAPI --> Zenvia
    
    Flask -->|SQL Query| PostgreSQL
    PostgreSQL -->|User List| Flask
    
    classDef client fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef security fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef server fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef gcp fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef azure fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef whats fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef database fill:#fff3e0,stroke:#e65100,stroke-width:2px
    
    class Client client
    class SSL,Talisman,ProxyFix security
    class Flask server
    class TTS,Storage gcp
    class AzureAD,GraphAPI,Teams azure
    class WhatsAPI,Zenvia whats
    class PostgreSQL database
```

## 📊 Estados da Chamada Teams

```mermaid
stateDiagram-v2
    [*] --> Iniciando : POST create_call
    
    Iniciando --> GerandoAudio : Validação OK
    GerandoAudio --> ObtendoToken : TTS Sucesso
    ObtendoToken --> CriandoCall : Token Obtido
    
    CriandoCall --> Estabelecendo : create_call OK
    CriandoCall --> Erro : Falha na criação
    
    Estabelecendo --> Conectada : wait_for_call_established
    Estabelecendo --> TentandoNovamente : Timeout 20s
    Estabelecendo --> Erro : Erro de conexão
    
    TentandoNovamente --> Estabelecendo : Retry até 5x
    TentandoNovamente --> Erro : Max retries
    
    Conectada --> ReproduzindoAudio : play_prompt
    ReproduzindoAudio --> AguardandoAudio : Audio iniciado
    AguardandoAudio --> Finalizando : Tempo do áudio
    
    Finalizando --> Finalizada : end_call
    Finalizando --> Finalizada : Erro no encerramento
    
    Erro --> [*] : Retorna falha
    Finalizada --> [*] : Retorna sucesso
    
    note right of GerandoAudio
        Google Cloud TTS
        Upload para Storage
        URL assinada 1h
    end note
    
    note right of Estabelecendo
        Loop 20x verificando
        estado established
        sleep 1 entre checks
    end note
    
    note right of AguardandoAudio
        sleep duration mais 5
        Margem de segurança
    end note
```

## 🔄 Fluxo de Retry e Recuperação

```mermaid
flowchart TD
    StartRetry([🔄 Início Retry Logic]) --> AttemptLoop{🔢 attempt menor que retries?}
    AttemptLoop -->|✅ Sim| TryCall[📞 Tentativa de Chamada]
    AttemptLoop -->|❌ Não| MaxRetries[❌ Máximo Atingido]
    
    TryCall --> CreateCall[🔗 create_call]
    CreateCall --> Success{✅ Sucesso?}
    Success -->|✅ Sim| WaitEstablish[⏰ wait_for_call_established]
    Success -->|❌ Não| HandleError[⚠️ Handle Error]
    
    WaitEstablish --> Established{📞 Estabelecida?}
    Established -->|✅ Sim| PlayAudio[🔊 play_prompt]
    Established -->|❌ Não| CleanupFailed[🧹 Cleanup Failed Call]
    
    PlayAudio --> WaitComplete[⏰ Aguarda Conclusão]
    WaitComplete --> EndCall[📴 end_call]
    EndCall --> ReturnSuccess[✅ Return Success]
    
    HandleError --> LogError[📝 Log Error]
    CleanupFailed --> LogError
    LogError --> CheckRetries{🔢 attempt menor que retries menos 1?}
    
    CheckRetries -->|✅ Sim| Sleep10[😴 sleep 10 segundos]
    CheckRetries -->|❌ Não| ReturnFailed[❌ Return Failed]
    
    Sleep10 --> IncrementAttempt[➕ attempt incrementa]
    IncrementAttempt --> AttemptLoop
    
    MaxRetries --> ReturnFailed
    ReturnSuccess --> End([🏁 Fim])
    ReturnFailed --> End
    
    classDef startEnd fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef wait fill:#e0f2f1,stroke:#00695c,stroke-width:2px
    
    class StartRetry,End startEnd
    class TryCall,CreateCall,PlayAudio,EndCall,LogError,IncrementAttempt process
    class AttemptLoop,Success,Established,CheckRetries decision
    class ReturnSuccess success
    class MaxRetries,HandleError,CleanupFailed,ReturnFailed error
    class WaitEstablish,WaitComplete,Sleep10 wait
```

---

## 🎯 Legenda dos Símbolos

| Símbolo | Significado |
|---------|-------------|
| 🚀 | Início do processo |
| 📞 | Chamada/Ligação |
| 🔍 | Validação/Verificação |
| 🧵 | Thread/Processamento paralelo |
| ⏰ | Aguardar/Temporização |
| 🗄️ | Banco de dados |
| ☁️ | Cloud/Serviços externos |
| ✅ | Sucesso |
| ❌ | Falha/Erro |
| 🔄 | Loop/Retry |
| 📊 | Resultado/Análise |
| 🏁 | Fim do processo |

## 🔧 Correções Aplicadas

1. **Removidos caracteres especiais problemáticos:**
   - `[]` em arrays → substituído por descrição textual
   - `()` em funções → removidos ou substituídos
   - `:` em objetos JSON → substituído por descrição
   - `<>` e operadores → convertidos para texto

2. **Simplificadas expressões matemáticas:**
   - `attempt < retries` → `attempt menor que retries`
   - `duration + 5` → `duration mais 5`
   - `attempt++` → `attempt incrementa`

3. **Corrigidos nomes de nós:**
   - Removidas aspas duplas desnecessárias
   - Simplificadas descrições muito longas
   - Mantidos emojis e formatação visual

Agora todos os diagramas devem renderizar corretamente! 🎨✨
