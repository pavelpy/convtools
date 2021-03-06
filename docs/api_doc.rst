.. _convtools_api_doc:

=================
convtools API Doc
=================

For the sake of conciseness, let's assume the following import statement is in place:

.. code-block:: python

 from convtools import conversion as c

This is an object which exposes public API.

.. _ref_c_base_conversion:

c.BaseConversion
================

 .. autoclass:: convtools.base.BaseConversion()
    :noindex:

 .. _ref_c_gen_converter:

 .. automethod:: convtools.base.BaseConversion.gen_converter
    :noindex:

 .. note::

  At first use ``.gen_converter(debug=True)`` everywhere to see the
  generated code

.. _ref_c_this:

c.this
======

 .. automethod:: convtools.conversion.this

 .. code-block:: python

  c.this().gen_converter(debug=True)
  # is compiled to
  def converter36_998(data_):
      return data_

.. _ref_c_input_arg:

c.input_arg
===========

 .. autoattribute:: convtools.conversion.input_arg

 .. autoclass:: convtools.base.InputArg()
  :noindex:

 .. automethod:: convtools.base.InputArg.__init__

 .. code-block:: python

  f = (c.this() + c.input_arg("x")).gen_converter(debug=True)
  # is compiled to
  def converter37_306(data_, *, x):
      return data_ + x
  # usage:
  assert f(10, x=700) == 710

**Example of changing the signature:**

 .. code-block:: python

  c.this().and_(
      c.input_arg("x").as_type(int)
  ).or_(None).gen_converter(
      debug=True,
      signature="x, data_",
  )
  # is compiled to
  def converter37_306(x, data_):
      return (data_ and vint61_679(x)) or None

.. _ref_c_naive:

c.naive
=======

 .. automethod:: convtools.conversion.naive()
 
 .. autoclass:: convtools.base.NaiveConversion()
    :noindex:

 .. automethod:: convtools.base.NaiveConversion.__init__(value)

**Example:**

.. code-block:: python

 # assume we have a map: 3rd party error code to verbose
 converter = c.naive({
     "99": "Invalid token - the user has changed password",
     "500": "try again later"
 }).item(c.item("error_code")).gen_converter()

 # is compiled to
 def converter209_358(data_):
    # v206_380 is the mapping exposed by c.naive 
    return v206_380[data_["error_code"]]

 converter({"error_code": "500"}) == "try again later"


.. _ref_c_wrapper:

c() wrapper
===========

 .. automethod:: convtools.conversion.__call__
 
 .. automethod:: convtools.base.ensure_conversion
    :noindex:
 
 .. note::
    It is used under the hood on every argument passed to any conversion.

**Example #1:**

.. code-block:: python

 converter = c({
  c.item("id"): c.item("name"),
 }).gen_converter()

 # is compiled to
 def converter42_484(data_):
     return {data_["id"]: data_["name"]}

 converter({"id": "uid", "name": "John"}) == {"uid": "John"}

**Example #2:**

.. code-block:: python

 converter = c(
     lambda x: f"{type(x).__name__}_{x}"
 ).call(c.item("value")).gen_converter()

 # is compiled to
 def converter42_484(data_):
     return vlambda80_37(data_["value"])

 converter(123) == "int_123"

**Example #2 + inline_expr (no unnecessary func call):**

.. code-block:: python

 converter = (
     c.inline_expr('''"%s_%s" % (type({x}).__name__, {x})''')
     .pass_args(x=c.item("value"))
     .gen_converter(debug=True)
 )({"value": 123}) == "int_123"

 # is compiled to
 def converter100_386(data_):
     return "%s_%s" % (type(data_["value"]).__name__, data_["value"])

 converter(123) == "int_123"

**Example #2 without inline_expr (no unnecessary func call):**

.. code-block:: python

 converter = (
     c("{}_{}").call_method(
         "format",
         c.call_func(type, c.item("value")).attr("__name__"),
         c.item("value"),
     )
     .gen_converter(debug=True)
 )({"value": 123}) == "int_123"

 # is compiled to
 def converter100_386(data_):
     return "{}_{}".format(
         getattr(vtype127_859(data_["value"]), "__name__"), data_["value"]
     )
 converter(123) == "int_123"


