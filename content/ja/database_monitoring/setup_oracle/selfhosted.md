---
description: セルフホストの Oracle のデータベースモニタリングのインストールと構成
further_reading:
- link: /integrations/oracle/
  tag: Documentation
  text: Basic Oracle インテグレーション
is_beta: true
kind: documentation
private: true
title: セルフホストの Oracle のデータベースモニタリングの設定
---

{{< site-region region="gov" >}}
<div class="alert alert-warning">データベースモニタリングはこのサイトでサポートされていません。</div>
{{< /site-region >}}

<div class="alert alert-info">
このページで説明されている機能は非公開ベータ版です。必要な Agent のビルドとインストール手順については、カスタマーサクセスマネージャーにお問い合わせください。
</div>

データベースモニタリングは、クエリサンプルを公開することで、Oracle データベースを深く可視化し、さまざまなワークロードをプロファイリングして問題を診断します。

データベースでデータベースモニタリングを有効にするには、以下の手順を実行します。

1. [Agent にデータベースへのアクセスを付与します](#grant-the-agent-access)。
2. [Agent をインストールします](#install-the-agent)。

## はじめに

サポート対象の Oracle バージョン
: 19c、21c


サポート対象の Agent バージョン
: 必要な Agent のビルドの詳細については、カスタマーサクセスマネージャーにお問い合わせください。

## Agent にアクセスを付与する

Datadog Agent がサンプルを収集するためには、データベース サーバーへの読み取り専用のアクセスが必要となります。

サーバーに接続するための読み取り専用ログインを作成し、必要な権限を付与します。

```SQL
CREATE USER c##datadog IDENTIFIED BY &password CONTAINER = ALL ;

ALTER USER c##datadog SET CONTAINER_DATA=ALL CONTAINER=CURRENT;

grant create session to c##datadog ;
grant select on v_$session to c##datadog ;
grant select on v_$database to c##datadog ;
grant select on v_$containers to c##datadog;
grant select on v_$sqlstats to c##datadog ;
grant select on v_$instance to c##datadog ;
grant select on dba_feature_usage_statistics to c##datadog ;
grant select on V_$SQL_PLAN_STATISTICS_ALL to c##datadog ;
grant select on V_$PROCESS to c##datadog ;
grant select on V_$SESSION to c##datadog ;
grant select on V_$CON_SYSMETRIC to c##datadog ;
grant select on CDB_TABLESPACE_USAGE_METRICS to c##datadog ;
grant select on CDB_TABLESPACES to c##datadog ;
grant select on V_$SQLCOMMAND to c##datadog ;
grant select on V_$DATAFILE to c##datadog ;
```

新しい `view` を作成し、Agent ユーザーにアクセス権を与えます。

```SQL
CREATE OR REPLACE VIEW dd_session AS
SELECT
  s.indx as sid,
  s.ksuseser as serial#,
  s.ksuudlna as username,
  DECODE(BITAND(s.ksuseidl, 9), 1, 'ACTIVE', 0, DECODE(BITAND(s.ksuseflg, 4096), 0, 'INACTIVE', 'CACHED'), 'KILLED') as status,
  s.ksuseunm as osuser,
  s.ksusepid as process,
  s.ksusemnm as machine,
  s.ksusemnp as port,
  s.ksusepnm as program,
  DECODE(BITAND(s.ksuseflg, 19), 17, 'BACKGROUND', 1, 'USER', 2, 'RECURSIVE', '?') as type,
  s.ksusesqi as sql_id,
  sq.force_matching_signature as force_matching_signature,
  s.ksusesph as sql_plan_hash_value,
  s.ksusesesta as sql_exec_start,
  s.ksusesql as sql_address,
  CASE WHEN BITAND(s.ksusstmbv, POWER(2, 04)) = POWER(2, 04) THEN 'Y' ELSE 'N' END as in_parse,
  CASE WHEN BITAND(s.ksusstmbv, POWER(2, 07)) = POWER(2, 07) THEN 'Y' ELSE 'N' END as in_hard_parse,
  s.ksusepsi as prev_sql_id,
  s.ksusepha as prev_sql_plan_hash_value,
  s.ksusepesta as prev_sql_exec_start,
  sq_prev.force_matching_signature as prev_force_matching_signature,
  s.ksusepsq as prev_sql_address,
  s.ksuseapp as module,
    s.ksuseact as action,
    s.ksusecli as client_info,
    s.ksuseltm as logon_time,
    s.ksuseclid as client_identifier,
    s.ksusstmbv as op_flags,
    decode(s.ksuseblocker,
        4294967295, 'UNKNOWN', 4294967294, 'UNKNOWN', 4294967293, 'UNKNOWN', 4294967292, 'NO HOLDER', 4294967291, 'NOT IN WAIT',
        'VALID'
    ) as blocking_session_status,
    DECODE(s.ksuseblocker,
        4294967295, TO_NUMBER(NULL), 4294967294, TO_NUMBER(NULL), 4294967293, TO_NUMBER(NULL),
        4294967292, TO_NUMBER(NULL), 4294967291, TO_NUMBER(NULL), BITAND(s.ksuseblocker, 2147418112) / 65536
    ) as blocking_instance,
    DECODE(s.ksuseblocker,
        4294967295, TO_NUMBER(NULL), 4294967294, TO_NUMBER(NULL), 4294967293, TO_NUMBER(NULL),
        4294967292, TO_NUMBER(NULL), 4294967291, TO_NUMBER(NULL), BITAND(s.ksuseblocker, 65535)
    ) as blocking_session,
    DECODE(s.ksusefblocker,
        4294967295, 'UNKNOWN', 4294967294, 'UNKNOWN', 4294967293, 'UNKNOWN', 4294967292, 'NO HOLDER', 4294967291, 'NOT IN WAIT', 'VALID'
    ) as final_blocking_session_status,
    DECODE(s.ksusefblocker,
        4294967295, TO_NUMBER(NULL), 4294967294, TO_NUMBER(NULL), 4294967293, TO_NUMBER(NULL), 4294967292, TO_NUMBER(NULL),
        4294967291, TO_NUMBER(NULL), BITAND(s.ksusefblocker, 2147418112) / 65536
    ) as final_blocking_instance,
    DECODE(s.ksusefblocker,
        4294967295, TO_NUMBER(NULL), 4294967294, TO_NUMBER(NULL), 4294967293, TO_NUMBER(NULL), 4294967292, TO_NUMBER(NULL),
        4294967291, TO_NUMBER(NULL), BITAND(s.ksusefblocker, 65535)
    ) as final_blocking_session,
    DECODE(w.kslwtinwait,
        1, 'WAITING', decode(bitand(w.kslwtflags, 256), 0, 'WAITED UNKNOWN TIME',
        decode(round(w.kslwtstime / 10000), 0, 'WAITED SHORT TIME', 'WAITED KNOWN TIME'))
    ) as STATE,
    e.kslednam as event,
    e.ksledclass as wait_class,
    w.kslwtstime as wait_time_micro,
    c.name as pdb_name,
    sq.sql_text as sql_text,
    sq.sql_fulltext as sql_fulltext,
    sq_prev.sql_fulltext as prev_sql_fulltext,
    comm.command_name
FROM
  x$ksuse s,
  x$kslwt w,
  x$ksled e,
  v$sqlstats sq,
  v$sqlstats sq_prev,
  v$containers c,
  v$sqlcommand comm
WHERE
  BITAND(s.ksspaflg, 1) != 0
  AND BITAND(s.ksuseflg, 1) != 0
  AND s.inst_id = USERENV('Instance')
  AND s.indx = w.kslwtsid
  AND w.kslwtevt = e.indx
  AND s.ksusesqi = sq.sql_id(+)
  AND s.ksusesph = sq.plan_hash_value(+)
  AND s.ksusepsi = sq_prev.sql_id(+)
  AND s.ksusepha = sq_prev.plan_hash_value(+)
  AND s.con_id = c.con_id(+)
  AND s.ksuudoct = comm.command_type(+)
;

GRANT SELECT ON dd_session TO c##datadog ;
```

## Agent のインストール

Oracle テレメトリーの収集を開始するには、まず [Datadog Agent をインストール][1]します。ベータ版に参加するためには、正しいバージョンをインストールする必要があります。詳細については、カスタマーサクセスマネージャーにお問い合わせください。

Oracle Agent のコンフィギュレーションファイル `/etc/datadog-agent/conf.d/oracle-dbm.d/conf.yaml` を作成します。使用可能なすべての構成オプションは、[サンプルコンフィギュレーションファイル][2]を参照してください。

```yaml
init_config:
instances:
  - dbm: true
    server: '<HOSTNAME>:<SQL_PORT>'
    service_name: "<SERVICE_NAME>" # Oracle CDB サービス名
    username: 'c##datadog'
    password: '<PASSWORD>'
    tags:  # オプション
      - 'service:<CUSTOM_SERVICE>'
      - 'env:<CUSTOM_ENV>'
```

`service` と `env` タグを使用して、共通のタグ付けスキームでデータベースのテレメトリーを他のテレメトリーにリンクします。これらのタグが Datadog でどのように使用されるかについてさらに詳しくは、[統合サービスタグ付け][3]を参照してください。

すべての Agent の構成が完了したら、[Datadog Agent を再起動][4]します。

### 検証

[Agent の status サブコマンドを実行][5]し、**Checks** セクションで `oracle-dbm` を探します。Datadog の[データベース][6]のページへ移動して開始します。

[1]: https://app.datadoghq.com/account/settings#agent
[2]: https://github.com/DataDog/datadog-agent/blob/main/cmd/agent/dist/conf.d/oracle-dbm.d/conf.yaml.example
[3]: /ja/getting_started/tagging/unified_service_tagging
[4]: /ja/agent/guide/agent-commands/#start-stop-and-restart-the-agent
[5]: /ja/agent/guide/agent-commands/#agent-status-and-information
[6]: https://app.datadoghq.com/databases


## その他の参考資料

{{< partial name="whats-next/whats-next.html" >}}