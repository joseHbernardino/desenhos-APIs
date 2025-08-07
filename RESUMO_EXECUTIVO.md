# Resumo Executivo - Sistema de Ligações Automatizadas

## 🎯 Visão Geral do Sistema

O sistema é uma **API REST Flask** que automatiza ligações telefônicas via **Microsoft Teams** e **WhatsApp**, com recursos avançados de:

- ✅ **Text-to-Speech (TTS)** via Google Cloud
- ✅ **Múltiplas ligações simultâneas** 
- ✅ **Sistema de escalonamento inteligente**
- ✅ **Integração híbrida** (Teams + WhatsApp)
- ✅ **Retry automático** com configuração flexível

---

## 🚀 Como Iniciar o Sistema

### Pré-requisitos Obrigatórios

| Componente | Status | Descrição |
|------------|--------|-----------|
| 🔐 **Certificados SSL** | ⚠️ Necessário | `.cer` e `.key` |
| 🔑 **Azure AD App** | ⚠️ Necessário | CLIENT_ID, CLIENT_SECRET, TENANT_ID |
| ☁️ **Google Cloud** | ⚠️ Necessário | Credenciais + Bucket Storage |
| 🗄️ **PostgreSQL** | ⚠️ Necessário | Tabela `escalas_atividade` |
| 📱 **API WhatsApp** | 🔧 Opcional | Servidor em `<IP>:7000` |

### Comando de Inicialização
```bash
cd /root/bot_ligacao
python main.py
```

**Resultado:** Servidor HTTPS ativo em `https://0.0.0.0:7000`

---

## 📊 Matriz de Parâmetros vs Funcionalidades

| Parâmetro | Obrigatório | Tipo | Descrição | Impacto |
|-----------|-------------|------|-----------|---------|
| `mensagemtts` | ✅ Sempre | string | Texto para áudio | Conteúdo da ligação |
| `call` | ✅ Sempre | enum | "Teams"/"Whatsapp"/"Ambos" | Define canal(is) |
| `email` | ⚠️ Se Teams | string/array | Email(s) Teams | Destinatário(s) |
| `telefone` | ⚠️ Se WhatsApp | string | Número WhatsApp | Destinatário WhatsApp |
| `retried` | 🔧 Opcional | int (1-5) | Tentativas por pessoa | Persistência |
| `escalation` | 🔧 Opcional | boolean | Ativa escalonamento | Cobertura |
| `area` | ⚠️ Se escalation | string | Área para escalona | Filtro de equipe |

---

## 🔄 Fluxos de Execução

### 1️⃣ **Ligação Simples** (Configuração Mínima)
```json
{
  "email": "user@empresa.com",
  "mensagemtts": "Teste simples",
  "call": "Teams"
}
```
**Executa:** 1 ligação → 1 tentativa → Fim

### 2️⃣ **Ligação com Retry** 
```json
{
  "email": "user@empresa.com", 
  "mensagemtts": "Teste com retry",
  "call": "Teams",
  "retried": 3
}
```
**Executa:** 1 ligação → até 3 tentativas → Fim

### 3️⃣ **Múltiplas Ligações**
```json
{
  "email": ["user1@empresa.com", "user2@empresa.com"],
  "mensagemtts": "Teste múltiplo",
  "call": "Teams", 
  "retried": 2
}
```
**Executa:** 2 ligações paralelas → 2 tentativas cada → Consolidação

### 4️⃣ **Escalonamento Inteligente**
```json
{
  "email": "supervisor@empresa.com",
  "mensagemtts": "Incidente crítico", 
  "call": "Teams",
  "retried": 3,
  "escalation": true,
  "area": "TI"
}
```
**Executa:** 
1. Liga para supervisor (3 tentativas)
2. Se não atender → Consulta BD (área TI + turno atual)  
3. Liga para próximo da lista → Repete até alguém atender

### 5️⃣ **Híbrido Completo**
```json
{
  "email": "manager@empresa.com",
  "telefone": "+5511999999999", 
  "mensagemtts": "Emergência sistema",
  "call": "Ambos",
  "retried": 5,
  "escalation": true,
  "area": "Infraestrutura"
}
```
**Executa:** Teams (com escalonamento) + WhatsApp em paralelo

---

## 🗄️ Consultas ao Banco de Dados

### Query Principal (Escalonamento)
```sql
-- Busca equipe de plantão ATUAL
SELECT user_id, situacao, data, turno, grupo_suporte
FROM {schema}.escalas_atividade  
WHERE grupo_suporte = 'area_solicitada'
  AND data = '2025-07-06'        -- Data ATUAL
  AND turno = 'T2'               -- Turno ATUAL  
  AND situacao IN ('TR', 'Disponivel')  -- Apenas disponíveis
ORDER BY 
  CASE situacao WHEN 'Disponivel' THEN 1 WHEN 'TR' THEN 2 END,
  created_at;
```