.. _ref_c_item:

c.item
======

 .. autoattribute:: convtools.conversion.item
 
 .. autoclass:: convtools.base.GetItem()
    :noindex:

 .. automethod:: convtools.base.GetItem.__init__

**Example:**

.. code-block:: python

 c.item("key1", 1, c.item("key2"), default=-1).gen_converter()

generates:

.. code-block:: python

 def get_or_default208_222(obj, default):
     try:
         return obj['key1'][1][obj['key2']]
     except (TypeError, KeyError, IndexError, AttributeError):
         return default

 def converter208_771(data):
    return get_or_default208_222(data, -1)


**Also since conversions have methods, the following expressions
are equivalent:**

.. code-block:: python

 c.item("key1", "key2", 1).gen_converter()
 c.item("key1").item("key2").item(1).gen_converter()

and generate same code, like:

.. code-block:: python

 def converter210_775(data):
     return data["key1"]["key2"][1]


.. _ref_c_attr:

c.attr
======

``c.attr`` works the same as :py:obj:`convtools.base.GetItem`, but for
getting attributes.

.. autoattribute:: convtools.conversion.attr

.. autoclass:: convtools.base.GetAttr()
   :noindex:

.. automethod:: convtools.base.GetAttr.__init__

.. code-block:: python

 c.attr("user", c.input_arg("field_name")).gen_converter()

generates:

.. code-block:: python

 def converter157_386(data_, *, field_name):
    return getattr(data_.user, field_name)


.. _ref_c_calls:

call_func, call and call_method
===============================

There are 3 different actions related to calling something:

**1. c.call -- Calling __call__ of an input**

 .. automethod:: convtools.base.BaseConversion.call
    :noindex:

**2. c.call_func -- Regular function calling, passing a function to call**

 .. autoattribute:: convtools.conversion.call_func
 
 .. autofunction:: convtools.base.CallFunc
    :noindex:

**3. (...).call_method -- Calling a method of an input**

 .. automethod:: convtools.base.BaseConversion.call_method
    :noindex:


**Examples:**

.. code-block:: python

 c.item("key1").call_method("replace", "abc", "").gen_converter()
 # generates
 def converter210_134(data):
     return data["key1"].replace("abc", "")

.. code-block:: python

 # also the following is available
 c.call_func(str.replace, c.item("key1"), "abc", "").gen_converter()
 c.item("key1").attr("replace").call("abc", "").gen_converter()


.. _ref_c_inline_expr:

c.inline_expr
=============

``c.inline_expr`` is used to inline a raw python expression into
the code of resulting conversion, to avoid function call overhead.

.. autoattribute:: convtools.conversion.inline_expr

.. autoclass:: convtools.base.InlineExpr()
   :noindex:

.. automethod:: convtools.base.InlineExpr.__init__

.. automethod:: convtools.base.InlineExpr.pass_args
   :noindex:

.. code-block:: python

 c.inline_expr("{0} + 1").pass_args(c.item("key1")).gen_converter()
 # same
 c.inline_expr("{number} + 1").pass_args(number=c.item("key1")).gen_converter()

 # gets compiled into
 def converter206_372(data):
    try:
        return data["key1"] + 1


.. _ref_c_operators:

Conversion methods/operators
============================

Wraps conversions with logical / math / comparison operators

.. code-block:: python
 
 # logical
 c.not_(conversion)
 c.or_(*conversions)
 c.and_(*conversions)

 c.this().or_(*conversions)        # OR c.this() | ...
 c.this().and_(*conversions)       # OR c.this() & ...
 c.this().not_()                   # OR ~c.this()
 c.this().is_(conversion)
 c.this().is_not(conversion)
 c.this().in_(conversion)
 c.this().not_in(conversion)

 # comparisons
 c.this().eq(conversion)           # OR c.this() == ...
 c.this().not_eq(conversion)       # OR c.this() != ...
 c.this().gt(conversion)           # OR c.this() > ...
 c.this().gte(conversion)          # OR c.this() >= ...
 c.this().lt(conversion)           # OR c.this() < ...
 c.this().lte(conversion)          # OR c.this() <= ...

 # math
 c.this().neg()                    # OR -c.this()
 c.this().add(conversion)          # OR c.this() + ...
 c.this().mul(conversion)          # OR c.this() * ...
 c.this().sub(conversion)          # OR c.this() - ...
 c.this().div(conversion)          # OR c.this() / ...
 c.this().mod(conversion)          # OR c.this() % ...
 c.this().floor_div(conversion)    # OR c.this() // ...


