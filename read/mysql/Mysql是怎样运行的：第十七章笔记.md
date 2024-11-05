# Mysql是怎样运行的：第十七章笔记

---

## 概要

---

在 MySQL 5.6 以前的版本中，我们只能通过 EXPLAIN 语句查看到最后查询优化器决定使用的执行计划，而无法查看到查询优化器生成执行计划的整个过程。但是在在 MySQL 5.6 以后的版本中，我们拥有了`optimizer trace`功能，用于查看查询优化器生成执行计划的整个过程。

## optimizer trace 功能的开启、使用与关闭

---

### 开启

---

MySQL 存在系统变量`optimizer_trace`，这个变量掌管着 optimizer trace 功能的开启与关闭。

```mysql
-- 查看变量 optimizer_trace
SHOW VARIABLES LIKE 'optimizer_trace';

-- 变量信息
-- enabled 控制 optimizer trace 功能的开启与关闭（默认值 off）
-- one_line 控制是否将查询优化器生成执行计划的整个过程在一行内展示（默认值 off，建议保持 off）
+-----------------+--------------------------+
| Variable_name   | Value                    |
+-----------------+--------------------------+
| optimizer_trace | enabled=off,one_line=off |
+-----------------+--------------------------+

-- 开启 optimizer trace 功能
SET optimizer_trace="enabled=on";
```

<br />

### 使用

---

在开启 optimizer trace 功能后，我们可以执行一遍我们想要查看优化过程的查询语句，而后可以到`information_schema`数据库下的`OPTIMIZER_TRACE` 表中查看完整的优化过程，`OPTIMIZER_TRACE` 表解构如下：

* QUERY：表示我们的查询语句。
* TRACE： 表示优化过程的JSON格式文本。
* MISSING_BYTES_BEYOND_MAX_MEM_SIZE：由于优化过程可能会输出很多，当超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
* INSUFFICIENT_PRIVILEGES：表示是否没有权限查看优化过程，默认值是 0，只有某些特殊情况下才会是 1。

查看`OPTIMIZER_TRACE` 表的 SQL 语句如下：

```mysql
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

若我们想查看其他查询语句的优化过程，重复以下步骤即可：

1. 执行一个想查看优化过程的查询语句
2. 从`OPTIMIZER_TRACE`表中查看对应查询语句的优化过程

<br />

### 关闭

---

当停止查看语句的优化过程时，记得把 optimizer trace 功能关闭。

```mysql
SET optimizer_trace="enabled=off";
```

<br />

## 举例

---

```mysql
-- 开启 optimizer trace 功能
SET optimizer_trace="enabled=on";

--  EXPLAIN 语句
EXPLAIN 
SELECT * 
FROM s1 
WHERE key1 > 'z' 
AND key2 < 1000000 
AND key3 IN ('a', 'b', 'c') 
AND common_field = 'abc';

-- 执行计划 possible_keys 字段值为 idx_key2, idx_key1, idx_key3
-- 执行计划 key 字段为 idx_key2 
-- 执行计划其他信息略...

-- 查看上述查询语句的优化过程
SELECT * FROM information_schema.OPTIMIZER_TRACE

-- 优化过程
# 分析的查询语句是什么
QUERY: SELECT * FROM s1 WHERE
    key1 > 'z' AND
    key2 < 1000000 AND
    key3 IN ('a', 'b', 'c') AND
    common_field = 'abc'

