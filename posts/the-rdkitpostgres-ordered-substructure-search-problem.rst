.. title: About the RDKit/Postgres Ordered Substructure Search Problem
.. slug: the-rdkitpostgres-ordered-substructure-search-problem
.. date: 2021-12-29 11:55:06 UTC+01:00
.. tags: RDKit,PostgreSQL
.. category: 
.. link: 
.. description: 
.. type: text

.. image:: /images/202112_t01.png
    :alt: another unrelated image of a festive cat
    :align: left

Last summer Richard Apodaca wrote a `blog post <https://depth-first.com/articles/2021/08/11/the-rdkit-postgres-ordered-substructure-search-problem/>`_
about using the RDKit PostgreSQL cartridge to support substructure search queries, which included a detailed discussion of how the performances of the
same query could significantly degrade when the substructure match constraint was combined with an "order by" clause, referencing another column from
the same table.

I couldn't find the time to actually read that post until much later, but during this break I finally found a moment to reproduce the reported
behavior, and also try to understand why it happens.

Very conveniently, the original blog post includes a complete description of how the test database was configured. Maybe any random subset of compounds from
ChEMBL could also work, but the instructions still saved me some time, and helped ensure that my setup was reasonably similar [#f0]_.

The tests use a single table created with the statement below:

.. code:: sql

    CREATE TABLE molecules (id SERIAL PRIMARY KEY,
                            mol mol NOT NULL,
                            mw NUMERIC NOT NULL,
                            fp bit(1024)
                        );

and filled with the molecule and molecular weight computed by the RDKit cartridge on the first 100000 [#f1]_ records from the downloaded eMolecules dataset.
On this table, a gist index is created on the `mol` column, and a regular btree index is created on the `mw` column.

The reference test query consists in selecting the molecules matching a search for a benzene/phenyl substructure, and returning the first 25 hits:

.. code:: sql
    
    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'c1ccccc1' LIMIT 25;
    +---------------------------------------------------------+---------+
    | mol                                                     | mw      |
    |---------------------------------------------------------+---------|
    | Cc1ccc(OCCCCl)cc1                                       | 184.666 |
    | O=C(CCc1ccccc1)c1ccc(O)cc1                              | 226.275 |
    ...
    +---------------------------------------------------------+---------+
    SELECT 25
    Time: 0.011s

Searching for a very common substructure limits the ability for the gist index implementation to prune the search tree, and may therefore require the
query execution to scan a larger portion of the index to identify the matching rows in the table. But in this case, things are actually made easier by the
limit clause, because the index scan ends as soon as the first 25 matching records are found, which is likely to happen quite soon, exactly because phenyl
rings are so frequently present in many compounds.

On the other hand, when a more selective query is performed, the index scan may happen to return a very small number of rows, and the query is again
quite fast:

.. code:: sql
    
    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'C(=S)Cl' LIMIT 25;
    +---------------------+---------+
    | mol                 | mw      |
    |---------------------+---------|
    | Cc1ccc(OC(=S)Cl)cc1 | 186.663 |
    +---------------------+---------+
    SELECT 1
    Time: 0.008s

But, even if substructure queries that include a limit clause tend to be less challenging, some problems become anyway apparent if an ordering clause
is added:

.. code:: sql

    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'c1ccccc1' order by mw limit 25;
    +-------------------------+---------+
    | mol                     | mw      |
    |-------------------------+---------|
    | Cc1cccc(C)c1            | 106.168 |
    | Cc1ccc(C)cc1            | 106.168 |
    ...
    +-------------------------+---------+
    SELECT 25
    Time: 1.963s

Requiring the result records to be ordered by molecular weight made the same query almost 200x slower. 

I think the problem is that with the available table and indices, the query planner is confronted with two main options. It could either use the index on
the `mw` column to scan the table in increasing molecular weight order, and explicitly check the query constraint `mol@>'c1ccccc1'` to filter out the
undesired records one by one, or it could use the index on the `mol` column to select the matching records, and then sort these records by molecular weight.

Because the index on the `mol` column has the potential for reducing the number of evaluated records, this is the plan that normally applies, as confirmed by
looking at the output from `EXPLAIN ANALYZE`. The bottleneck, I think is not as much in whether the output records are efficiently sorted, but rather that for
a substructure query with a very poor selectivity (like `c1ccccc1`), the index scan is required to process a very large fraction of the search tree associated
to the `mol` column (in this specific case ~66K hits are initially selected from the index, about 2/3 of the overall data - a sequential scan of the
unindexed column would be probably faster).

More selective queries can still perform fine also when the query includes the `order by` query (the `mol` index search tree is efficiently pruned,
and a small number of records is selected), but searching for very common substructures can be very inefficient, and the limit clause in this case doesn't
make things easier, because it only applies after the results are ordered.

The suggested workaround of turning off the use of explicit sorting steps in the query planner can indeed help:

.. code:: sql

    ric@/tmp:emolecules> set enable_sort=off;

    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'c1ccccc1' order by mw limit 25;
    +-------------------------+---------+
    | mol                     | mw      |
    |-------------------------+---------|
    | Cc1ccc(C)cc1            | 106.168 |
    | Cc1cccc(C)c1            | 106.168 |
    ...
    +-------------------------+---------+
    SELECT 25
    Time: 0.014s

but I am afraid it could be just because in practice it prevents the planner from using the index on the `mol` column, and not using this index may impact
negatively other queries that would normally show a high selectivity:

.. code:: sql

    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'C(=S)Cl' order by mw limit 25;
    +---------------------+---------+
    | mol                 | mw      |
    |---------------------+---------|
    | Cc1ccc(OC(=S)Cl)cc1 | 186.663 |
    +---------------------+---------+
    SELECT 1
    Time: 2.652s

My impression is that one main limitation might be in the RDKit cartridge implementation, and more specifically in the selectivity estimation function
associated to the substructure operator `@>`. If this function could return a better estimate of the selectivity associated to the given substructure query,
the query planner could make a better decision about whether to use the index and how (maybe an interesting project for the next year?).

In the meantime, I think a two-column index [#f2]_ could actually help the query planner leverage the information from both the `mw` and `mol` columns
at once.

The PostgreSQL gist index implementation provides support for multi-column indices, but a suitable "operator class" is required to be available for the
indexed data types. The RDKit cartridge provides this operator class for the molecule, fingerprint and reaction data types it implements, and similarly,
the `btree_gist` [#f3]_ extension provides gist operator classes for most numerical types that are available for PostgreSQL.

.. code:: sql

    ric@/tmp:emolecules> CREATE EXTENSION btree_gist;

However, in order to leverage the gist index in supporting the `ORDER BY` query, we need to change the data type to use one whose operator class defines
the distance operator and function that are required for nearest neighbor queries. This operator (`<->`) is not implemented for the `NUMERIC` data type [#f4]_,
but it's available for the other numerical primitive types, including the floating point ones:

.. code:: sql

    ric@/tmp:emolecules> ALTER TABLE molecules ALTER COLUMN mw TYPE real;

This way, we can then create a combined gist index on both the `mol` and `mw` columns:
    
.. code:: sql

    ric@/tmp:emolecules> CREATE INDEX molecules_mol_mw on molecules USING gist(mol, mw);

And finally, a small change needs to apply to the executed query, in order to use the new index in the `ORDER BY` clause:

.. code:: sql

    ric@/tmp:emolecules> SELECT mol, mw FROM molecules WHERE mol@>'c1ccccc1' order by mw <-> 0.0 limit 25;
    +-------------------------+---------+
    | mol                     | mw      |
    |-------------------------+---------|
    | Cc1cccc(C)c1            | 106.168 |
    | Cc1ccc(C)cc1            | 106.168 |
    ...
    +-------------------------+---------+
    SELECT 25
    Time: 0.013s

The output from `EXPLAIN ANALYZE` may help confirm that the two-column index is actually used:

.. code:: sql

    ric@/tmp:emolecules> EXPLAIN ANALYZE SELECT mol, mw FROM molecules WHERE mol@>'c1ccccc1' order by mw <-> 0.0 limit 25;
    +-----------------------------------------------------------------------------------------------------------------------------------------+
    | QUERY PLAN                                                                                                                              |
    |-----------------------------------------------------------------------------------------------------------------------------------------|
    | Limit  (cost=0.28..106.84 rows=25 width=409) (actual time=2.399..4.066 rows=25 loops=1)                                                 |
    |   ->  Index Scan using molecules_mol_mw on molecules  (cost=0.28..426.53 rows=100 width=409) (actual time=2.396..4.057 rows=25 loops=1) |
    |         Index Cond: (mol @> 'c1ccccc1'::mol)                                                                                            |
    |         Order By: (mw <-> '0'::real)                                                                                                    |
    | Planning Time: 0.202 ms                                                                                                                 |
    | Execution Time: 4.146 ms                                                                                                                |
    +-----------------------------------------------------------------------------------------------------------------------------------------+


.. rubric:: Footnotes

.. [#f0] I was required to use a more current version of the eMolecules data (version 2021-07-01 is no longer available for download, so I used version
    2022-01-01), and I also used PostgreSQL 14.1 from conda-forge, with a custom build of the RDKit cartridge (based on the current RDKit release
    2021.09.3). I believe the slightly different configuration didn't introduce any significant difference compared to the original post.

.. [#f1] The size of this test database is indeed small, but because the executed queries are limited to only select a small number of hits, it's
    already sufficient to detect when there are any issues with the generated query plan. Also because this observation was in agreement with the 
    findings in the original post, I haven't spent a big amount of time exploring the dependencies from the database size, but a quick test with a
    dataset about 10x larger provided about the same results.

.. [#f2] The option of using a two-column index based on `btree_gist` was also already considered in the original post, but for some reason it was
    reported to be ineffective.

.. [#f3] The `btree_gist` extension is part of the PostgreSQL source code distribution and should be most often available with any installation.

.. [#f4] `NUMERIC` is an arbitrary precision decimal data type. It's recommended for storing monetary amounts and other quantities where exactness
    is required. I'm not sure its use for storing the molecular weight was anyway intended.