### Determinação de Turno
- **T1:** 06:00 - 13:50 (Manhã)
- **T2:** 13:50 - 22:30 (Tarde) 
- **T3:** 22:30 - 06:00 (Noite)

**Sistema consulta APENAS:**
- ✅ Data atual
- ✅ Turno atual  
- ✅ Área específica
- ✅ Status "Disponivel" ou "TR"

---

## 📈 Cenários de Resposta

### ✅ **Sucesso Total**
```json
{
  "status": "success",
  "details": {
    "teams": {"status": "success", "answered": true},
    "whatsapp": {"status": "success"}
  }
}
```

### ⚠️ **Sucesso Parcial** (Múltiplas)
```json
{
  "status": "success", 
  "details": {
    "teams": {
      "total_calls": 5,
      "successful_calls": 3,
      "answered_by": ["user1@...", "user3@...", "user4@..."]
    }
  }
}
```

### ❌ **Falha Total**
```json
{
  "status": "failed",
  "details": {
    "teams": {
      "status": "failed",
      "error": "Ninguém na lista de escalonamento atendeu."
    }
  }
}
```

---

## ⚡ Checklist de Implementação

### 🔧 **Configuração Inicial**
- [ ] Certificados SSL configurados em `Certs/`
- [ ] Arquivo `.env` com todas as variáveis
- [ ] App registrada no Azure AD com permissões
- [ ] Google Cloud configurado (TTS + Storage)
- [ ] PostgreSQL com tabela de escalonamento
- [ ] API WhatsApp acessível (se necessário)

### 🧪 **Testes Básicos**
- [ ] Teste ligação simples Teams
- [ ] Teste ligação WhatsApp  
- [ ] Teste múltiplas ligações
- [ ] Teste escalonamento com dados reais
- [ ] Validação de logs e monitoramento

### 🛡️ **Segurança**
- [ ] HTTPS obrigatório funcionando
- [ ] Headers de segurança configurados
- [ ] Rate limiting implementado
- [ ] Validação de entrada robusta
- [ ] Logs estruturados configurados

### 📊 **Monitoramento**
- [ ] Métricas de taxa de atendimento
- [ ] Alertas de falha do sistema
- [ ] Monitoramento de custos Google Cloud
- [ ] Dashboard de uso por área/usuário

---

## 🎯 **Casos de Uso Prioritários**

| Prioridade | Cenário | Configuração |
|------------|---------|-------------|
| 🔴 **Crítico** | Incidente produção | Híbrido + Escalonamento + Retry 5x |
| 🟡 **Alto** | Reunião emergência | Múltiplas ligações Teams |  
| 🟢 **Médio** | Manutenção programada | Ligação simples + Retry 2x |
| 🔵 **Baixo** | Notificação geral | WhatsApp apenas |

---

## 📞 **Comandos de Teste Rápido**

### Teste Mínimo
```bash
curl -k -X POST https://localhost:7000/create_call \
  -H "Content-Type: application/json" \
  -d '{"email":"test@empresa.com","mensagemtts":"Teste","call":"Teams"}'
```

### Teste Completo  
```bash
curl -k -X POST https://localhost:7000/create_call \
  -H "Content-Type: application/json" \
  -d '{
    "email":"admin@empresa.com",
    "telefone":"+5511999999999", 
    "mensagemtts":"Teste completo do sistema",
    "call":"Ambos",
    "retried":3,
    "escalation":true,
    "area":"TI"
  }'
```

---

## ⚠️ **Limitações Conhecidas**

1. **Máximo 5 tentativas** por ligação individual
2. **Escalonamento incompatível** com múltiplas ligações simultâneas  
3. **Dependência externa** da API WhatsApp
4. **Rate limits** do Microsoft Graph API
5. **Custos** proporcionais ao uso Google Cloud

---

## 🔍 **Troubleshooting Rápido**

| Erro | Causa Provável | Solução |
|------|----------------|---------|
| SSL Error | Certificados ausentes | Verificar `Certs/` |
| 401 Azure | Credenciais inválidas | Validar `.env` |  
| TTS Error | Google Cloud config | Verificar service account |
| DB Error | PostgreSQL inacessível | Validar connection string |
| WhatsApp Error | API externa down | Verificar `<IP>:7000` |

---

## 📋 **Status Operacional**

| Componente | Status | Última Verificação |
|------------|--------|--------------------|
| 🌐 **API Principal** | 🟢 Operacional | Tempo real |
| ☁️ **Google Cloud TTS** | 🟢 Operacional | Dependente |
| 🔵 **Microsoft Graph** | 🟢 Operacional | Dependente |  
| 📱 **WhatsApp API** | 🟡 Verificar | Manual |
| 🗄️ **PostgreSQL** | 🟢 Operacional | Dependente |

**✅ Sistema pronto para produção com monitoramento ativo**
