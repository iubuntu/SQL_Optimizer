# mysql sql optimizer

## 1. `JOIN:prepare`
> (sql/sql_resolver.cc)


### How many tables

### Prepare tables and check access `setup_tables_and_check_access()`
> (sql/sql_base.cc)

- check the table exists `setup_tables()`

    > Check also that the 'used keys' and 'ignored keys' exists and set up the table structure accordingly.
                    
- for each all tables `check_single_table_access`

### Expand all `*` in given fields `setup_wild()`
>（sql/sql_base.cc:）

-   Drops in all fields instead of current '*' field `insert_fields()`

### Initial group order fun `setup_without_group()` 
> (sql/sql_resolver.cc)

### Remove `order/group(not) having /distinct` from single row sub queries 
`remove_redundant_subquery_clauses()` 

> (sql/sql_resolver.cc)

       a) SELECT * FROM t1 WHERE t1.a = (<single row subquery>)
       b) SELECT a, (<single row subquery) FROM t1 
IN/ALL/ANY/EXISTS subqueries


### resolve_subquery():
> Perform early unconditional subquery transformations

   - Convert `subquery` predicate into `semi-join`, or
   - Mark the subquery for execution using `materialization`, or
   - Perform `IN` =>`EXISTS` transformation, or
   - Perform `more/less ALL/ANY` -> `MIN/MAX` rewrite
   - Substitute trivial scalar-context subquery with its value
 
##### flatten subqueries into a semi-join
      1. Subquery predicate is an IN/=ANY subquery predicate
      2. Subquery is a single SELECT (not a UNION)
      3. Subquery does not have GROUP BY
      4. Subquery does not use aggregate functions or HAVING
      5. Subquery predicate is at the AND-top-level of ON/WHERE clause
      6. We are not in a subquery of a single table UPDATE/DELETE that 
           doesn't have a JOIN (TODO: We should handle this at some
           point by switching to multi-table UPDATE/DELETE)
      7. We're not in a confluent table-less subquery, like "SELECT 1".
      8. No execution method was already chosen (by a prepared statement)
      9. Parent select is not a confluent table-less select
      10. Neither parent nor child select have STRAIGHT_JOIN option.



## 2. Global select optimisation `JOIN::optimize()`
> (sql/sql_optimizer.cc)

## 2.1 logical optimization
### Convert semi-join `subquery` into `join nests` `flatten_subqueries()`


    Convert candidate subquery predicates into semi-join join nests. This
    transformation is performed once in query lifetime and is irreversible.

    Conversion of one subquery predicate
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    We start with a join that has a semi-join subquery:

      SELECT ...
      FROM ot, ...
      WHERE oe IN (SELECT ie FROM it1 ... itN WHERE subq_where) AND outer_where

    and convert it into a semi-join nest:

      SELECT ...
      FROM ot SEMI JOIN (it1 ... itN), ...
      WHERE outer_where AND subq_where AND oe=ie

    that is, in order to do the conversion, we need to

     * Create the "SEMI JOIN (it1 .. itN)" part and add it into the parent
       query's FROM structure.
     * Add "AND subq_where AND oe=ie" into parent query's WHERE (or ON if
       the subquery predicate was in an ON expression)
     * Remove the subquery predicate from the parent query's WHERE

     
    
### `handle_derived`
> Run optimize phase for all derived tables/views 
> used in this SELECT,including those in semi-joins.


### `limit` break full table scan
- distinct | order | group by
    ```
    row_limit= (
        (select_distinct || order || group_list) ?
            HA_POS_ERROR : 
            unit->select_limit_cnt
    );
    ```

- having | OPTION_FOUND_ROWS
    ```
    m_select_limit= unit->select_limit_cnt;
    if (having || (select_options & OPTION_FOUND_ROWS))
        m_select_limit= HA_POS_ERROR;
        
    do_send_rows = (unit->select_limit_cnt > 0) ? 1 : 0;
    ```