# 优化的具体过程（大致分为三个阶段：prepare 阶段、optimize 阶段和 execute 阶段）
TRACE: {
  "steps": [
    {
      # prepare 阶段
      "join_preparation": {  
        "select#": 1,
        "steps": [
          {
            "IN_uses_bisection": true
          },
          {
            "expanded_query": "/* select#1 */ select `s1`.`id` AS `id`,`s1`.`key1` AS `key1`,`s1`.`key2` AS `key2`,`s1`.`key3` AS `key3`,`s1`.`key_part1` AS `key_part1`,`s1`.`key_part2` AS `key_part2`,`s1`.`key_part3` AS `key_part3`,`s1`.`common_field` AS `common_field` from `s1` where ((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      # optimize 阶段（基于成本的优化过程集中在这个阶段）
      "join_optimization": {  
        "select#": 1,
        "steps": [
          {
            # 处理搜索条件
            "condition_processing": {  
              "condition": "WHERE",
              # 原始搜索条件
              "original_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))",
              "steps": [
                {
                  # 等值传递转换
                  "transformation": "equality_propagation",
                  "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                },
                {
                  # 常量传递转换    
                  "transformation": "constant_propagation",
                  "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                },
                {
                  # 去除没用的条件
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            # 替换虚拟生成列
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            # 表的依赖信息
            "table_dependencies": [
              {
                "table": "`s1`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
            ] /* ref_optimizer_key_uses */
          },
          {
          
            # 预估不同单表访问方法的访问成本（对单表查询，关注此字段，这个字段深入分析了对单表查询的各种执行方案的成本）
            "rows_estimation": [
              {
                "table": "`s1`",
                "range_analysis": {
                  "table_scan": {   
                	# 全表扫描的行数以及成本
                    "rows": 9688,
                    "cost": 2036.7
                  } /* table_scan */,
                  
                  # 分析可能使用的索引
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",  
                      # 主键不可用
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_key2",  
                      # idx_key2 可能被使用
                      "usable": true,
                      "key_parts": [
                        "key2"
                      ] /* key_parts */
                    },
                    {
                      "index": "idx_key1",  
                      # idx_key1 可能被使用
                      "usable": true,
                      "key_parts": [
                        "key1",
                        "id"
                      ] /* key_parts */
                    },
                    {
                      "index": "idx_key3",  
                      # idx_key3 可能被使用
                      "usable": true,
                      "key_parts": [
                        "key3",
                        "id"
                      ] /* key_parts */
                    },
                    {
                      "index": "idx_key_part",  
                      # idx_keypart 不可用
                      "usable": false,
                      "cause": "not_applicable"
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  
                  # 分析各种可能使用的索引的成本
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        # 使用 idx_key2 的成本分析
                        "index": "idx_key2",
                        # 使用 idx_key2 的范围区间
                        "ranges": [
                          "NULL < key2 < 1000000"
                        ] /* ranges */,
                        # 是否使用 index dive（第十二章有详细介绍）
                        "index_dives_for_eq_ranges": true,   
                        # 使用该索引获取的记录是否按照主键排序
                        "rowid_ordered": false,    
                        # 是否使用 mrr
                        "using_mrr": false,  
                        # 是否是索引覆盖访问
                        "index_only": false,    
                        # 使用该索引获取的记录条数
                        "rows": 12,     
                        # 使用该索引的成本
                        "cost": 15.41,  
                        # 是否选择该索引
                        "chosen": true  
                      },
                      {
                        # 使用idx_key1的成本分析
                        "index": "idx_key1",
                        # 使用idx_key1的范围区间
                        "ranges": [
                          "z < key1"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 266,
                        "cost": 320.21,
                        # 不选择该索引
                        "chosen": false,
                        # 原因：因为成本太大
                        "cause": "cost"   
                      },
                      {
                        # 使用idx_key3的成本分析
                        "index": "idx_key3",
                        # 使用idx_key3的范围区间
                        "ranges": [
                          "a <= key3 <= a",
                          "b <= key3 <= b",
                          "c <= key3 <= c"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,   
                        "rowid_ordered": false,   
                        "using_mrr": false,   
                        "index_only": false,   
                        "rows": 21,   
                        "cost": 28.21,  
                        "chosen": false,
                        "cause": "cost"
                      }
                    ] /* range_scan_alternatives */,
                    
                    # 分析使用索引合并的成本
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  
                  # 对于上述单表查询s1最优的访问方法
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "idx_key2",
                      "rows": 12,
                      "ranges": [
                        "NULL < key2 < 1000000"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 12,
                    "cost_for_plan": 15.41,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            
            # 分析各种可能的执行计划（对多表连接查询，关注此字段，这个字段里写明了各种不同的连接方式所对应的成本）
            #（对多表查询这可能有很多种不同的方案，单表查询的方案上面已经分析过了，直接选取 idx_key2 就好）
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`s1`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 12,
                      "access_type": "range",
                      "range_details": {
                        "used_index": "idx_key2"
                      } /* range_details */,
                      "resulting_rows": 12,
                      "cost": 17.81,
                      "chosen": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 12,
                "cost_for_plan": 17.81,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            # 尝试给查询添加一些其他的查询条件
            "attaching_conditions_to_tables": {
              "original_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`s1`",
                  "attached": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            # 再稍稍的改进一下执行计划
            "refine_plan": [
              {
                "table": "`s1`",
                "pushed_index_condition": "(`s1`.`key2` < 1000000)",
                "table_condition_attached": "((`s1`.`key1` > 'z') and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      # execute阶段
      "join_execution": {    
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}

# 因优化过程文本太多而丢弃的文本字节大小，值为0时表示并没有丢弃
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0

# 权限字段
INSUFFICIENT_PRIVILEGES: 0
```











