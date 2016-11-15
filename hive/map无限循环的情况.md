## 问题
运行以下sql，map无限循环
```sql
insert overwrite table ods_qta_price_item_orc partition(dt='$DATE')
select
    id,
    connect_id,
    from_date,
    to_date,
    piece,
    update_time
from ods_qta_price_item
where dt='$DATE';
```
错误情况如下
```bash
2016-11-15 14:17:15,661 Stage-3 map = 0%,  reduce = 0%
2016-11-15 14:18:13,721 Stage-3 map = 3%,  reduce = 0%, Cumulative CPU 57.87 sec
2016-11-15 14:19:02,625 Stage-3 map = 6%,  reduce = 0%, Cumulative CPU 108.37 sec
2016-11-15 14:20:03,457 Stage-3 map = 11%,  reduce = 0%, Cumulative CPU 171.84 sec
2016-11-15 14:20:33,294 Stage-3 map = 0%,  reduce = 0%
2016-11-15 14:21:34,067 Stage-3 map = 0%,  reduce = 0%, Cumulative CPU 35.62 sec
2016-11-15 14:21:55,701 Stage-3 map = 3%,  reduce = 0%, Cumulative CPU 57.54 sec
2016-11-15 14:22:41,748 Stage-3 map = 6%,  reduce = 0%, Cumulative CPU 104.33 sec
2016-11-15 14:23:42,265 Stage-3 map = 11%,  reduce = 0%, Cumulative CPU 165.85 sec
2016-11-15 14:24:30,461 Stage-3 map = 14%,  reduce = 0%, Cumulative CPU 213.24 sec
2016-11-15 14:25:21,705 Stage-3 map = 17%,  reduce = 0%, Cumulative CPU 264.59 sec
2016-11-15 14:26:13,931 Stage-3 map = 20%,  reduce = 0%, Cumulative CPU 316.51 sec
2016-11-15 14:27:08,338 Stage-3 map = 24%,  reduce = 0%, Cumulative CPU 368.96 sec
2016-11-15 14:27:59,824 Stage-3 map = 28%,  reduce = 0%, Cumulative CPU 418.62 sec
2016-11-15 14:28:57,334 Stage-3 map = 32%,  reduce = 0%, Cumulative CPU 474.66 sec
2016-11-15 14:29:49,010 Stage-3 map = 35%,  reduce = 0%, Cumulative CPU 527.08 sec
2016-11-15 14:30:40,681 Stage-3 map = 38%,  reduce = 0%, Cumulative CPU 578.54 sec
2016-11-15 14:31:35,552 Stage-3 map = 42%,  reduce = 0%, Cumulative CPU 634.32 sec
2016-11-15 14:32:29,946 Stage-3 map = 45%,  reduce = 0%, Cumulative CPU 689.05 sec
2016-11-15 14:33:27,433 Stage-3 map = 48%,  reduce = 0%, Cumulative CPU 745.93 sec
2016-11-15 14:34:15,970 Stage-3 map = 52%,  reduce = 0%, Cumulative CPU 795.53 sec
2016-11-15 14:35:04,393 Stage-3 map = 55%,  reduce = 0%, Cumulative CPU 844.93 sec
2016-11-15 14:35:58,891 Stage-3 map = 59%,  reduce = 0%, Cumulative CPU 900.48 sec
2016-11-15 14:36:53,583 Stage-3 map = 63%,  reduce = 0%, Cumulative CPU 955.82 sec
2016-11-15 14:37:05,171 Stage-3 map = 0%,  reduce = 0%
2016-11-15 14:38:06,212 Stage-3 map = 0%,  reduce = 0%, Cumulative CPU 51.98 sec
2016-11-15 14:38:11,523 Stage-3 map = 3%,  reduce = 0%, Cumulative CPU 56.92 sec
2016-11-15 14:39:00,653 Stage-3 map = 6%,  reduce = 0%, Cumulative CPU 105.66 sec
```