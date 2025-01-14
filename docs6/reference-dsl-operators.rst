..
    PLEASE DO NOT EDIT DIRECTLY. EDIT THE .rst.in FILE PLEASE.

DSL reference: operators
========================

Operator precedence
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Operators are listed in order of decreasing precedence, highest first.

.. code-block:: none

    Operators              Associativity
    ---------              -------------
    ()                     left to right
    **                    right to left
    ! ~ unary+ unary- &    right to left
    binary* / // %         left to right
    binary+ binary- .      left to right
    << >>                  left to right
    &                      left to right
    ^                      left to right
    |                      left to right
    < <= > >=              left to right
    == != =~ !=~           left to right
    &&                     left to right
    ^^                     left to right
    ||                     left to right
    ? :                    right to left
    =                      N/A for Miller (there is no $a=$b=$c)

Operator and function semantics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Functions are often pass-throughs straight to the system-standard Go libraries.

* The ``min`` and ``max`` functions are different from other multi-argument functions which return null if any of their inputs are null: for ``min`` and ``max``, by contrast, if one argument is absent-null, the other is returned. Empty-null loses min or max against numeric or boolean; empty-null is less than any other string.

* Symmetrically with respect to the bitwise OR, XOR, and AND operators ``|``, ``^``, ``&``, Miller has logical operators ``||``, ``^^``, ``&&``: the logical XOR not existing in Go.

* The exponentiation operator ``**`` is familiar from many languages.

* The regex-match and regex-not-match operators ``=~`` and ``!=~`` are similar to those in Ruby and Perl.

