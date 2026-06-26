# Customization Governance

**Inventário automatizado de customizações para instâncias ServiceNow.**

Desenvolvido por [Bruno Santos](https://github.com/brunoasantos) | BairesLab

---

## O que é

Customization Governance é uma aplicação ServiceNow que mapeia automaticamente todos os artefatos customizados de uma instância, usando o critério oficial da plataforma (`SncAppFiles.hasCustomerUpdate()`), o mesmo utilizado pelo mecanismo de upgrade do ServiceNow.

O objetivo é responder com precisão à pergunta que todo cliente faz: **quantas customizações existem na minha instância, quem as criou e quando — e o que fazer com cada uma delas?**

---

## Por que isso importa

Instâncias ServiceNow acumulam customizações ao longo dos anos. Muitas dessas customizações:

- Não têm rastreabilidade de autoria
- Estão em Update Sets sem nome ou no Default
- Nunca foram documentadas
- Impactam negativamente upgrades e manutenção

Esta aplicação entrega um inventário confiável, auditável e apresentável para stakeholders — sem depender de estimativas ou análises manuais.

---

## Categorias mapeadas

### v1.0.0 — Categorias originais (14)

| Categoria | Tabela ServiceNow |
|---|---|
| Custom Tables | `sys_db_object` |
| Custom Fields | `sys_dictionary` |
| Business Rules | `sys_script` |
| Client Scripts | `sys_script_client` |
| UI Policies | `sys_ui_policy` |
| UI Actions | `sys_ui_action` |
| Script Includes | `sys_script_include` |
| Scheduled Jobs | `sysauto_script` |
| Flows | `sys_hub_flow` |
| Record Producers | `sc_cat_item_producer` |
| Notifications | `sysevent_email_action` |
| Inbound Actions | `sysevent_in_email_action` |
| SLA Definitions | `contract_sla` |
| ACLs | `sys_security_acl` |

### v1.2.0 — Categorias adicionadas (9)

| Categoria | Tabela ServiceNow |
|---|---|
| Reports | `sys_report` |
| SP Widgets | `sp_widget` |
| Custom Roles | `sys_user_role` |
| Transform Maps | `sys_transform_map` |
| Data Sources | `sys_data_source` |
| Catalog Items | `sc_cat_item` |
| UI Pages | `sys_ui_page` |
| Flow Actions | `sys_hub_action_type_definition` |
| Scoped Apps | `sys_app` |

> **Nota sobre Scoped Apps:** esta categoria não aplica o filtro `SncAppFiles.hasCustomerUpdate()` pois aplicações criadas do zero pelo cliente não são modificações de artefatos OOB. Todas as Scoped Apps encontradas são coletadas. Use o filtro de categoria no dashboard para incluir ou excluir esse grupo dos totais.

---

## Critério de customização

O collector usa `SncAppFiles.hasCustomerUpdate()`, o mesmo método interno que o ServiceNow utiliza para identificar artefatos modificados pelo cliente durante upgrades. Isso elimina falsos positivos de artefatos OOB que apenas passaram por Update Sets de sistema.

**Por que não usar `sys_metadata_customization`?**

A tabela `sys_metadata_customization` com `author_type = Custom` pode conter dezenas de milhares de registros em instâncias com histórico longo, mas inclui ruído expressivo: artefatos deletados, traduções e labels, sub-artefatos internos de dashboards e ACLs. Nosso critério foca em artefatos executáveis de alto impacto, com número auditável e defensável categoria por categoria.

---

## Estrutura da aplicação

```
Customization Governance (Global scope)
├── Script Include: CG_InventoryCollector
├── Scheduled Script Execution: CG – Weekly Inventory (domingo 02:00)
├── Table: u_cg_inventory
├── System Property: cg.excluded_scopes
├── System Property: cg.days_inactive_threshold
└── Platform Analytics Dashboard: Customization Governance (3 abas)
```

---

## Campos da tabela `u_cg_inventory`

| Campo | Tipo | Descrição |
|---|---|---|
| `u_category` | String | Categoria do artefato |
| `u_name` | String | Nome técnico |
| `u_label` | String | Label legível |
| `u_scope` | String | Scope da aplicação |
| `u_sys_id_ref` | String | Sys ID do artefato original |
| `u_table_name` | String | Tabela onde o artefato atua |
| `u_active` | Boolean | Se o artefato está ativo na instância |
| `u_is_customized` | Boolean | Confirmação pelo critério SncAppFiles |
| `u_update_set` | String | Nome do Update Set de origem |
| `u_last_updated_on` | DateTime | Data da última modificação |
| `u_last_updated_by` | String | Responsável pela última modificação |
| `u_last_snapshot` | DateTime | Data da última coleta |
| `u_status` | Choice | Status de governança (ver classificação automática) |
| `u_notes` | String | Notas automáticas e manuais de decisão |

---

## Classificação automática de status (v1.3.0)

A partir da v1.3.0, o campo `u_status` é populado automaticamente durante a coleta e pode ser re-classificado a qualquer momento via `runClassification()`.

### Valores do campo `u_status`

| Status | Critério automático |
|---|---|
| `oob_modified` | Escopo Global + categoria de alto impacto (BR, CS, UI Policy, ACL, Notification, UI Action, Inbound Action, Scheduled Job, Script Include) |
| `to_remove` | Artefato inativo há mais de `cg.days_inactive_threshold` dias |
| `active` | Todos os demais casos |
| `justified` | Preenchimento manual — nunca sobrescrito automaticamente |

### Ordem de prioridade das regras

```
1. oob_modified  (Escopo Global + categoria crítica)
2. to_remove     (Inativo há mais de N dias)
3. active        (padrão)
```

> O status `justified` é protegido — uma vez definido manualmente, o script nunca o sobrescreve.

### Campo `u_notes`

Quando o script classifica um artefato, popula `u_notes` automaticamente com prefixo `[AUTO]`:

```
[AUTO] Inativo há 4367 dias. Último update: 2014-10-06 por mauricio.machado. Candidato à remoção (threshold: 180 dias).
[AUTO] Artefato OOB modificado. Categoria: ACL. Escopo: Global. Revisar necessidade de manter customização.
```

O campo permanece editável para notas manuais com prefixo `[MANUAL]`.

---

## System Properties

| Property | Tipo | Valor padrão | Descrição |
|---|---|---|---|
| `cg.excluded_scopes` | String | (vazio) | Scopes a ignorar na coleta, separados por vírgula |
| `cg.days_inactive_threshold` | Integer | `180` | Dias sem uso para classificar artefato como `to_remove` |

---

## Instalação

### Pré-requisitos

- ServiceNow Zurich ou superior
- Acesso de administrador na instância destino
- Background Script execution habilitado

### Passo a passo

**1. Importar o Update Set**

Na instância destino, acesse:

```
System Update Sets > Retrieved Update Sets > Import Update Set from XML
```

Faça upload do arquivo `Customization Governance.xml` e clique em **Upload**.

**2. Preview e Commit**

Clique em **Preview Update Set** e verifique que não há erros. Depois clique em **Commit Update Set**.

**3. Configurar as System Properties**

Após o commit, acesse:

```
System Properties > cg.excluded_scopes
```

Preencha com o scope identifier da aplicação que deseja excluir da coleta, separados por vírgula. Exemplo:

```
x_minha_app,x_outro_scope
```

> Deixe vazio se não houver scopes a excluir.

```
System Properties > cg.days_inactive_threshold
```

Defina o threshold em dias para classificação `to_remove`. Padrão: `180`.

**4. Rodar o inventário inicial**

Acesse `System Definition > Background Scripts` e execute:

```javascript
var collector = new CG_InventoryCollector();
var results = collector.runFullInventory();
gs.info('Total: ' + results.collected + ' | Erros: ' + results.errors.length);
```

**5. Classificar registros existentes (apenas na primeira instalação em instância com dados)**

```javascript
var collector = new CG_InventoryCollector();
var result = collector.runClassification();
gs.info('Classificados: ' + result.classified + ' | Erros: ' + result.errors);
```

**6. Acessar o dashboard**

Acesse `Platform Analytics > Dashboards` e abra **Customization Governance**.

---

## Dashboard

O dashboard **Customization Governance** possui 3 abas:

### Aba 1 — Overview

Visão geral de toda a instância:

- Total de customizações, ativas e inativas
- Distribuição por categoria (donut)
- Distribuição por status (donut)
- Customizações por escopo (barras)
- Customizações por Update Set (barras)
- Customizações por usuário (barras)
- Candidatas a remoção por categoria (barras — filtro: `u_status = to_remove`)

### Aba 2 — Featured Apps

Foco em aplicações escopadas (exclui Global):

- Total de customizações escopadas e ativas
- Distribuição por categoria (donut)
- Customizações por app (barras)
- Status das apps escopadas (donut)

### Aba 3 — High Impact

Foco nos artefatos de maior impacto em runtime — exclui Reports e UI Policies:

- Categorias cobertas: Business Rule, Client Script, ACL, Notification, Inbound Action, Scheduled Job
- Total e total ativo
- Distribuição por categoria, status, escopo e usuário

---

## Agendamento automático

O inventário roda automaticamente todo domingo às 02:00 via `Scheduled Script Execution`. A partir da v1.3.0, a classificação de `u_status` ocorre automaticamente durante cada coleta. Para rodar manualmente:

```
System Definition > Scheduled Script Executions > CG – Customization Governance – Weekly Inventory
```

Clique em **Execute Now**.

---

## Portabilidade

A aplicação foi projetada para ser instalada em qualquer instância ServiceNow. Os únicos ajustes necessários por instância são as System Properties `cg.excluded_scopes` e `cg.days_inactive_threshold`.

**Testado em:**

| Instância | Versão | Customizações mapeadas |
|---|---|---|
| Instância de produção (grande porte) | Zurich | ~8.400 (v1.0.0 — 14 categorias) |
| Instância de produção (grande porte) | Zurich | 18.790 (v1.2.0 — 23 categorias) |
| Instância LAB | Zurich | ~485 (v1.0.0 — 14 categorias) |
| Instância LAB | Zurich | 493 (v1.2.0 — 23 categorias) |
| Instância LAB | Zurich | 493 (v1.3.0 — classificação automática de status) |

---

## Changelog

### v1.3.0

- Classificação automática de `u_status` integrada ao `collect()` e ao `runClassification()`
- Novo método público `runClassification()` para re-classificar registros existentes sem re-coletar
- Nova System Property `cg.days_inactive_threshold` (padrão: 180 dias) — configurável por instância
- Campo `u_notes` populado automaticamente com prefixo `[AUTO]` e limpo quando status muda
- Status `justified` protegido contra sobrescrita automática
- Correção do cálculo de dias inativos — substituído `gs.dateDiff()` por `GlideDateTime.getNumericValue()`
- Dashboard ampliado para 3 abas: Overview, Featured Apps e High Impact

### v1.2.0

- `_collectCustomFields` agora varre campos `u_` em **todas as tabelas** (portável entre instâncias), sem filtro hardcoded de tabelas específicas
- `_collectScopedApps` corrigido: removidos filtros incompatíveis com `sys_app` (`sys_update_name`, `sys_scope`) que geravam warnings nos logs
- Campo `scope` de Scoped Apps corrigido para usar `gr.getValue('scope')` (campo real da tabela)
- Total de categorias: 23

### v1.1.0 *(interno)*

- Adicionadas 9 novas categorias: Reports, SP Widgets, Custom Roles, Transform Maps, Data Sources, Catalog Items, UI Pages, Flow Actions, Scoped Apps
- Análise comparativa com `sys_metadata_customization` — mantida abordagem por tabelas individuais com `SncAppFiles.hasCustomerUpdate()` por precisão e rastreabilidade de Update Set

### v1.0.0

- Release inicial com 14 categorias de artefatos executáveis
- Critério `SncAppFiles.hasCustomerUpdate()` substituindo `sys_update_name ISNOTEMPTY` (eliminação de ~93% de falsos positivos)
- System Property `cg.excluded_scopes` para portabilidade entre instâncias
- Scheduled Job semanal (domingo 02:00) para minimizar impacto em produção 24x7
- Dashboard Platform Analytics com 6 visualizações

---

## Contribuição

Pull requests são bem-vindos. Para mudanças significativas, abra uma issue primeiro para discutir o que você gostaria de alterar.

---

## Licença

MIT
