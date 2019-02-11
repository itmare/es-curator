다양한 배치작업 가능케 하는 Curator
-----------------------------------

-	보관 기간이 넘은 인덱스 삭제
-	인덱싱이 끝난 인덱스의 forcemerge (인덱스의 검색성능을 높이기 위해)
-	warm data node로의 샤드 이동
	-	ex: hot데이터노드로 한달동안 데이터를 밀어넣고, 총 6개월 서비스, 나머지 5개월은 warm데이터로 이동, 한달치가 다 찬이후로는 warm데이터를 hot데이터로 이동, 1달 후부터는 매일해야하는 작업
-	매일 해야하는 일들을 일일히 사람이 챙기기 어려움
-	일일 배치로 편리하게 사용할 수 있는 툴
-	crontab을 활용하여 원하는 시간에 배치 진행 (curator는 스케줄링 기능은 없음)

<br>

### 설치

-	python package manager pip을 통해 설치

```shell
sudo easy_install pip
sudo pip install elasticsearch-curator

sudo pip show elasticsearch-curator
ls /bin/curator
```

-	CONFIG.yml, ACTION.yml
	-	클러스터의 url이나 로그위치를 지정하는 config파일
	-	실제 인덱스를 삭제하고 warm데이터로 내리고 forcemerge하는, 실제 curating할 action파일

```shell
# dry-run으로 모의 실행
$ /bin/curator [--dry-run]  [--config CONFIG_FILE.yml]   ACTION_FILE.yml
```

-	es.config.yml

```shell
client:
  hosts:
    - es01 # es클러스터이름
  port: 9200 # es포트
  url_prefix:
  use_ssl: False
  certificate:
  client_cert:
  client_key:
  ssl_no_validate: False
  http_auth:  #엔진엑스에 인증걸어서 사용할 경우,
  timeout: 30
  master_only: False

logging:
  loglevel: INFO
  logfile: /home/<USER>/curator/log/es.log
  logformat: default
  blacklist: ['elasticsearch', 'urllib3']

```

<br><br>

### 사전 준비

```json
POST test-2019-02-09/_doc
{
  "TT":"TT"
}

POST test-2019-02-10/_doc
{
  "TT":"TT"
}

POST test-2019-02-11/_doc
{
  "TT":"TT"
}

POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "test-2019-02-10",
        "alias": "test-today"
      }
    }
  ]
}

GET test-2019-02-10

PUT test-2019-02-10/_settings
{
  "index.routing.allocation.require.box_type":"hot"
}
```

-	curator가 사용할 파일 경로 생성

```shell
# curator 경로 생성
mkdir curator
# action파일 경로 생성
mkdir curator/action
# config파일 경로 생성
mkdir curator/config
# log파일 경로 생성 및 log적재 파일 생성
mkdir curator/logs
touch es.log
```

<br>

### Add or remove indices (or both) from an alias

-	어제 index에 붙어있던 alias를 오늘로 옮기고 싶을때

-	alias-es.action.yml

```shell
actions:
  1:
   action: alias
   options:
      name: test-today
   add:
      filters:
      - filtertype: pattern
        kind: prefix
        value: 'test-*'
        exclude:
      - filtertype: age
        source: name
        direction: younger
        timestring: '%Y-%m-%d'
        unit: days
        unit_count: 1
        exclude:
   remove:
      filters:
      - filtertype: pattern
        kind: prefix
        value: 'test-*'
        exclude:
      - filtertype: age
        source: name
        direction: older
        timestring: '%Y-%m-%d'
        unit: days
        unit_count: 1
        exclude:
```

```shell
# 실행
$ curator --config config/es.config.yml action/alias-es.action.yml
```

-	**참고: 위와 같이 해당 날짜에 인덱싱되었다면, source에 creation_date사용, 아니라면 source를 name으로 사용하고, timestring 추가**

<br>

### Close indices

-	오랜기간 법적으로 5년 저장해야하는데, 실사용은 1개월, 1개월뒤 인덱스 close, 인덱스성능에 좋다.
-	활성화 되어있는 인덱스를 close로 변경

```shell
actions:
  1:
    action: close
    options:
      delete_aliases: False
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
  2:
    action: open
    options:
      disable_action: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
```

<br>

### Delete indices

-	인덱스 삭제

```shell
actions:
  1:
    action: delete_indices
    options:
      ignore_empty_list: True
      continue_if_exception: False
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
```

<br>

### Open closed indices

close된 인덱스를 open

```shell
actions:
  1:
    action: open
    options:
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
```

<br>

### forceMerge indices

```shell
actions:
  1:
    action: forcemerge
    options:
      max_num_segments: 1
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
```

<br>

### rollover indices

-	특정상황에 따라 새로운 인덱스에 인덱싱

```shell
actions:
  1:
    action: rollover
    options:
      name: test-today
      conditions:
        max_age: 1d
        max_docs: 2
      extra_settings:
        index.number_of_shards: 3
        index.number_of_replicas: 1
      continue_if_exception: False
      disable_action: False
```

<br>

### warm data indices

-	hot node에 있는 데이터를 warm node로 이동

```shell
actions:
  1:
    action: allocation
    options:
      key: box_type
      value: warm
      allocation_type: require
    filters:
    - filtertype: pattern
      kind: prefix
      value: 'test-*'
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y-%m-%d'
      unit: days
      unit_count: 1
      exclude:
```
