
----
20/03

Versão estavel, rodando em Docker Compose.

Após subir o serviço do elk, executar os comandos abaixo para criar a segmentação dos ambientes

Lifecycle por serviço gerado, mas geramos um que abrange tudo, rotacionando todo dia e com retenção de 7D

v8.15.0

### 1. Regra para os Traces (Transações das rotas):
```bash
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

### 2. Regra para as Métricas (como esse log que você me mandou):
```bash
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

### 3. Regra para os Logs de Erro (sua rota /error):
```bash
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
### 4. Para deixar o seu ambiente 100% automatizado no Ubuntu 24.04 com Docker, a estratégia correta não é criar um Lifecycle (ILM) por serviço, mas sim usar Component Templates e Index Templates que se aplicam dinamicamente a qualquer novo serviço que apareça.

#### No Elastic 8.15, o segredo é: O Template dita a regra, o Pipeline faz o roteamento.

### 4.1 Criar uma Política de Lifecycle (ILM) Única
#### Em vez de uma por serviço, crie uma política padrão (ex: apm-7-days-delete) que define que os dados devem ser apagados após 7 dias.

```bash
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

### 4.2 Criar um Component Template para o Lifecycle
#### Isso "empacota" a configuração do ILM para que ela possa ser injetada em qualquer índice novo.

```bash
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

### 4.3 Criar o Index Template Global (A "Mágica" da Automação)
#### Este template diz ao Elasticsearch: "Sempre que um rastro (trace) chegar para qualquer namespace (gw, auth, orders, etc), aplique as configurações do apm-custom-settings".

```bash
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
#### Repita o comando acima mudando apenas o **index_patterns** para **metrics-apm-*** e **logs-apm-*** se quiser automatizar tudo.