.. _ref_c_collections:
 
Collections
===========
Converts an input into a collection, same is achievable by using
`c() wrapper`_, see below:

 * **c.list or c([])**

  .. autoattribute:: convtools.conversion.list

  .. autoclass:: convtools.base.List()
     :noindex:

  .. automethod:: convtools.base.List.__init__

 * **c.tuple or c(())**

  .. autoattribute:: convtools.conversion.tuple

  .. autoclass:: convtools.base.Tuple()
     :noindex:

  .. automethod:: convtools.base.Tuple.__init__

 * **c.set or c(set())**

  .. autoattribute:: convtools.conversion.set
  
  .. autoclass:: convtools.base.Set()
     :noindex:

  .. automethod:: convtools.base.Set.__init__

 * **c.dict or c({})**

  .. autoattribute:: convtools.conversion.dict
  
  .. autoclass:: convtools.base.Dict()
     :noindex:

  .. automethod:: convtools.base.Dict.__init__

.. code-block:: python

 # passing a list; the same works with tuples, sets,
 converter = c([
     "val_baked_into_conversion",
     c.item(2),
     c.item(1),
     # nesting works too, since every argument is being passed
     # through the `c() wrapper`, so no additional wrapping is needed
     (c.item(0), c.item(1)),
     c.input_arg("kwarg1")
 ]).gen_converter() # <=> c.list(...)

 converter([0, 1, 2], kwarg1=777) == ["val_baked_into_conversion", 2, 1, 777]

 # the code above generates the following:
 def converter215_171(data, *, kwarg1):
     return [
         "val_baked_into_conversion",
         data[2],
         data[1],
         (data[0], data[1],),
         kwarg1,
     ]

 # dicts either
 converter = c({
     c.item(0): c.item(1),
     c.item(1): c.item(0),
     '3': c.item(0),
 }).gen_converter()
 converter(['1', '2']) == {'1': '2', '2': '1', '3': '1'}

 # the code above generates the following:
 def converter216_250(data):
     return {data[0]: data[1], data[1]: data[0], "3": data[0]}


.. _ref_comprehensions:

Comprehensions
==============

.. autoclass:: convtools.base.BaseComprehensionConversion()

  .. automethod:: convtools.base.BaseComprehensionConversion.__init__
  .. automethod:: convtools.base.BaseComprehensionConversion.filter
  .. automethod:: convtools.base.BaseComprehensionConversion.sort


.. autoattribute:: convtools.conversion.list_comp

 .. autoclass:: convtools.base.ListComp()
   :noindex:

.. autoattribute:: convtools.conversion.tuple_comp

 .. autoclass:: convtools.base.TupleComp()
  :noindex:

.. autoattribute:: convtools.conversion.set_comp

 .. autoclass:: convtools.base.SetComp()
  :noindex:

.. autoattribute:: convtools.conversion.dict_comp

 .. autoclass:: convtools.base.DictComp()
  :noindex:
 .. automethod:: convtools.base.DictComp.__init__

**Examples:**

.. code-block:: python

 c.list_comp(c.item("name")).gen_converter()
 # equivalent to
 [i208_702["name"] for i208_702 in data]

 c.dict_comp(c.item("id"), c.item("name")).gen_converter()
 # equivalent to
 {i210_336["id"]: i210_336["name"] for i210_336 in data}

**Support of filtering & sorting:**

.. code-block:: python

 c.list_comp(
     (c.item("id"), c.item("name"))
 ).filter(
     c.item("age").gte(18)
 ).sort(
     key=lambda t: t[0],
     reverse=True,
 ).gen_converter()
 # equivalent to
 return sorted(
     [
         (i210_700["id"], i210_700["name"],)
         for i210_700 in data
         if (i210_700["age"] >= 18)
     ],
     key=vlambda216_644,
     reverse=True,
 )


