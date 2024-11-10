---
title: "Django Deletion Dragons"
date: 2019-07-28T00:00:00+00:00
draft: false
---

# Django Deletion Dragons

Django models offers an ORM API that abstracts the database layer.  There are many options provided to take advantage of common database features.  But certain combinations of these options may catch you by surprise; going so far as unintentional data deletion.

As an example, consider the following models:
![Order Schema](/mySiteStatic/images/DjangoDeletionOrderExample.png)

Presume the developers that created your database schema did not have lunch often with the developers who created the software using the database. The software was written to create the instances from the bottom up. First `OrderLine`s, then `Order`s from `OrderLine`s, then `OrderReport`s from `Order`s.  This has a robustness advantage; as soon as an entity is defined it is saved to the database. The software may do something like this:

```python
# Somebody bought some things from us! Hooray!
lines = [OrderLine.objects.create()] * 2

# We package the lines up into an order
order = Order.objects.create()
for line in lines:
    line.order = order
    line.save()

# It's the end of the month, we create a report to generate a document
report = OrderReport.objects.create()
report.orders.add(order)
report.save()
```

But since the `OrderReport` instance was only used to generate a report it is no longer needed (at least for the MVP version you pitched to the VCs).  Your savvy software developers like things tidy so they delete it.

```python
report.delete()
```

A few weeks later you go to access the `OrderLine`s of the `Order` again, but find they no longer exist. 

Oh snap! What happened?

What happened is the database developers decided to be nice and provide convenience features.  They made it so you could create an `OrderLine` first, then create an `Order` from `OrderLine`s, as in the above code. To have this behavior, they told Django in the `ForeignKey` declaration it was OK for a child (`OrderLine`) to be created without a parent (`Order`) with the argument

```python    
null=True
```

The database developers were also kind enough to provide an auto-delete feature such that if you delete some entity, then all of it's children would be deleted. This is done with the `ForeignKey` argument

```python
on_delete=CASCADE
```

This is why when the software deleted the `OrderReport`, all the `Order`s and `OrderLine`s in the report were also deleted.

This may seem like a contrived example, but happens often in real life, especially before Django 2.0 came out. As in the [release notes](https://docs.djangoproject.com/en/2.0/releases/2.0/)

> The on_delete argument for ForeignKey and OneToOneField is now required in models and migrations.

Before version 2.0, the `on_delete` argument [defaulted to CASCADE](https://docs.djangoproject.com/en/1.11/ref/models/fields/#django.db.models.ForeignKey). So if your dev team created models, but forgot to include the `on_delete` parameter, and did not test properly then data loss could easily happen.

If you would like to play around with this example, and also the combinations of \[One, Many\] X \[CASCADE, SET_NULL, PROTECT\], see [this sandbox project which provides a Dockerized Jupyter environment on a Django project](https://github.com/RandyMoore/DjangoDeletion).  The models have the ability to report what children they have, and all models report when they are being deleted.

[Model definitions (including those used by the example code)](https://github.com/RandyMoore/DjangoDeletion/blob/master/delete_example/models.py)

Some sample pre-run notebooks are provided:

* [This Example](https://github.com/RandyMoore/DjangoDeletion/blob/master/Motivating%20Example.ipynb)
* [Many to Many](https://github.com/RandyMoore/DjangoDeletion/blob/master/ManyToMany.ipynb)
* [Many to One](https://github.com/RandyMoore/DjangoDeletion/blob/master/ManyToOne.ipynb)
* [One to One](https://github.com/RandyMoore/DjangoDeletion/blob/master/OneToOne.ipynb)



