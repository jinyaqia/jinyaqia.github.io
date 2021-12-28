# union all并行查询提速

{:toc}

* TOC

# 原查询是：

```
SELECT
    a.count AS reqcount,
    b.count AS tenseccount,
    c.count AS thirtyseccount,
    d.count AS onemincount,
    e.count AS fivemincount,
    f.count AS failcount,
    g.count AS verifycount,
    h.count AS verifysuccesscount,
    i.count AS cost,
    j.count AS onetofivemincount
FROM
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS a
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime < 10000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS b
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 10000) AND (g.receiptusetime < 30000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS c
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 30000) AND (g.receiptusetime < 60000) AND (g.receiptusetime >= 60000) AND (g.receiptusetime < 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS d
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS e
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND ((g.status = '0') OR (g.status = '2')) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS f
,
(
    SELECT count(0) AS count
    FROM dis_sms_captcha AS c
    WHERE (1 = 1) AND (c.appid = '5001') AND (c.its >= 1587312000) AND (c.its <= 1587484800)
) AS g
,
(
    SELECT count(0) AS count
    FROM dis_sms_captcha AS c
    WHERE (1 = 1) AND (c.appid = '5001') AND (c.mobileverifycount > '0') AND (c.its >= 1587312000) AND (c.its <= 1587484800)
) AS h
,
(
    SELECT sum(toDecimal128(notEmpty(price), 4)) AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.appid = '5001') AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS i
,
(
    SELECT count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 60000) AND (g.receiptusetime < 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) AS j
WHERE 1 = 1
```

# 查询是串行的，执行结果为:

```
1 rows in set. Elapsed: 6.443 sec. Processed 8.20 million rows, 47.30 MB (1.27 million rows/s., 7.34 MB/s.)
```

# 使用union all是查询可以并行，sql变为
```
SELECT
   name,count
FROM (
(
    SELECT 'a' as name,count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
union all
(
    SELECT 'b',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime < 10000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
union all
(
    SELECT 'c',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 10000) AND (g.receiptusetime < 30000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) 
union all
(
    SELECT 'd',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 30000) AND (g.receiptusetime < 60000) AND (g.receiptusetime >= 60000) AND (g.receiptusetime < 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
) 
union all
(
    SELECT 'e',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
union all
(
    SELECT 'f',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND ((g.status = '0') OR (g.status = '2')) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
union all
(
    SELECT 'g',count(0) AS count
    FROM dis_sms_captcha AS c
    WHERE (1 = 1) AND (c.appid = '5001') AND (c.its >= 1587312000) AND (c.its <= 1587484800)
) 
union all
(
    SELECT 'h',count(0) AS count
    FROM dis_sms_captcha AS c
    WHERE (1 = 1) AND (c.appid = '5001') AND (c.mobileverifycount > '0') AND (c.its >= 1587312000) AND (c.its <= 1587484800)
)
union all
(
    SELECT 'i',sum(toDecimal128(notEmpty(price), 4)) AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.appid = '5001') AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
union all
(
    SELECT 'j',count() AS count
    FROM dis_sms_gateway AS g
    WHERE (1 = 1) AND (g.receiptusetime >= 60000) AND (g.receiptusetime < 300000) AND (g.its >= 1587312000) AND (g.its <= 1587484800)
)
)

```

# 查询结果为：
```
10 rows in set. Elapsed: 0.376 sec. Processed 8.20 million rows, 47.30 MB (21.83 million rows/s., 125.94 MB/s.)
```