# Sumo Logic API

This role handles various actions regarding Sumo Logic's API
an effort was made to stay Ansible native and to not write anything to disk.

**IMPORTANT** the Ingest Budgets field value can take any kind of value! this can lead to wrong assignments or not assinging at all

> **Automation** the role can run mutliple times with no issues.
> multiple `when` conditions at different level will try and prevent api calls that will fail
> such as trying to create existing resources (Returns a 404 naturally and MSG saying the resource exists)
>> Instead of executing an api call on all items and catch the response we simply don't execute when we know it will fail

**Rate-limiting** API requests, append `{{ sumologic['endpoints']['rate_limit'] | default() }}` to the `url:`

## Important links

[ingest bugets api](https://api.us2.sumologic.com/docs/#operation/)
[assign collector to ingest buget](https://help.sumologic.com/Beta/Ingest_Budgets/Assign_Collector_to_Ingest_Budget)
[collector api methods](https://help.sumologic.com/APIs/01Collector-Management-API/Collector-API-Methods-and-Examples#PUT_methods)

## Role Tags

* sumo.logic.api.ingestbudget.get
* sumo.logic.api.ingestbudget.actions
* sumo.logic.api.collector.get
* sumo.logic.api.collector.actions

## Role Vars

`API response` variables and `Other variables` are included in order to clarify the role.

### Filters

Variables that can be passed to the role in order to filter when to execute a task.

| Variable | Type        | Description | Default Value | Comment |
|:-:|:-:|:-:|:-:|:-:|
|_ingestbudget_names|list|a list of names which is also a flag for the task-`update ingest budget by name`|`undefined`|example: `'["default","demo","prod"]'`|
|filter_bynames|list|a list of names which is also a flag for the task-`get collectors by name (filter_bynames)`|`undefined`|example: `"['dev swarm-manager-new-1','demo swarm-manager-demo']"`|
|filter_byids|list|a list of ids which is also a flag for the task-`get collectors by ids (collector_ids)`|`undefined`|example: `'["153389659","153656487"]'`|

> The `when` condition used with the lists uses `in` to find the name of the item in the loop `in` the list of names/ids  
> **`filter_byids`** and **`filter_byids`** are mutually excluded  

**Lists of names usage in the role**  
**Names** can be partial or exact                           (we use `in` to search and match with a list)  
**Ids** should be exact although partial ids will match     (we use `in` to search and match with a list)  
**_collector_types** see [required variables]                (we use `in` to search and match with a list)  

> **AVOID Whitespace in Lists!**

### Role control

| Variable | Type | Default Value | Description |
|:-:|:-:|:-:|:-:|
|create_ingestbudgets|Bool|False|a flag to exectue Taks: `create ingest budgets`|
|update_ingestbudgets|Bool|False|a flag to execute Task: `update ingest budgets`|
|delete_ingestbudgets|Bool|False|a flag to execute Tasks in the `delete ingest budgets` block|
|get_dangling_collectors|Bool|False|a flag to execute Task: `get all dangling collectors`|

#### DELETING Ingest Budgets

i. set `delete_ingestbudgets` to `True` to delete **all** of the ingest budgets
ii. set `_ingestbudget_names` with `delete_ingestbudgets` to delete a named budget/s

### Required variables

| Variable | Type | Default Value | Description |
|:-:|:-:|:-:|:-:|
|base_uri|string|[Sumo Logic Endpoint US2](https://api.us2.sumologic.com/api)|Sumo logic Endpoint (Check Docs for other regions)|
|_collector_types|list|`"['Installable']"`|Types of Collector to further filter api calls; Examples: ('Hosted','Installable')|

| Variable | Type | Description | Comment |
|:-:|:-:|:-:|:-:|
|sumologic|dict|used for SumoLogic vars|Subkeys: 'api_credentials','endpoints'|
|predefined_ingestbudgets|dict|Predefined ingest budgets [JSON]|(1) template json for api post (2) comparing names before posting|

### API Response variables

* Ingest Budgets

| Variable | Type | Description | Comment |
|:-:|:-:|:-:|:-:|
|rjson_ingestbudgets|list of dicts|results of URI module get|API response|
|_ingestbudgets|dict|a dictionary of ingest budgets|create from `rjson_ingestbudgets`|

* Collectors

| Variable | Type | Description | Comment |
|:-:|:-:|:-:|:-:|
|rjson_collectors|list of dicts|results of URI module get|API response|
|rjson_collectors_detailed_byname|dict|a dictionary of collectors filtered by names|MUST use the filter `when: item not skipped` on downstream tasks|

### Other variables

| Variable | Type | Description | Comment |
|:-:|:-:|:-:|:-:|
|_ingestbudget_templates|dict|Used to create a list of ingest budget jsons|created from `predefined_ingestbudgets`|
|_predef_ingestbudgets|list|Used to create **new** ingest budgets, to loop over predefined ingest budgets and `when` conditions|created from `.results` of `_ingestbudget_templates`|
|_env_ingestbudget|string|contains a string like `dev_5_gb` per environment|defaults to `env_ingest_budget` which should be present in inventory vars|

### ETAG

etag is a unique change identifier which changes whenever we *update* a collector.
etag is returned when doing a GET request for a specific collector. (Hosted or not)

* ETAG (Unique change identifier)
* Changes after a successful POST to update the collector
    >For this reason you cannot update a collector twice without doing a GET between successful updates

## Immutable Properties

For collector versions 19.137 and later, the user.properties file lets you pass configuration parameters during the installation of a new unregistered Collector. Once the collector is registered, to see if a parameter can be changed with a collector restart, check the "Can be changed after installation?" column of the table below.

*No*: cannot be changed (Requires a removal of the collector in Sumo Logic and recreating it)  
*API*: must use the API to change  
*Edit*: can be changed via the UI or API  

>**NOTE On "API" Changes**: When updating a collector via the API ensure you've also re-deployed `user.properties` with matching settings to avoid confusion or cases where the collector was deleted in Sumo Logic and re-registered with different values.  
>**Side note on re-registration**: collector who had their access key & id removed cannot re-register unless we re-deploy (we ensure those secrets are not saved past the initial registration)  

|Name|Description|Can be changed after installation?|
|:--:|:---------:|:--------------------------------:|
|fields=[list of fields]|Comma-separated list of key=value fields. To assign an ingest budget use the field _budget with its Field Value.|No, use the Collector Management API to modify.|
|accessid=accessId|Access ID used when logging in with Access ID and Key|No|
|accesskey=accessKey|Access Key used when logging in with Access ID and Key|No|
|category=category|Source category to use when a source does not specify a category.|No, use the Collector Management API to modify.|
|clobber=true/false|When true, if there is any existing collector with the same name, that collector will be deleted (clobbered).|No, use the Collector Management API to modify.|
|description=description|Description for the collector to appear in Sumo Logic.|No, use the Collector Management API to modify.|
|ephemeral=true/false|When true, the collector will be deleted after 12 hours of inactivity.|No, use the Collector Management API to modify|
|hostName=hostname|The host name of the machine on which the collector is running. The host name can be a maximum of 128 characters.|No, use the Collector Management API to modify.|
|name=name|Sets the name of collector used on Sumo Logic.|No, use Edit the Collector or the Collector Management API to modify|
|skipAccessKeyRemoval=true/false|If true, it will skip the access key removal from the user.properties file.|No|
|targetCPU=target|You can choose to set a CPU target to limit the amount of CPU processing a collector uses. The value must be expressed as a whole number percentage.|No, use the Collector Management API to modify.|
|timeZone=timezone|The time zone to use when it cannot be extracted|No, use the Collector Management API to modify.|

**Other settings not specified above**: Yes, with Collector restart.

|Name|Description|Can be changed after installation?|
|:--:|:---------:|:--------------------------------:|
|sources=filepath or folderpath|Specifies a single UTF-8 encoded JSON file, or a folder containing UTF-8 encoded JSON files, that defines the Sources to configure upon Collector registration. **The contents of the file or files is read upon Collector registration only**, it is not synchronized with the Collector's configuration on an on-going basis.|No|
|syncSources=filepath or folderpath|Specifies either a single UTF-8 encoded JSON file, or a folder containing UTF-8 encoded JSON files, that define the Sources to configure upon Collector registration. The Source definitions will be continuously monitored and synchronized with the Collector's configuration.|Yes, with Collector restart.|

>**SynSources** is how we've chosen to handle `sources` in order to easily change them (add/remove/modify) which would not otherwise be possible

### Collector JSON Payload

Supplying the collector JSON to API can be done inline or using a template file.  
In both of those forms we can use further Jinja2 loops and conditions to customize the template.

> Both examples relay on the use of 'with_items:' over '"{{ rjson_collectors_detailed_byname.results|list|unique }}"'  
> Using list to flatten the results and unique to avoid duplicates that happen because of the way 'register' work in Ansible

* Supplying an inline json:

> This is an example of a "safe" template ensuring key-order

```ansible
          _collector_json: |
            {
              "collector": {
                "id": {{ item.item.id }},
                "name": "{{ item.item.name }}",
                "description": "{{ env }}_{{ item.item.name }}_collector",
                "hostName": "{{ item.item.hostName }}",
                "timeZone": "{{ item.item.timeZone }}",
                "fields": {
                  "_budget": "{{ _env_ingestbudget }}",
                },
                "links": [
                  {
                    "rel": "sources",
                    "href": "/v1/collectors/{{ item.item.id }}/sources"
                  }
                ],
                "ephemeral": {{ item.item.ephemeral | lower }},
                "targetCpu": {{ item.item.targetCpu }},
                "sourceSyncMode": "{{ item.item.sourceSyncMode }}",
                "collectorType": "{{ item.item.collectorType }}",
                "collectorVersion": "{{ item.item.collectorVersion }}",
                "osVersion": "{{ item.item.osVersion }}",
                "osName": "{{ item.item.osName }}",
                "osArch": "{{ item.item.osArch }}",
                "lastSeenAlive": {{ item.item.lastSeenAlive }},
                "alive": {{ item.item.alive | lower }}
              }
            }
```

* Using a template file

```ansible
{
  "collector": {
    "id": {{ item.item.id }},
    "name": "{{ item.item.name }}",
    "description": "{{ env }}_{{ item.item.name }}_collector",
    "hostName": "{{ item.item.hostName }}",
    "timeZone": "{{ item.item.timeZone }}",
    "fields": {
      "_budget": "{{ _env_ingestbudget }}",
    },
    "links": [
      {
        "rel": "sources",
        "href": "/v1/collectors/item.item.id/sources"
      }
    ],
    "ephemeral": {{ item.item.ephemeral | lower }},
    "targetCpu": {{ item.item.targetCpu }},
    "sourceSyncMode": "{{ item.item.sourceSyncMode }}",
    "collectorType": "{{ item.item.collectorType }}",
    "collectorVersion": "{{ item.item.collectorVersion }}",
    "osVersion": "{{ item.item.osVersion }}",
    "osName": "{{ item.item.osName }}",
    "osArch": "{{ item.item.osArch }}",
    "lastSeenAlive": {{ item.item.lastSeenAlive }},
    "alive": {{ item.item.alive | lower }}
  }
}
```

### Registering variables

When used in a loop the register variable will append the results of each iteration of the loop to the variable.  
those results can be accessed under the `results` subkey under the variable name.

> **Caution**: when registering a variable in a loop with a `when` condition the results will contain **skipped** items as well.
> You will have to account for that fact and check if the item is skipped when iterating over the results.  
> **Caution**: when using the same variable name in multiple tasks (example below) the variable will be overwritten if all of those tasks
> execute **BUT** it will also be overwritten even if the task was skipped because of then when condition.  
> TL;DR **Do not use the same variable in multiple tasks**

```ansible
---
  - block:
      - name: get collectors by name (filter_bynames)
        debug:
          msg: "{{ item }}"
        with_items: "{{ rjson_collectors.json.collectors }}"
        register: rjson_collectors_detailed
        when:
          - (filter_byids is not defined and filter_bynames is defined)
          - (rjson_collectors is succeeded and item.collectorType in _collector_types)
          - (item.name in filter_bynames)

      - name: get collectors by ids (collector_ids)
        debug:
          msg: "{{ item }}"
        with_items: "{{ rjson_collectors.json.collectors }}"
        register: rjson_collectors_detailed
        when: filter_byids is defined
```

## Loops

Ideally we would like to do small efficent loops and quickly move to the next task.  
The easiest way this was tested was using the example below. this however had serious drawback in regards to readability of the output.

```ansible
OUTER LOOP
- name: test
  include: loop.yml
  with_items: "{{ rjson_collectors.json.collectors }}"
  loop_control:
    loop_var: collector

INNER LOOP.YML
- block:
    - name: DEBUG get collectors by name (filter_bynames)
      debug:
        msg: "{{ collector.name }}"
      with_items: "{{ filter_bynames }}"
      when: item in collector.name
```

## Ansible Playbook

> **ENV Variable Usage**
> 1. `env` is required to include variables and get credentials from credstash
> 2. `env` is used to filter responses from Sumo Logic api or loop over results
> **NOTE**: Take care when mixing env and other settings

* Creating ingest budgets

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.ingestbudget.actions' -e env='dev'
```

> The command above will create the default budget and the budget for the environment
> By default tasks will be skipped if the budget already exists

* Update ingest budgets

To specify values to update an existing budget use `updated_ingestbudgets`.
`updated_ingestbudgets` is a list of dicts. name & description values are optional as they will be set to the dictionary key by default.

```yaml
updated_ingestbudgets:
  - prod_application:
      fieldValue: '6'
      capacityBytes: '6000000000'
      resetTime: 00:00
      action: stopCollecting
      timezone: 'Etc/UTC'
```

> Use the structure above in `defaults` or your own variable file.

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.ingestbudget.actions.update, sumo.logic.api.ingestbudget.actions' -e update_ingestbudget=True -e env='dev'
```

* Delete an ingest budget named 'prod'

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.ingestbudget.get, sumo.logic.api.ingestbudget.actions.delete' -e env='prod' -e delete_ingestbudgets='True' -e _ingestbudget_name='prod'
```

* Get collectors by id

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.get' -e env='dev' -e filter_byids='["153389659","153656487"]'
```

* Update collectors by name

**Names** the filter `in` is used instead of `is` or `==` to allow for partial names

**NOTE** The list must not contain spaces between items

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions' -e env='dev' -e _collector_types="['Installable']" -e filter_bynames="['dev swarm-manager-new-1','dev swarm-manager-new-2','dev tilix-dev-jenkins']"
```

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions' -e env='prod' -e filter_bynames="['prod vpn']"
```

> Mixed types

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions' -e env='dev' -e _collector_types="['Installable','Hosted']" -e filter_bynames="['dev swarm-manager-new-1','dev swarm-manager-new-2','dev tilix-dev-jenkins','dev AWS']"
```

* Update collectors by ids

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions' -e env='dev' -e filter_bynames="['dev swarm-worker-9','dev swarm-worker-8']" -e filter_byids='["153389659","153656487"]'
```

* Update collectors without "{{ env }}" in their name

> Referred in the role as `dangling collectors`

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions' -e env='dev' -e _collector_types="['Hosted','Installable']" -e 'get_dangling_collectors=true
```

* Delete Collectors by environment

> The delete task is intentionally strict to avoid mistakes  
> Exclude if the collector name has 'AWS'

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions, sumo.logic.api.collector.actions.delete' -e env='dev' -e delete_collectors=True -e _collector_types="['Installable']"
```

* Delete collectors  by environment and a list of ids or names

> **NOTE** you must also specify the type of the list via `delete_filter` 'name' or 'id'; since it will be used to determine how the 'when' condition will be evaluated

```shell
DISPLAY_SKIPPED_HOSTS=false ansible-playbook -i localhost sumo-logic-api-playbook.yml --tags 'sumo.logic.api.collector.actions, sumo.logic.api.collector.actions.delete' -e env='dev' -e delete_collectors=True -e _collector_types="['Installable']" -e delete_filter='name' -e delete_list="['dev swarm-manager-new-1','dev swarm-manager-new-2']"
```