.. _ref_c_filter:

c.filter
========

.. automethod:: convtools.conversion.filter
   :noindex:
.. automethod:: convtools.base.BaseConversion.filter
   :noindex:

Iterate an input filtering out items where the conversion resolves to False

.. code-block:: python

 c.filter(c.item("age").gte(18), cast=None).gen_converter()
 # equivalent to the following generator
 (i211_213 for i211_213 in data if (i211_213["age"] >= 18))

 # cast also supports: list, set, tuple or any callable to wrap generator
 c.filter(
     c.item("age").gte(c.input_arg("age")),
     cast=list
 ).gen_converter()
 # equivalent to
 def converter182_386(data_, *, age):
     return [i182_509 for i182_509 in data_ if (i182_509["age"] >= age)]


.. _ref_pipes:

Pipes
=====

.. automethod:: convtools.base.BaseConversion.pipe
   :noindex:

It's easier to read the code written with pipes in contrast to heavily
nested approach:

.. code-block:: python

 (
     c.item("data", "users")
     .call_method("values")
 ).pipe(
     c.generator_comp({"id": c.item("id"), "name": c.item("name")})
 ).pipe(
     c.list_comp(
         c.inline_expr(
             "{ModelCls}(updated=updated, **data)",
         ).pass_args(
             ModelCls=c.input_arg("model_cls"),
             updated=c.input_arg("updated"),
             data=c.this(),
         )
     )
 ).gen_converter()

 # generates:

 def converter239_386(data_, *, model_cls, updated):
     pipe239_654 = data_["data"]["users"].values()
     pipe239_355 = (
         {"id": i231_917["id"], "name": i231_917["name"]}
         for i231_917 in pipe239_654
     )
     return [
         (model_cls(updated=updated, **data)) for i239_648 in pipe239_355
     ]


Also it's often useful to pass the result to a callable (*of course you can 
wrap any callable to make it a conversion and pass any parameters in any way
you wish*), but there is some syntactic sugar:

.. code-block:: python

   c.item("date_str").pipe(datetime.strptime, "%Y-%m-%d")

   # results in:

   def converter397_422(data_):
       return vstrptime396_498(data_["date_str"], "%Y-%m-%d")
   # so in cases where you pipe to python callable, 
   # the input will be passed as the first param and other params onward



.. _ref_c_conditions:

Conditions
==========

 * **IF expressions**

     .. autoattribute:: convtools.conversion.if_

     .. autoclass:: convtools.base.If()
        :noindex:

     .. automethod:: convtools.base.If.__init__

 * **AND/OR expressions**
     .. autoattribute:: convtools.conversion.and_
     .. autoattribute:: convtools.conversion.or_

     .. autoclass:: convtools.base.And()
        :noindex:

     .. autoclass:: convtools.base.Or()
        :noindex:

     .. automethod:: convtools.base.And.__init__
     .. automethod:: convtools.base.Or.__init__


Examples:

.. code-block:: python

   f = c.list_comp(
       c.if_(
           c.this().is_(None),
           -1,
           c.this()
       )
   ).gen_converter(debug=True)

   f([0, 1, None, 2]) == [0, 1, -1, 2]

   # generates:

   def converter66_417(data_):
       return [(-1 if (i66_248 is None) else i66_248) for i66_248 in data_]


**Imagine we pass something more complex than just a simple value:**

.. code-block:: python

   f = c.list_comp(
       c.call_func(
           lambda x: x, c.this()
       ).pipe(
           c.if_(if_true=c.this() + 2)
       )
   ).gen_converter(debug=True)

   f([1, 0, 2]) == [3, 0, 4]

   # generates:
   # as you can see, it uses a function to cache the input data
   def converter75_417(data_):
       return [
           (
               (vvalue_cache76_824() + 2)
               if vvalue_cache76_824(vlambda69_109(i75_248))
               else vvalue_cache76_824()
           )
           for i75_248 in data_
       ]

