DockerでPostgresを扱う際の個人メモ
===

check version
----


    [apollo:~/postgres:0]% docker exec -it postgres_postgres_1 bash -c "echo 'select version();' | su - postgres -c psql 2>/dev/null"
    No directory, logging in with HOME=/
    version
    ----------------------------------------------------------------------------------------------------------------------------------
     PostgreSQL 10.3 (Debian 10.3-1.pgdg90+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 6.3.0-18+deb9u1) 6.3.0 20170516, 64-bit
     (1 row)


alter timezone
---

pgwebでは変更不可。


    postgres=# ALTER DATABASE test SET timezone TO 'Asia/Tokyo';
    ALTER DATABASE

    postgres=# SELECT pg_reload_conf();
    pg_reload_conf
    ----------------
    t
    (1 row)
    
    postgres=# select now();
    now
    -------------------------------
    2018-05-29 17:24:57.937827+09
    (1 row)
    

    
tableの作成
---

    create table log( host varchar(64), created timestamp with time zone, tstamp timestamp with time zone , level varchar(64), message text);

select
---

winlogbeat経由のイベントログのseverityが日本語で入ってきて、logstashのフィルタでは厳しいので
postgres側で対処。

    select 
      host, 
      tstamp, 
      case 
        when level = 'エラー'  then 'ERROR'
        else level
      end as severity, 
      message 
    from log 
    where tstamp > (current_timestamp + '-1 days')
    order by tstamp desc;

### 2018/05/30 修正

select
  host,
  tstamp,
  case
    when level = '情報'  then 'NOTICE'
    when level = 'エラー'  then 'ERROR'
    when level = '0' then 'EMERGENCY'
    when level = '1' then 'ALERTS'
    when level = '2' then 'CRITICAL'
    when level = '3' then 'ERROR'
    when level = '4' then 'WARNING'
    when level = '5' then 'NOTICE'
  else level
  end as severity,
  message
from log
where tstamp > (current_timestamp + '-1 hours')
order by tstamp desc;


view
---

"create view view名 as" をselectの前につける

TODO
---

エラーのみを抽出したい場合、where句にselectのcaseで捻る前の
値を入れないとselectできない。仕様だが、代替手段は。

select
  host,
  tstamp,
  case
    when level = '情報'  then 'NOTICE'
    when level = 'エラー'  then 'ERROR'
    when level = '0' then 'EMERGENCY'
    when level = '1' then 'ALERTS'
    when level = '2' then 'CRITICAL'
    when level = '3' then 'ERROR'
    when level = '4' then 'WARNING'
    when level = '5' then 'NOTICE'
  else level
  end as severity,
  message
from log
where 
  tstamp > (current_timestamp + '-1 hours') and level in ('ERROR', 'WARNING', 'CRITICAL', 'ALERTS', 'EMERGENCY', '3')
order by tstamp desc;