### Is first optimization

- Convert all `outer joins` to `inner joins` if possible `simplify_joins`

- `record_join_nest_info`
    
    - record the remaining semi-join structures in the enclosing query block
    - record transformed join conditions in `TABLE_LIST` objects
- build_bitmap_for_nested_joins


### Optimize `where` clause condition `optimize_cond`
- a) applying transitivity to build multiple equality predicates `(MEP)`
      - (`x=y` and `y=z`) ->  `x=y=z` 
      
- b) apply constants where possible.
 
    - If the value of `x` is known to be
    `42`, `x` is replaced with a constant of value `42`. 
    
    - this also applies to MEPs `a)` will become `42=x=y=z`
        
- c) remove conditions that are `impossible` or `always true`

### Optimize `having` conditions, if it can be merge to `where` 

### Optimize FTS queries with `ORDER BY/LIMIT`, but `no WHERE clause` 
- `optimize_fts_limit_query`

     1. It is a single table query
     2. There is no WHERE condition
     3. There is a single ORDER BY element
     4. Ordering is descending  -> "desc"
     5. There is a LIMIT clause
     6. Ordering is on a MATCH expression

### optimize `count(*)`, `min()` and `max()` to const fields
>  there is implicit grouping (aggregate functions but no
     group_list). In this case, the result set shall only contain one
     row.

#### `opt_sum_query`
>
1. no conditions 
2. arguments not null
3. not outer_tables
4. storage engine support maybe_exact_count
    
- `count()` get_exact_record_count

- `min`/`max` 
    - get_index_max_value
    - get_index_min_value
    - the expression is not constant `MAX(1)`


## 2.2 physical optimization
### Calculate best `join order`  `make_join_statistics`

  - Initialize JOIN data structures and setup basic dependencies between tables.

  - Update dependencies based on join information.

  - Make key descriptions (update_ref_and_keys()).

  - Pull out semi-join tables based on table dependencies.

  - Extract tables with zero or one rows as const tables.

  - Read contents of const tables, substitute columns from these tables with
    actual data. Also keep track of empty tables vs. one-row tables.

  - After const table extraction based on row count, more tables may
    have become functionally dependent. Extract these as const tables.

  - Add new sargable predicates based on retrieved const values.

  - Calculate number of rows to be retrieved from each table.
        - constant table is `1`
        - `range` call `get_quick_record_count`
        - `ALL` is `ALL rows`
  - Calculate cost of potential semi-join materializations.

  - Calculate best possible join order based on available statistics.

  - Fill in remaining information for the generated join order.



### `optimize_keyuse`

### `optimize_semijoin_nests_for_materialization`
### Get table order `Optimize_table_order`
> sql/sql_planner.cc

#### set `determine_search_depth()`  default `62` -> `optimizer_search_depth`
> sql/sql_planner.h

- optimizer_search_depth > `0`
    - use `optimizer_search_depth`
- optimizer_search_depth == `0`
    - `join->tables - join->const_tables` <=  `7`
        - use `exhaustive` for small number of tables
    - use `greedy search`  
    
    
#### `Optimize_table_order::choose_table_order()`

##### 1. only `const table`
```
join->best_read= 1.0;
join->best_rowcount= 1;
return
```

##### 2. get `join_tables` orders
###### 2.1 semi-join materialization nest 
###### 2.2 `straight_join` hint
> keep tables in the order they were specified in the query

###### 2.3 other tables
1. const table in the front
2. other table order by table rows 

##### 3. compute `cost`

###### 3.1 `straight_join` | `optimize_straight_join`
```c
/* Find the best access method from 's' to the current partial plan */
POSITION  loose_scan_pos;
best_access_path(s, join_tables, idx, false, record_count,
                 position, &loose_scan_pos);

/* compute the cost of the new plan extended with 's' */
record_count*= position->records_read;
read_time+=    position->read_time;
read_time+=    record_count * ROW_EVALUATE_COST;
position->set_prefix_costs(read_time, record_count);
```     
###### 3.2 `else` | `greedy_search`
>  hybrid greedy/exhaustive search

