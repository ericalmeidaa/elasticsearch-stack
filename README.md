# 🚀 Setup ELK Stack

Repositório dedicado à configuração e automação da stack Elasticsearch,  Kibana e APM-Server via Docker.

Segmenta automaticamente os projetos dos DEV no **Index Management**.

Baixar o arquivo **docker-compose.yml** no servidor e executar !!

---
 🚀 Log de Atualizações & Status

- Última Atualização: 20/03 ✅
- Ubuntu: 24.04 Lts
- Docker Compose: v29.0 🐳
- Versão da Stack: v8.15.0 ⚡
---

🛠️ Configuração de Segmentação de Ambientes

Após subir o serviço do ELK, execute as regras de Ingest Pipeline abaixo. Elas garantem que os dados de diferentes serviços sejam segmentados corretamente por **namespace** ou **service.name**.

💡 Criado um fluxo que abrange tudo, com rotação diária e retenção de 7 dias (7D). Prático e eficiente! 🔄

**Executar os comandos no Kibana Dev Tools.**

---

### 🛣️ 1. Regra para os Traces (Transações das rotas)

```yaml
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

```yaml
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

```yaml
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

### 🤖 4. Automatizar a Rotação dos Projetos
Para um ambiente 100% automatizado, não é necessário criar um ILM por serviço. Vamos usar o Component Templates e Index Templates dinâmicos! 🪄

No Elastic 8.15 o Template dita a regra, o Pipeline faz o roteamento. 🦾

#### ⏳ 4.1 Criar uma Política de Lifecycle (ILM) Única
Crie uma política padrão (ex: apm-7-days-delete) para limpeza automática após uma semana.

```yaml
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

```yaml
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

```yaml
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

📝 Nota: Para automatizar tudo, repita o passo 4.3 alterando o **index_patterns** para **metrics-apm-\*** e **logs-apm-\***. 🏁