It works as follows: if it finds any function calls, index/attribute lookups,
it just caches the input, because the IF cannot be sure whether it's cheap or
applicable to run the input code twice.



.. _ref_c_aggregations:

c.group_by, c.aggregate & c.reduce
==================================


.. _ref_c_group_by:

c.group_by
__________

.. autoattribute:: convtools.conversion.group_by

  .. autoclass:: convtools.aggregations.GroupBy()
     :noindex:

  .. automethod:: convtools.aggregations.GroupBy.__init__
     :noindex:

  .. automethod:: convtools.aggregations.GroupBy.aggregate
     :noindex:

  .. automethod:: convtools.aggregations.GroupBy.filter
     :noindex:

  .. automethod:: convtools.aggregations.GroupBy.sort
     :noindex:


.. _ref_c_aggregate:

c.aggregate
___________

.. autoattribute:: convtools.conversion.aggregate

   .. autofunction:: convtools.aggregations.Aggregate


.. _ref_c_reduce:

c.reduce
________

.. autoattribute:: convtools.conversion.reduce

  .. autoclass:: convtools.aggregations.Reduce()
     :noindex:

  .. automethod:: convtools.aggregations.Reduce.__init__
     :noindex:

  .. automethod:: convtools.aggregations.Reduce.filter
     :noindex:

Examples:

.. code-block:: python

   converter = c.group_by(c.item("category")).aggregate({
       "category": c.item("category").call_method("upper"),
       "earnings": c.reduce(
           c.ReduceFuncs.Sum,
           c.item("earnings").as_type(Decimal),
       ),
       "best_day": c.reduce(
           c.ReduceFuncs.MaxRow,
           c.item("earnings").as_type(float),
       ).item("date").pipe(datetime.strptime, "%Y-%m-%d"),
   }).gen_converter(debug=True)
   # list of dicts
   converter(data)

   converter = c.aggregate({
       "category": c.reduce(
           c.ReduceFuncs.ArrayDistinct,
           c.item("category").call_method("upper"),
       ),
       "earnings": c.reduce(
           c.ReduceFuncs.Sum,
           c.item("earnings").as_type(Decimal),
       ),
       "best_day": c.reduce(
           c.ReduceFuncs.MaxRow,
           c.item("earnings").as_type(float),
       ).item("date").pipe(datetime.strptime, "%Y-%m-%d"),
   }).gen_converter(debug=True)
   # a single dict
   converter(data)


.. _ref_c_reduce_funcs:

c.ReduceFuncs
_____________

.. autoclass:: convtools.aggregations.ReduceFuncs
   :noindex:

   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Sum
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.SumOrNone
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Max
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.MaxRow
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Min
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.MinRow
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Count
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.CountDistinct
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.First
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Last
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Array
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.ArrayDistinct
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.Dict
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictArray
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictArrayDistinct
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictSum
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictSumOrNone
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictMax
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictMin
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictCount
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictCountDistinct
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictFirst
        :noindex:
   * .. autoattribute:: convtools.aggregations.ReduceFuncs.DictLast
        :noindex:


Let's get to an almost self-describing example:

