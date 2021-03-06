Extending Colander
==================

You can extend Colander by defining a new :term:`type` or by defining
a new :term:`validator`.

.. _defining_a_new_type:

Defining a New Type
-------------------

A type is a class that inherits from ``colander.SchemaType`` and implements
these methods:

- ``serialize``: converts a Python data structure (:term:`appstruct`)
  into a serialization (:term:`cstruct`).
- ``deserialize``: converts a serialized value (:term:`cstruct`) into a
  Python data structure (:term:`appstruct`).
- If it contains child nodes, it must also implement ``cstruct_children``,
  ``flatten``, ``unflatten``, ``set_value`` and ``get_value`` methods. It
  may inherit from ``Mapping``, ``Tuple``, ``Set``, ``List`` or ``Sequence``
  to obtain these methods, but only if the expected behavior is the same.

.. note::

   See also: :class:`colander.interfaces.Type`.

.. note::

   The ``cstruct_children`` method became required in Colander 0.9.9.

An Example
~~~~~~~~~~

Here's a type which implements boolean serialization and deserialization.  It
serializes a boolean to the string ``"true"`` or ``"false"`` or the special
:attr:`colander.null` sentinel; it then deserializes a string (presumably
``"true"`` or ``"false"``, but allows some wiggle room for ``"t"``, ``"on"``,
``"yes"``, ``"y"``, and ``"1"``) to a boolean value.

.. code-block::  python
   :linenos:

   from colander import SchemaType, Invalid, null

   class Boolean(SchemaType):
       def serialize(self, node, appstruct):
           if appstruct is null:
               return null
           if not isinstance(appstruct, bool):
               raise Invalid(node, '%r is not a boolean' % appstruct)
           return appstruct and 'true' or 'false'

       def deserialize(self, node, cstruct):
           if cstruct is null:
               return null
           if not isinstance(cstruct, basestring):
               raise Invalid(node, '%r is not a string' % cstruct)
           value = cstruct.lower()
           if value in ('true', 'yes', 'y', 'on', 't', '1'):
               return True
           return False

Here's how you would use the resulting class as part of a schema:

.. code-block:: python
   :linenos:

   import colander

   class Schema(colander.MappingSchema):
       interested = colander.SchemaNode(Boolean())

The above schema has a member named ``interested`` which will now be
serialized and deserialized as a boolean, according to the logic defined in
the ``Boolean`` type class.

Method Specifications
~~~~~~~~~~~~~~~~~~~~~

``serialize``
^^^^^^^^^^^^^

Arguments:

- ``node``: the ``SchemaNode`` associated with this type
- ``appstruct``: the :term:`appstruct` value that needs to be serialized

If ``appstruct`` is invalid, it should raise :exc:`colander.Invalid`,
passing ``node`` as the first constructor argument.

It must deal specially with the value :attr:`colander.null`.

It must be able to make sense of any value generated by ``deserialize``.

``deserialize``
^^^^^^^^^^^^^^^

Arguments:

- ``node``: the ``SchemaNode`` associated with this type
- ``cstruct``: the :term:`cstruct` value that needs to be deserialized

If ``cstruct`` is invalid, it should raise :exc:`colander.Invalid`,
passing ``node`` as the first constructor argument.

It must deal specially with the value :attr:`colander.null`.

It must be able to make sense of any value generated by ``serialize``.

``cstruct_children``
^^^^^^^^^^^^^^^^^^^^

Arguments:

- ``node``: the ``SchemaNode`` associated with this type
- ``cstruct``: the :term:`cstruct` that the caller wants to obtain child values
  for

You only need to define this method for complex types that have child nodes,
such as mappings and sequences.

``cstruct_children`` should return a value based on ``cstruct`` for
each child node in ``node`` (or an empty list if ``node`` has no children). If
``cstruct`` does not contain a value for a particular child, that child should
be replaced with the ``colander.null`` value in the returned list.

``cstruct_children`` should *never* raise an exception, even if it is passed a
nonsensical ``cstruct`` argument. In that case, it should return a sequence of
as many ``colander.null`` values as there are child nodes.


Constructor (``__init__``)
^^^^^^^^^^^^^^^^^^^^^^^^^^

`SchemaType` does not define a constructor, and user code (not Colander)
instantiates type objects, so custom types may define this method and use it
for their own purposes.


Null Values
~~~~~~~~~~~

Both the ``serialize`` and ``deserialize`` methods must be able to
receive :attr:`colander.null` values and handle them intelligently. This
will happen whenever the data structure being serialized or deserialized
does not provide a value for this node. In many cases, ``serialize`` or
``deserialize`` should just return :attr:`colander.null` when passed
:attr:`colander.null`.

A type might also choose to return :attr:`colander.null` if the value it
receives is *logically* (but not literally) null.  For example,
:class:`colander.String` type converts the empty string to ``colander.null``
within its ``deserialize`` method.

.. code-block:: python
   :linenos:

    def deserialize(self, node, cstruct):
        if not cstruct:
            return null


.. _defining_a_new_validator:

Defining a New Validator
------------------------

A validator is a callable which accepts two positional arguments:
``node`` and ``value``.  It returns ``None`` if the value is valid.
It raises a :class:`colander.Invalid` exception if the value is not
valid.  Here's a validator that checks if the value is a valid credit
card number.

.. code-block:: python
   :linenos:

   def luhnok(node, value):
       """ checks to make sure that the value passes a luhn mod-10 checksum """
       sum = 0
       num_digits = len(value)
       oddeven = num_digits & 1

       for count in range(0, num_digits):
           digit = int(value[count])

           if not (( count & 1 ) ^ oddeven ):
               digit = digit * 2
           if digit > 9:
               digit = digit - 9

           sum = sum + digit

       if not (sum % 10) == 0:
           raise Invalid(node,
                         '%r is not a valid credit card number' % value)

Here's how the resulting ``luhnok`` validator might be used in a
schema:

.. code-block:: python
   :linenos:

   import colander

   class Schema(colander.MappingSchema):
       cc_number = colander.SchemaNode(colander.String(), validator=lunhnok)

Note that the validator doesn't need to check if the ``value`` is a
string: this has already been done as the result of the type of the
``cc_number`` schema node being :class:`colander.String`. Validators
are always passed the *deserialized* value when they are invoked.

The ``node`` value passed to the validator is a schema node object; it
must in turn be passed to the :exc:`colander.Invalid` exception
constructor if one needs to be raised.

For a more formal definition of a the interface of a validator, see
:class:`colander.interfaces.Validator`.
