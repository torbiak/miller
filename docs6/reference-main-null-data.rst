..
    PLEASE DO NOT EDIT DIRECTLY. EDIT THE .rst.in FILE PLEASE.

Reference: null data
================================================================

One of Miller's key features is its support for **heterogeneous** data.  For example, take ``mlr sort``: if you try to sort on field ``hostname`` when not all records in the data stream *have* a field named ``hostname``, it is not an error (although you could pre-filter the data stream using ``mlr having-fields --at-least hostname then sort ...``).  Rather, records lacking one or more sort keys are simply output contiguously by ``mlr sort``.

Miller has two kinds of null data:

* **Empty (key present, value empty)**: a field name is present in a record (or in an out-of-stream variable) with empty value: e.g. ``x=,y=2`` in the data input stream, or assignment ``$x=""`` or ``@x=""`` in ``mlr put``.

* **Absent (key not present)**: a field name is not present, e.g. input record is ``x=1,y=2`` and a ``put`` or ``filter`` expression refers to ``$z``. Or, reading an out-of-stream variable which hasn't been assigned a value yet, e.g.  ``mlr put -q '@sum += $x; end{emit @sum}'`` or ``mlr put -q '@sum[$a][$b] += $x; end{emit @sum, "a", "b"}'``.

You can test these programatically using the functions ``is_empty``/``is_not_empty``, ``is_absent``/``is_present``, and ``is_null``/``is_not_null``. For the last pair, note that null means either empty or absent.

Rules for null-handling:

* Records with one or more empty sort-field values sort after records with all sort-field values present:

.. code-block:: none
   :emphasize-lines: 1-1

    mlr cat data/sort-null.dat
    a=3,b=2
    a=1,b=8
    a=,b=4
    x=9,b=10
    a=5,b=7

.. code-block:: none
   :emphasize-lines: 1-1

    mlr sort -n  a data/sort-null.dat
    a=1,b=8
    a=3,b=2
    a=5,b=7
    a=,b=4
    x=9,b=10

.. code-block:: none
   :emphasize-lines: 1-1

    mlr sort -nr a data/sort-null.dat
    a=,b=4
    a=5,b=7
    a=3,b=2
    a=1,b=8
    x=9,b=10

* Functions/operators which have one or more *empty* arguments produce empty output: e.g.

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=2,y=3' | mlr put '$a=$x+$y'
    x=2,y=3,a=5

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=,y=3' | mlr put '$a=$x+$y'
    x=,y=3,a=

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=,y=3' | mlr put '$a=log($x);$b=log($y)'
    x=,y=3,a=,b=1.0986122886681096

with the exception that the ``min`` and ``max`` functions are special: if one argument is non-null, it wins:

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=,y=3' | mlr put '$a=min($x,$y);$b=max($x,$y)'
    x=,y=3,a=3,b=

* Functions of *absent* variables (e.g. ``mlr put '$y = log10($nonesuch)'``) evaluate to absent, and arithmetic/bitwise/boolean operators with both operands being absent evaluate to absent. Arithmetic operators with one absent operand return the other operand. More specifically, absent values act like zero for addition/subtraction, and one for multiplication: Furthermore, **any expression which evaluates to absent is not stored in the left-hand side of an assignment statement**:

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=2,y=3' | mlr put '$a=$u+$v; $b=$u+$y; $c=$x+$y'
    x=2,y=3,b=3,c=5

.. code-block:: none
   :emphasize-lines: 1-1

    echo 'x=2,y=3' | mlr put '$a=min($x,$v);$b=max($u,$y);$c=min($u,$v)'
    x=2,y=3,a=2,b=3

* Likewise, for assignment to maps, **absent-valued keys or values result in a skipped assignment**.

The reasoning is as follows:

* Empty values are explicit in the data so they should explicitly affect accumulations: ``mlr put '@sum += $x'`` should accumulate numeric ``x`` values into the sum but an empty ``x``, when encountered in the input data stream, should make the sum non-numeric. To work around this you can use the ``is_not_null`` function as follows: ``mlr put 'is_not_null($x) { @sum += $x }'``

* Absent stream-record values should not break accumulations, since Miller by design handles heterogenous data: the running ``@sum`` in ``mlr put '@sum += $x'`` should not be invalidated for records which have no ``x``.

* Absent out-of-stream-variable values are precisely what allow you to write ``mlr put '@sum += $x'``. Otherwise you would have to write ``mlr put 'begin{@sum = 0}; @sum += $x'`` -- which is tolerable -- but for ``mlr put 'begin{...}; @sum[$a][$b] += $x'`` you'd have to pre-initialize ``@sum`` for all values of ``$a`` and ``$b`` in your input data stream, which is intolerable.

* The penalty for the absent feature is that misspelled variables can be hard to find: e.g. in ``mlr put 'begin{@sumx = 10}; ...; update @sumx somehow per-record; ...; end {@something = @sum * 2}'`` the accumulator is spelt ``@sumx`` in the begin-block but ``@sum`` in the end-block, where since it is absent, ``@sum*2`` evaluates to 2. See also the section on :doc:`reference-dsl-errors`.

Since absent plus absent is absent (and likewise for other operators), accumulations such as ``@sum += $x`` work correctly on heterogenous data, as do within-record formulas if both operands are absent. If one operand is present, you may get behavior you don't desire.  To work around this -- namely, to set an output field only for records which have all the inputs present -- you can use a pattern-action block with ``is_present``:

.. code-block:: none
   :emphasize-lines: 1-1

    mlr cat data/het.dkvp
    resource=/path/to/file,loadsec=0.45,ok=true
    record_count=100,resource=/path/to/file
    resource=/path/to/second/file,loadsec=0.32,ok=true
    record_count=150,resource=/path/to/second/file
    resource=/some/other/path,loadsec=0.97,ok=false

.. code-block:: none
   :emphasize-lines: 1-1

    mlr put 'is_present($loadsec) { $loadmillis = $loadsec * 1000 }' data/het.dkvp
    resource=/path/to/file,loadsec=0.45,ok=true,loadmillis=450
    record_count=100,resource=/path/to/file
    resource=/path/to/second/file,loadsec=0.32,ok=true,loadmillis=320
    record_count=150,resource=/path/to/second/file
    resource=/some/other/path,loadsec=0.97,ok=false,loadmillis=970

.. code-block:: none
   :emphasize-lines: 1-1

    mlr put '$loadmillis = (is_present($loadsec) ? $loadsec : 0.0) * 1000' data/het.dkvp
    resource=/path/to/file,loadsec=0.45,ok=true,loadmillis=450
    record_count=100,resource=/path/to/file,loadmillis=0
    resource=/path/to/second/file,loadsec=0.32,ok=true,loadmillis=320
    record_count=150,resource=/path/to/second/file,loadmillis=0
    resource=/some/other/path,loadsec=0.97,ok=false,loadmillis=970

If you're interested in a formal description of how empty and absent fields participate in arithmetic, here's a table for plus (other arithmetic/boolean/bitwise operators are similar):

.. code-block:: none
   :emphasize-lines: 1-1

    mlr help type-arithmetic-info
    (+)        | 1          2.5        (absent)   (error)   
    ------     + ------     ------     ------     ------    
    1          | 2          3.5        1          (error)   
    2.5        | 3.5        5          2.5        (error)   
    (absent)   | 1          2.5        (absent)   (error)   
    (error)    | (error)    (error)    (error)    (error)   
