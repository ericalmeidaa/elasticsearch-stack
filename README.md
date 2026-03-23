# 🚀 Setup ELK Stack
---
 📅 20/03 - Release Notes

Status: Versão estável ✅

Infra: Rodando via Docker Compose 🐳

Versão da Stack: v8.15.0 ⚡

🛠️ Configuração de Segmentação de Ambientes
Após subir o serviço do ELK, execute os comandos abaixo para garantir a segmentação correta.

💡 Dica de Lifecycle Management: Criamos um fluxo que abrange tudo, com rotação diária e retenção de 7 dias (7D). Prático e eficiente! 🔄

---

### 🛣️ 1. Regra para os Traces (Transações das rotas)

```Bash
PUT _ingest/pipeline/traces-apm@custom
{
  "processors": [
    {
      "set": {
        "description": "Tenta usar o namespace se existir",
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "description": "Se não houver namespace (Spans), usa o Service Name",
        "if": "ctx.temp_ns == null",
        "field": "temp_ns",
        "copy_from": "service.name",
        "ignore_failure": true
      }
    },
    {
      "lowercase": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    },
    {
      "reroute": {
        "if": "ctx.temp_ns != null",
        "namespace": "{{temp_ns}}"
      }
    },
    {
      "remove": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    }
  ]
}
```
---

### 📊 2. Regra para as Métricas

```Bash
PUT _ingest/pipeline/metrics-apm.app@custom
{
  "processors": [
    {
      "set": {
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "if": "ctx.temp_ns == null",
        "field": "temp_ns",
        "copy_from": "service.name",
        "ignore_failure": true
      }
    },
    {
      "lowercase": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    },
    {
      "reroute": {
        "if": "ctx.temp_ns != null",
        "namespace": "{{temp_ns}}"
      }
    },
    {
      "remove": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    }
  ]
}
```
---

### ⚠️ 3. Regra para os Logs de Erro

```Bash
PUT _ingest/pipeline/logs-apm.error@custom
{
  "processors": [
    {
      "set": {
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "if": "ctx.temp_ns == null",
        "field": "temp_ns",
        "copy_from": "service.name",
        "ignore_failure": true
      }
    },
    {
      "lowercase": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    },
    {
      "reroute": {
        "if": "ctx.temp_ns != null",
        "namespace": "{{temp_ns}}"
      }
    },
    {
      "remove": {
        "field": "temp_ns",
        "ignore_missing": true
      }
    }
  ]
}
```
---

### 🤖 4. Automação Total (Ubuntu 24.04 + Docker)
Para um ambiente 100% automatizado, não precisamos criar um ILM por serviço. A jogada mestre aqui é usar Component Templates e Index Templates dinâmicos! 🪄

O segredo no Elastic 8.15: O Template dita a regra, o Pipeline faz o roteamento. 🦾

#### ⏳ 4.1 Criar uma Política de Lifecycle (ILM) Única
Crie uma política padrão (ex: apm-7-days-delete) para limpeza automática após uma semana.

```Bash
PUT _ilm/policy/apm-lifecycle-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### 📦 4.2 Criar um Component Template para o Lifecycle
Isso "empacota" a configuração para ser injetada em qualquer índice novo.

```Bash
PUT _component_template/apm-custom-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "apm-lifecycle-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Configuracoes base para APM: 1 shard e 0 replicas"
  }
}
```

#### ✨ 4.3 O "Pulo do Gato": Index Template Global
Sempre que um rastro (trace) chegar para qualquer namespace (gw, auth, orders, etc), o Elasticsearch aplicará as regras automaticamente.

```Bash
PUT _index_template/apm-traces-automation
{
  "index_patterns": ["traces-apm-*"],
  "data_stream": { },
  "priority": 500,
  "composed_of": ["apm-custom-settings"],
  "_meta": {
    "description": "Automacao total de Lifecycle para todos os servicos de APM Traces"
  }
}
```

📝 Nota: Para automatizar tudo, repita o passo 4.3 alterando o index_patterns para metrics-apm-* e logs-apm-*. 🏁