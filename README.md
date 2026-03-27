# 🚀 Setup ELK Stack

Repositório dedicado à configuração e automação da stack Elasticsearch,  Kibana e APM-Server via Docker.

Segmenta automaticamente os projetos dos DEV por por **namespace** ou **service.name** e rotaciona os logs a cada 5 dias. 

Baixar o arquivo **docker-compose.yml** no servidor e executar !!

---
 🚀 Log de Atualizações & Status

- Última Atualização: 23/03 ✅
- Ubuntu: 24.04 Lts
- Docker Compose: v29.0 🐳
- Versão da Stack: v8.15.0 ⚡
---

🛠️ Configuração de Segmentação de Ambientes

Após subir o serviço do ELK, execute as regras de Ingest Pipeline abaixo. Elas garantem que os dados de diferentes serviços sejam segmentados corretamente por **namespace** ou **service.name**.

💡 Criado um fluxo que abrange tudo, com rotação diária e retenção de 5 dias (5D). Prático e eficiente! 🔄

**Executar os comandos no Kibana Dev Tools.**

---

### 🛣️ 1. Regra para os Traces (Transações das rotas)

```yaml
PUT _ingest/pipeline/traces-apm@custom
{
  "processors": [
    {
      "set": {
        "description": "1. Tenta usar o namespace",
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "description": "2. Se temp_ns ainda estiver vazio (campo inexistente), usa o Service Name",
        "if": "ctx.temp_ns == null || ctx.temp_ns == ''",
        "field": "temp_ns",
        "copy_from": "service.name",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "description": "3. Fallback final: se service.name também falhar, joga para 'desconhecido'",
        "if": "ctx.temp_ns == null || ctx.temp_ns == ''",
        "field": "temp_ns",
        "value": "others"
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
        "description": "4. Roteia para o namespace baseado na variável temp_ns",
        "if": "ctx.temp_ns != null",
        "namespace": "{{temp_ns}}"
      }
    },
    {
      "remove": {
        "description": "5. Limpa a sujeira",
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
        "description": "1. Tenta pegar o namespace",
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "description": "2. Fallback para Service Name",
        "if": "ctx.temp_ns == null || ctx.temp_ns == ''",
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
        "description": "1. Tenta pegar o namespace",
        "field": "temp_ns",
        "copy_from": "service.namespace",
        "ignore_failure": true
      }
    },
    {
      "set": {
        "description": "2. Fallback para Service Name",
        "if": "ctx.temp_ns == null || ctx.temp_ns == ''",
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
Para um ambiente 100% automatizado, não é necessário criar um ILM por serviço. Vamos usar a política de Lifecycle (ILM) e injetá-la diretamente nos **Component Templates** @custom nativos do Elastic APM.

Isso garante que nossas regras de retenção e shards sejam aplicadas **sem quebrar os mapeamentos (mappings ECS) originais** do APM (que são essenciais para a UI do Kibana renderizar os Spans e Waterfalls corretamente). 🦾

#### ⏳ 4.1 Criar uma Política de Lifecycle (ILM) Única
Crie uma política padrão (ex: apm-5-days-delete) para limpeza automática após uma semana.

```yaml
PUT _ilm/policy/apm-lifecycle-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d"
          }
        }
      },
      "delete": {
        "min_age": "5d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### ✨ 4.2 O "Pulo do Gato": Customizar os Templates Nativos do APM
O Elastic APM já possui templates base (ex: traces-apm@custom). Nós só precisamos editá-los para incluir nossa política de ILM e definir a quantidade de Shards/Replicas. Assim que os dados forem roteados pelos nossos Ingest Pipelines, eles herdarão essas configurações automaticamente!

#### Para os Traces do APM:
```yaml
PUT _component_template/traces-apm@custom
{
  "template": {
    "settings": {
      "index.lifecycle.name": "apm-lifecycle-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Customizacoes de ILM e Shards para Traces preservando o ECS mapping"
  }
}
```

#### Para as Métricas do APM:
```yaml
PUT _component_template/metrics-apm.app@custom
{
  "template": {
    "settings": {
      "index.lifecycle.name": "apm-lifecycle-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    }
  }
}
```

#### Para os Logs de Erro do APM:
```yaml
PUT _component_template/logs-apm.error@custom
{
  "template": {
    "settings": {
      "index.lifecycle.name": "apm-lifecycle-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    }
  }
}
```

#### Para os Logs de Erro Geral:
```yaml
PUT _component_template/logs@custom
{
  "template": {
    "settings": {
      "index.lifecycle.name": "apm-lifecycle-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Aplica ILM de 5 dias e 0 replicas para todos os logs de aplicacao gerados nativamente"
  }
}
```
🏁 Pronto! Agora a stack recebe os dados, o Pipeline faz o roteamento segmentado, os mapeamentos nativos do APM montam as telas no Kibana e os Component Templates cuidam da limpeza de 5 dias. Tudo no piloto automático!

## Boas Práticas para o Cluster pós instalação

Como esses são índices de métricas internas do sistema, podemos dizer ao Elasticsearch que não queremos réplicas para eles (ou que eles podem ser Green mesmo sem réplica).

Rode este comando no Dev Tools para "esverdear" tudo:
```yml
PUT .ds-metrics-apm.*-default-*/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```
---

Para descobrir exatamente o que o "motor" do ILM está fazendo com esse índice neste exato milissegundo (se está aguardando o tempo, se está em processo de rotação ou se deu algum erro), existe um comando de diagnóstico chamado **ILM Explain**.

Vá no seu Kibana > Dev Tools e rode isto:

```yaml
GET nome_do_indice/_ilm/explain

GET .ds-traces-apm-contextprocessorout-2026.03.26-000001/_ilm/explain
```