##### 4 Fix `semi-join`  and `final cost` calculation `fix_semijoin_strategies()`
  
####  Decides between `EXISTS` and `materialization` `decide_subquery_strategy()`

#### Generate an `execution plan` from the found optimal join order
`join->get_best_combination()`


###  Remove `distinct` if only `const` tables 

### UNLOCK const table

```c
for (uint i= 0; i < const_tables; i++)
  ct[i]= join_tab[i].table;
mysql_unlock_some_tables(thd, ct, const_tables);
```

###  Perform the same optimization on field evaluation for all join conditions.
```c
for (JOIN_TAB *tab= join_tab + const_tables; tab < join_tab + tables ; tab++)
  {
    if (tab->on_expr_ref && *tab->on_expr_ref)
    {
      *tab->on_expr_ref= substitute_for_best_equal_field(*tab->on_expr_ref,
                                                         tab->cond_equal,
                                                         map2table);
      (*tab->on_expr_ref)->update_used_tables();
    }
  }
```


###  `drop_unused_derived_keys` for materialized table

###   Set access methods for the tables of a query plan `set_access_methods`
> sql/sql_select.cc

 - There is `no key` selected (`JT_ALL`)
 - Loose scan `semi-join` strategy is selected (`JT_ALL`)
 - A `ref key` can be used (`JT_REF`, `JT_REF_OR_NULL`, `JT_EQ_REF` or `JT_FT`)

```c
bool JOIN::set_access_methods()
{

  full_join= false;

  for (uint tableno= const_tables; tableno < tables; tableno++)
  {
    JOIN_TAB *const tab= join_tab + tableno;

    if (!tab->position)
      continue;

    DBUG_PRINT("info",("type: %d", tab->type));

    // Set preliminary join cache setting based on decision from greedy search
    tab->use_join_cache= tab->position->use_join_buffer ?
                           JOIN_CACHE::ALG_BNL : JOIN_CACHE::ALG_NONE;

    if (tab->type == JT_CONST || tab->type == JT_SYSTEM)
      continue;                      // Handled in make_join_statistics()

    Key_use *const keyuse= tab->position->key;
    if (tab->position->sj_strategy == SJ_OPT_LOOSE_SCAN)
    {
      DBUG_ASSERT(tab->keys.is_set(tab->position->loosescan_key));
      tab->type= JT_ALL; // @todo is this consistent for a LooseScan table ?
      tab->index= tab->position->loosescan_key;
     }
    else if (!keyuse)
    {
      tab->type= JT_ALL;
      if (tableno > const_tables)
       full_join= true;
    }
    else
    {
      if (create_ref_for_key(this, tab, keyuse, tab->prefix_tables()))
        DBUG_RETURN(true);
    }
   }

  DBUG_RETURN(false);
}
```

### Skip  order by `NULL` or `const_expression`


###  optimize `group by/distinct` without aggregate functions

- `DISTINCT` and/or `GROUP BY` references unique index
    - all key `parts of` an unique index
    - whatever `order`
- unique index `cannot` contain `NULL`s

### Plan refinement stage `make_join_readinfo`

### Check if we need to create a `temporary table`.
- We are using `DISTINCT` (simple distinct's are already optimized away)
- We are using an `ORDER BY` or `GROUP BY` on fields not in the `first` table
- We are using different ORDER BY and GROUP BY `orders`
- The user wants us to `buffer` the result.
    
```c
need_tmp= (
    (!plan_is_const() &&
      (
            (select_distinct || 
            !simple_order || 
            !simple_group) ||
            (group_list && order) ||
            MY_TEST(select_options & OPTION_BUFFER_RESULT)
       )
    ) ||
    (rollup.state != ROLLUP::STATE_NONE 
    && select_distinct)
);
```
