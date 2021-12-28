.. title: Extending ChEMBL with ChemicaLite (and of course RDKit)
.. slug: extending-chembl-with-chemicalite-and-of-course-rdkit
.. date: 2020-12-29 21:42:33 UTC+01:00
.. tags: ChEMBL,RDKit,ChemicaLite,SQLite
.. category: 
.. link: 
.. description: 
.. type: text

This blog post collects some draft notes about how to use `ChemicaLite <https://github.com/rvianello/chemicalite>`_ 
to integrate some chemistry logic directly into the SQLite version of the `ChEMBL <https://www.ebi.ac.uk/chembl/>`_ database [#f1]_.

.. attention::

    This post is now quite old, the overall workflow is still informative, but several details in the ChemicaLite API have changed. Please consider
    referring to the similar `example <https://github.com/rvianello/chemicalite/blob/master/examples/setup_chembl.sql>`_ script that is part of
    the source code distribution, and in general to the documentation that is also available from
    `Read the Docs <https://chemicalite.readthedocs.io/en/latest/>`_.

.. image:: /images/202012_t01.png
    :alt: an unrelated image of a festive cat
    :align: right

.. code-block:: shell

    $ # you can download the ChEMBL database from ftp://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBLdb/latest/
    $ tar xvf ./chembl_27_sqlite.tar.gz

At this time the easiest way (for linux and osx [#f2]_ users) to get ChemicaLite probably consists in installing the experimental
conda packages I uploaded to `anaconda.org <https://anaconda.org/ric/chemicalite>`_, together with the RDKit package from
conda-forge.

.. code-block:: shell

    $ conda create -n chembl-chemicalite -c conda-forge -c ric/label/dev rdkit chemicalite
    $ . activate chembl-chemicalite

SQLite drivers exist for basically all programming languages and environments, but to keep things simple, I will just
use some SQL code interactively in the SQLite shell:

.. code-block::

    (chembl-chemicalite) $ sqlite3 ./chembl_27/chembl_27_sqlite/chembl_27.db 
    SQLite version 3.34.0 2020-12-01 16:14:00
    Enter ".help" for usage hints.
    sqlite> 

First, the ChemicaLite extension needs to be loaded (because the extension is provided as a runtime loadable module, this
operation is required on every new connection to the database, otherwise SQLite will complain that functions and types implemented
in the extension are not known/available). Please note that loading extensions is automatically enabled in
the SQLite shell. If you use a different driver or API, you may need to explicitly
`enable this operation <https://sqlite.org/loadext.html>`_.

.. code-block:: sql

    sqlite> .load chemicalite

For reference (and to quickly smoke-test the loaded extension) one can briefly verify which versions of ChemicaLite and RDKit are
available:

.. code-block:: sql

    sqlite> SELECT chemicalite_version(), rdkit_version();
    2020.12.5|2020.09.3

Once the extension is loaded, bindings to the RDKit toolkit become available as SQL functions and it is for example
possible to compute molecular descriptors directly on the compounds in the ChEMBL tables. However, it is usually
convenient to permanently extend the database schema with a table of RDKit molecules:

.. code-block:: sql

    sqlite> CREATE TABLE rdkit_molecules(id INTEGER PRIMARY KEY, molregno BIGINT NOT NULL, mol MOL);
    sqlite> INSERT INTO rdkit_molecules (molregno, mol) SELECT molregno, mol(canonical_smiles) FROM compound_structures;

This way chemical constraints (e.g. substructure searches) can be easily integrated in queries applying to the ChEMBL data:

.. code-block:: sql

    sqlite> SELECT cs.molregno, cs.canonical_smiles FROM
       ...> compound_structures AS cs JOIN rdkit_molecules AS rm USING(molregno)
       ...> WHERE mol_is_substruct(rm.mol, 'c1cccc2c1nncc2') LIMIT 5
       ...> ;
    9830|CC(C)Sc1ccc(CC2CCN(C3CCN(C(=O)c4cnnc5ccccc45)CC3)CC2)cc1
    37351|Cc1cccc(NC(=O)Nc2ccc3nnccc3c2)c1
    47898|COC(=O)c1c(C(F)(F)F)[nH]c2c(O)cc3c(c12)[C@H](CCl)C(c1cc2cc(NC(=O)c4cc5c(OC)c(OC)c(OC)cc5nn4)ccc2[nH]1)N3C=O
    62519|N#Cc1nnc2ccccc2c1NCCC(c1ccccc1)c1ccccc1
    109577|CNC(=O)[C@H](CC(C)C)C[C@H](O)[C@H](Cc1ccccc1)NC(=O)c1cnnc2ccccc12

.. code-block:: sql

    sqlite> SELECT COUNT(*) FROM rdkit_molecules WHERE mol_is_substruct(mol, 'c1cccc2c1nncc2');
    485

The above queries are nice and simple, but without a proper (chemistry aware) index, the whole ``rdkit_molecules`` table is
scanned sequentially, the substructure constraint is checked explicitly on every molecule, resulting in some poor performances.

Differently from other database engines, SQLite doesn't support creating a custom index directly on the ``mol`` column. 
Instead, SQLite supports a `virtual table <https://sqlite.org/vtab.html>`_ mechanism, which allows wrapping the index data
structure with a table-like interface. This way, an indexed search can be performed by joining the queried column with the
virtual table that implements the index.

ChemicaLite provides an ``rdtree`` virtual table that implements an index structure for binary fingerprints. For
substructure queries, an index based on the ``rdtree`` and the RDKit pattern fingerprint can be created with the following
two statements:

.. code-block:: sql

    sqlite> CREATE VIRTUAL TABLE rdkit_struct_index USING rdtree(id, s bits(2048), OPT_FOR_SUBSET_QUERIES);
    sqlite> INSERT INTO rdkit_struct_index (id, s) SELECT id, mol_bfp_signature(mol, 2048) FROM rdkit_molecules WHERE mol IS NOT NULL;

The example substructure queries above can be refactored to leverage this index table, which will allow them to run significantly
faster [#f3]_ :

.. code-block:: sql

    sqlite> WITH matching AS (
    ...>     SELECT rm.molregno, rm.mol FROM rdkit_molecules AS rm JOIN rdkit_struct_index AS idx USING (id)
    ...>     WHERE idx.id MATCH rdtree_subset(mol_bfp_signature('c1cccc2c1nncc2', 2048))
    ...> )
    ...> SELECT cs.molregno, cs.canonical_smiles FROM compound_structures AS cs JOIN matching AS m USING (molregno)
    ...> WHERE mol_is_substruct(m.mol, 'c1cccc2c1nncc2') LIMIT 5
    ...> ;
    1216581|Cc1cnnc2ccccc12
    1233534|O=C(O)c1cnnc2ccccc12
    947201|Cc1cc2c(N)cccc2nn1
    501513|Cc1cc2cc(O)c(O)cc2nn1
    1295924|O=[N+]([O-])c1cccc2nnccc12

.. code-block:: sql

    sqlite> SELECT COUNT(*) FROM
    ...> rdkit_molecules AS rm JOIN rdkit_struct_index AS idx USING (id)
    ...> WHERE mol_is_substruct(rm.mol, 'c1cccc2c1nncc2')
    ...> AND idx.id MATCH rdtree_subset(mol_bfp_signature('c1cccc2c1nncc2', 2048));
    485

Very similarly, an index can be created to support similarity queries, in this case using Morgan binary fingerprints:

.. code-block:: sql

    sqlite> CREATE VIRTUAL TABLE rdkit_morgan_index USING rdtree(id, s bits(2048), OPT_FOR_SIMILARITY_QUERIES);
    sqlite> INSERT INTO rdkit_morgan_index (id, s) SELECT id, mol_morgan_bfp(mol, 2, 2048) FROM rdkit_molecules WHERE mol IS NOT NULL;

.. code-block:: sql

    sqlite> WITH similar AS (
    ...>     SELECT rm.molregno, idx.s FROM rdkit_molecules AS rm JOIN rdkit_morgan_index AS idx USING (id)
    ...>     WHERE idx.s MATCH rdtree_tanimoto(mol_morgan_bfp('Cc1ccc2nc(-c3ccc(NC(C4N(C(c5cccs5)=O)CCC4)=O)cc3)sc2c1', 2, 2048), 0.6)
    ...> )
    ...> SELECT
    ...>     cs.molregno, cs.canonical_smiles,
    ...>     bfp_tanimoto(mol_morgan_bfp('Cc1ccc2nc(-c3ccc(NC(C4N(C(c5cccs5)=O)CCC4)=O)cc3)sc2c1', 2, 2048), similar.s) AS similarity
    ...> FROM compound_structures AS cs JOIN similar USING (molregno)
    ...> ORDER BY similarity DESC
    ...> ;
    472512|Cc1ccc2nc(-c3ccc(NC(=O)C4CCN(C(=O)c5cccs5)CC4)cc3)sc2c1|0.761194029850746
    471317|Cc1ccc2nc(-c3ccc(NC(=O)C4CCCN(S(=O)(=O)c5cccs5)C4)cc3)sc2c1|0.64
    471461|Cc1ccc2nc(-c3ccc(NC(=O)C4CCN(S(=O)(=O)c5cccs5)CC4)cc3)sc2c1|0.621621621621622
    1032469|O=C(Nc1nc2ccc(Cl)cc2s1)[C@@H]1CCCN1C(=O)c1cccs1|0.614285714285714
    471319|Cc1ccc2nc(-c3ccc(NC(=O)C4CCN(S(=O)(=O)c5cccs5)C4)cc3)sc2c1|0.613333333333333
    751668|COc1ccc2nc(NC(=O)[C@@H]3CCCN3C(=O)c3cccs3)sc2c1|0.611111111111111
    740754|Cc1ccc(NC(=O)C2CCCN2C(=O)c2cccs2)cc1C|0.606060606060606

I collected the modifying queries from this post into a small script (please note that the script doesn't include
any error handling or graceful recovery from any error conditions):

.. gist:: 87a6173b033c9ea0ee206a9a7b9bd042

This script may take some time to execute (about 50' on my home laptop, more than a half of which apparently spent
computing and filling the SSS index). With the additional tables, the database file size (ignoring any storage temporarily used by SQLite
while applying the changes) grows by about 2 GB.

.. rubric: Footnotes

.. [#f1] The attentive reader will notice that this post is late by almost
    `5 years <http://chembl.blogspot.com/2016/03/chembl-db-on-sqlite-is-that-even.html>`_. But better
    late than never :-)
.. [#f2] These packages are built using the `GitHub Actions <https://github.com/features/actions>`_ infrastructure (thank you GitHub for
    making it available). Please note that I am not an osx user, and the osx packages are only unit-tested as part of the build workflow.
.. [#f3] You may notice that the ``compound_structures`` records returned by the `LIMIT 5` clause do not match those from the earlier,
    supposedly equivalent, example. This is because in this second query the ordering of the selected records is determined by the
    traversal of the index tree. An additional `ORDER BY molregno` clause would have ensured consistent results.