.. code-block:: python

 c.group_by(
     # any number of keys to group by
     c.item("company"), c.item("department"),
 ).aggregate({
     # a list, a tuple or a set would work as well; here we'll get the
     # list of dicts
     "company": c.item("company"),
     "department": c.item("department"),

     # just to demonstrate that even dict keys and groupby keys are dynamic
     c.call_func(
         lambda c: c.upper(),
         c.item("company")
     ): c.item("department").call_method("replace", "PREFIX", ""),


     # some normal SQL-like functionality
     "sales_total": c.reduce(
         c.ReduceFuncs.Sum,
         c.item("sales"),
     ),
     "sales_2019": c.reduce(
         c.ReduceFuncs.Sum,
         c.item("sales"),
     ).filter(
         c.item("date").gte("2019-01-01")
     ),
     "sales_total_too": c.reduce(
         lambda a, b: a + b,
         c.item("sales"),
         initial=0, # initial value for reducer
         default=None, # if not a single value is met (e.g. because of filter)
     ),

     # some additional functionality
     "company_address": c.reduce(
         c.ReduceFuncs.First,
         c.item("company_address"),
     ),

     "oldest_employee": c.reduce(
         c.ReduceFuncs.MaxRow,
         c.item("age"),
     ).item("employee"), # looking for a row with maximum employee age and then taking "employee" field (actually we could take the row untouched)

     # here we perform another aggregation, inside grouping by company & department
     # the output of this reduce is a dict: from employee to sum of sales (within the group)
     "employee_to_sales": c.reduce(
         c.ReduceFuncs.DictSum,
         (c.item("employee"), c.item("sales"))
     ),

     # piping works here too
     "postprocessed_dict_reduce": c.reduce(
         c.ReduceFuncs.DictSum,
         (c.item("currency"), c.item("sales")),
         default=dict,
     ).call_method("items").pipe(
         c.generator_comp(
             c.inline_expr(
                 "{convert_currency}({currency}, {convert_to_currency}, {sales})"
             ).pass_args(
                 convert_currency=convert_currency_func,
                 currency=c.item(0),
                 convert_to_currency=c.input_arg("convert_to_currency"),
                 sales=c.item(1),
             )
         )
     ),
 }).gen_converter()


Adding ``debug=True`` to see the compiled code:
_______________________________________________

.. code-block:: python

     def vgroup_data251_8(data):
         _none = v353_639
         try:
             signature_to_agg_data = defaultdict(AggData)
             for row in data:
                 agg_data = signature_to_agg_data[
                     (row["company"], row["department"],)
                 ]

                 if agg_data.v0 is _none:
                     agg_data.v0 = row["sales"] or 0
                 else:
                     agg_data.v0 += row["sales"] or 0

                 if row["date"] >= "2019-01-01":
                     if agg_data.v1 is _none:
                         agg_data.v1 = row["sales"] or 0
                     else:
                         agg_data.v1 += row["sales"] or 0

                 if agg_data.v2 is _none:
                     agg_data.v2 = vlambda286_765(0, row["sales"])
                 else:
                     agg_data.v2 = vlambda286_765(agg_data.v2, row["sales"])

                 if agg_data.v3 is _none:
                     agg_data.v3 = row["company_address"]
                 else:
                     pass

                 if agg_data.v4 is _none:
                     if row["age"] is not None:
                         agg_data.v4 = (row["age"], row)
                 else:
                     if row["age"] is not None and agg_data.v4[0] < row["age"]:
                         agg_data.v4 = (row["age"], row)

                 if agg_data.v5 is _none:
                     agg_data.v5 = _d = defaultdict(int)
                     _d[row["employee"]] += row["sales"] or 0
                 else:
                     agg_data.v5[row["employee"]] += row["sales"] or 0

             result = [
                 {
                     "company": signature[0],
                     "department": signature[1],
                     vlambda217_607(signature[0]): signature[1].replace(
                         "PREFIX", ""
                     ),
                     "sales_total": (0 if agg_data.v0 is _none else agg_data.v0),
                     "sales_2019": (0 if agg_data.v1 is _none else agg_data.v1),
                     "sales_total_too": (
                         None if agg_data.v2 is _none else agg_data.v2
                     ),
                     "company_address": (
                         None if agg_data.v3 is _none else agg_data.v3
                     ),
                     "oldest_employee": (
                         None if agg_data.v4 is _none else agg_data.v4[1]
                     )["employee"],
                     "employee_to_sales": (
                         None if agg_data.v5 is _none else vdict19_752(agg_data.v5)
                     ),
                     "postprocessed_dict_reduce": (
                         (
                             vlambda276_128(
                                 i277_787[0], convert_to_currency, i277_787[1]
                             )
                         )
                         for i277_787 in (
                             vdict19_548()
                             if agg_data.v6 is _none
                             else vdict19_39(agg_data.v6)
                         ).items()
                     ),
                 }
                 for signature, agg_data in signature_to_agg_data.items()
             ]

             return result

 def converter210_631(data):
     return vgroup_data251_8(data)

Fortunately this code is auto-generated (there's no joy in writing this).
