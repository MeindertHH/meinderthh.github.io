---
layout: post
title: Subclassing pandas data structures
date: 2021-05-15
categories: python
---

# Subclassing pandas data structures

## The problem

Say you want to extend pandas data structures with custom attributes and methods. The classes you define are subclasses of DataFrame and Series. That means that instances of your class are instances of DataFrame or Series as well. You won't loose any functionality of the original pandas data structures. Methods that accept pandas data structures, such as matplotlib, will treat your objects in the same manner.

{% highlight python %}
>>> x = MySeries()
>>> isinstance(x, pd.Series)
True
>>> y = MyDataFrame()
>>> isinstance(y, pd.DataFrame)
True
{% endhighlight %}


## A note of caution

Subclassing data structures is only recommended for advanced pandas users who aren't scared of bug fixing. There is not an abundance of [documentation](https://pandas.pydata.org/pandas-docs/stable/development/extending.html#subclassing-pandas-data-structures) and you might bump into issues for which stackoverflow does not have a ready made sollution.

Also, there are alternatives to subclassing. If you want to extend pandas data structures but you don't need your objects to be instances of a Series/DataFrame, then consider composition, e.g.

{% highlight python %}
class MyClass:

  def __init__(self):
    self.data = pd.DataFrame(...)

  def my_method(self):
    # do something with self.data
    return ...
{% endhighlight %}

Usage:

{% highlight python %}
>>> x = MyClass()
>>> x.my_method()
...
{% endhighlight %}

Are you sure you need to subclass? Then please continue reading. The guide below is based on this [documentation](https://pandas.pydata.org/pandas-docs/stable/development/extending.html#subclassing-pandas-data-structures) and other sources addressing specific issues (linked below).

## How to

1. Define the subclasses
2. Override constructor properties
3. Define original properties

### 1. Define the subclasses

In this guide we will assume that you want to create two subclasses, a child for Series and a child for DataFrame. We create the child classes as usual, simply by sending the parent classes as a parameter.

{% highlight python %}
import pandas as pd


class SubclassedSeries(pd.Series):
    pass


class SubclassedDataFrame(pd.DataFrame):
    pass
{% endhighlight %}

When creating objects from these classes, the parameters get send to the parent class (`__init__` is inherited from the parent):

{% highlight python %}
>>> s = SubclassedSeries(data=['a', 'b', 'c'])
>>> s
0    a
1    b
2    c
dtype: object
>>> type(s)
<class "__main__.SubclassedSeries">
>>> df = SubclassedDataFrame(data=['a', 'b', 'c'])
>>> df
   0
0  a
1  b
2  c
>>> type(df)
<class "__main__.SubclassedDataFrame">
{% endhighlight %}

### 2. Override constructor properties

If we manipulate these structures, then the child class might be lost. For example:

{% highlight python %}
>>> s2 = s + s
>>> s2
0    aa
1    bb
2    cc
dtype: object
>>> type(s2)
<class "pandas.core.series.Series">
{% endhighlight %}

When manipulating, you want your SubclassedSeries to construct and return a SubclassedSeries and SubclassedDataFrame to construct and return a SubclassedDataFrame. For that one needs to overwrite the `_constructor` property:

{% highlight python %}
class SubclassedSeries(pd.Series):
    @property
    def _constructor(self):
        return SubclassedSeries

class SubclassedDataFrame(pd.DataFrame):
    @property
    def _constructor(self):
        return SubclassedDataFrame
{% endhighlight %}

Now:

{% highlight python %}
>>> s2 = s + s
>>> s2
0    aa
1    bb
2    cc
dtype: object
>>> type(s2)
<class "__main__.SubclassedSeries">
{% endhighlight %}

### 3. Define original properties

## Potential issues

I ran across the following issues:
1. `AttributeError: 'function' object has no attribute '_get_axis_number'`
2. Transferring metadata from subclassed DataFrame to subclassed Series and vice versa.

###  '_get_axis_number'

### Transferring metadata
