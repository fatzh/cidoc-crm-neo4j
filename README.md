# cidoc-crm-neo4j
A meta-implementation of the [CIDOC Conceptual Reference Model](http://cidoc-crm.org/) in [Neo4j](https://neo4j.com), using [neomodel](http://neomodel.readthedocs.io).

This package implements the CRM by dynamically creating model classes from the CRM RDF specification. This limits code maintenance, and makes it easy to switch among versions of the model (although migration is not yet addressed). There's actually no obvious reason why this couldn't be used with any semantic model expressed in RDF/XML, I just haven't tried it yet. Perhaps you will?

I included a subclass of [neomodel.core.StructuredNode](http://neomodel.readthedocs.io/en/latest/module_documentation.html#neomodel.core.StructuredNode) that implements a method called ``downcast``, which re-instantiates a node instances using the most derivative class, or a target class that is a child of the node's current class. Combined with the fact that neomodel applies all tags in a class hierarchy to node instances and uses those labels to enforce relationship definitions, the result is a full implementation of heritable relationships (CRM properties).

## Loading the models

If you need to specify database configuration, you should do that **before** importing from this package.

```
>>> from neomodel import config
>>> config.DATABASE_URL = 'bolt://neo4j:neo4j@localhost:7687'
>>> schema_location = 'http://..../path/to/schema.rdfs.xml'
>>> from crm import models    # The correct way!
>>> models.import_schema(schema_location)    # Download and implement.
```

This might take a few moments, since the package will download and parse the RDF specification for CRM at this point. This is fine for the intended use-cases of this package, e.g. loading models once at application start. You can reduce the overhead of this procedure by storing a copy of the RDF in a more proximate location.

Note that ``schema_location`` can be anything that is valid for the ``source``
parameter in [``rdflib.Graph.parse``](https://rdflib.readthedocs.io/en/stable/apidocs/rdflib.html#rdflib.graph.Graph.parse) (e.g. a local document).

The ``models`` namespace should now be populated with CRM class and property instances.

```
>>> joe = model.E21Person(value='Joe Bloggs')
>>> joe.save()
<E21Person: {'id': 33, 'value': 'Joe Bloggs'}>

>>> tempe = model.E53Place(value='Tempe, Arizona')
>>> tempe.save()
<E53Place: {'id': 34, 'value': 'Tempe, Arizona'}>

>>> joe.P74_has_current_or_former_residence.connect(tempe)
<neomodel.relationship.P74HasCurrentOrFormerResidence at 0x109800250>
```

## Model consistency is enforced, with mindfulness of inheritance!

neomodel already allows sub-classes to inherit the powers of their superiors, so for example:

```
>>> class SomeModel(StructuredNode):
...     likes = RelationshipTo('AnotherModel')
...
>>> class SomeSubModel(SomeModel):
...     pass
...
>>> class AnotherModel(StructuredNode):
...     pass
...
>>> class ChildModel(AnotherModel):
...     pass
...
>>> child = ChildModel()
>>> child.save()
>>> something = SomeSubModel()
>>> something.save()
>>> something.likes.connect(child)
```

So we get property-inheritance in the CRM for free. For example, even though ``P2 has type`` is a property of ``E1 CRM Entity``, we can use it on a ``E21 Person`` (who inherits from ``E1``):

```
>>> joe = models.E21Person(value='Joe Bloggs')
>>> joe.save()
>>> joe.P2_has_type.connect(a_type_object)
```


We are also prevented from doing things that are prevented by the CRM:

```
>>> jane = models.E21Person(value='Jane Doe')
>>> jane.save()
>>> joe.P74_has_current_or_former_residence.connect(jane)
ValueError: Expected node of class E53Place
```

## Retrieving node data as instances of derivative classes

The one problem that neomodel doesn't solve for us is how we get those child classes back out again when we're using properties (relations) that belong to higher-level classes. For example, suppose I'm modeling participants in an event of some kind:

```
>>> event = models.E5Event(value='Some cool happening')
>>> event.save()
>>> event.P11_had_participant.connect(joe)
<neomodel.relationship.P11HadParticipant at 0x105c82fd0>
```

So great, no complaints. But [``P11 had participant``](http://cidoc-crm.org/Property/P11-had-participant/Version-6.2) targets [``E39 Actor``](http://cidoc-crm.org/Property/P11-had-participant/Version-6.2); since our Joe an [``E21 Person``](http://cidoc-crm.org/Entity/E21-Person/Version-6.2) is also an ``E39 Actor`` our relation is valid, but when we go to retrieve targets of that relation we won't know that Joe was originally instantiated as ``E21 Person`` (the more derivative class).

```
>>> for target in event.P11_had_participant.all():
...     print target, type(target)
{'id': 39, 'value': u'Joe Bloggs'} <class 'neomodel.core.E39Actor'>
```

Since all models in this package subclass ``HeritableStructuredNode``, they have a method called ``downcast()`` that solves this problem. For example:

```
>>> for target in event.P11_had_participant.all():
...     original_target = target.downcast()
...     print original_target, type(original_target)
{'id': 39, 'value': u'Joe Bloggs'} <class 'neomodel.core.E21Person'>
```

We can also ``downcast()`` to a specific derivative class, and/or ``upcast()`` to a specific super-class:

```
>>> actor = event.P11_had_participant.single()
>>> actor
<E39Actor: {'id': 54, 'value': u'Joe Bloggs'}>
>>> actor.downcast('E21Person')
<E21Person: {'id': 54, 'value': u'Joe Bloggs'}>
>>> actor.downcast('E21Person').upcast(models.E18PhysicalThing)
<E18PhysicalThing: {'id': 54, 'value': u'Joe Bloggs'}>
```

But we run into trouble (as we should) if we try to do things that violate the CRM:

```
>>> actor.downcast('E21Person').upcast('E18PhysicalThing').downcast('E28ConceptualObject')
ValueError: E28ConceptualObject is not a sub-class of E18PhysicalThing

>>> actor.upcast('E7Activity')
ValueError: E7Activity is not a super-class of E39Actor
```
