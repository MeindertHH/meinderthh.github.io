---
layout: post
title: Subclassing pandas data structures
date: 2022-12-09
categories: python
---

# Subclassing pandas data structures

## The problem

You want to extend pandas data structures with custom attributes and methods. The classes you define are subclasses of DataFrame and Series. That means that instances of your class are instances of DataFrame or Series as well. You won't loose functionality of the original pandas data structures. Methods that accept pandas data structures, such as matplotlib, will treat your objects in the same manner.

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

For this post I have used Python 3.10 and pandas 1.5.1.

## Steps

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
<class __main__.SubclassedSeries>
>>> df = SubclassedDataFrame(data=['a', 'b', 'c'])
>>> df
   0
0  a
1  b
2  c
>>> type(df)
<class __main__.SubclassedDataFrame>
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
<class pandas.core.series.Series>
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
<class __main__.SubclassedSeries>
{% endhighlight %}

Likewise, we want the SubclassedSeries to construct a SubclassedDataFrame when going from 1D to 2D and vice versa.

Right now, a SubclassedSeries constructs a pandas DataFrame:

{% highlight python %}
>>> s_to_df = s.to_frame()
>>> type(s_to_df)
<class 'pandas.core.frame.DataFrame'>
{% endhighlight %}

Vice versa, slicing a SubclassedDataFrame returns a pandas.Series instead of a SubclassedSeries.

To fix this, we need to override the `_constructor_expanddim`  and `_constructor_sliced` properties.

{% highlight python %}
class SubclassedSeries(pd.Series):
    @property
    def _constructor(self):
    return SubclassedSeries

    @property
    def _constructor_expanddim(self):
        return SubclassedDataFrame

class SubclassedDataFrame(pd.DataFrame):
    @property
    def _constructor(self):
    return SubclassedDataFrame

    @property
    def _constructor_sliced(self):
        return SubclassedSeries

{% endhighlight %}

Unfortunately, this will not copy the metadata (e.g. data you have added for original properties, see below). Here is a [workaround](https://github.com/pandas-dev/pandas/issues/13208#issuecomment-326556232) (that may become unnecessary with a future release of pandas):

{% highlight python %}
# Adapted from
# https://github.com/pandas-dev/pandas/issues/13208#issuecomment-326556232

class SubclassedSeries(pd.Series):
    @property
    def _constructor(self):
        def f(*args, **kwargs):
            return SubclassedSeries(*args, **kwargs).__finalize__(self)
        return f

    @property
    def _constructor_expanddim(self):
        def f(*args, **kwargs):
            return SubclassedDataFrame(
                *args, **kwargs).__finalize__(self, method='inherit')
        return f

class SubclassedDataFrame(pd.DataFrame):
    @property
    def _constructor(self):
        def f(*args, **kwargs):
            return SubclassedDataFrame(*args, **kwargs).__finalize__(self)
        return f

    @property
    def _constructor_sliced(self):
        def f(*args, **kwargs):
            return SubclassedSeries(
                *args, **kwargs).__finalize__(self, method='inherit')
        return f

{% endhighlight %}

### 3. Define original properties

You are now ready to define new properties.

{% highlight python %}
class SubclassedSeries(pd.Series):
    ...

    @property
    def my_property(self):
        # do something
        return ...

class SubclassedDataFrame(pd.DataFrame):
    ...

    @property
    def my_property(self):
        # do something
        return ...

{% endhighlight %}

## Transfering metadata

If at any point you need to transfer metadata from one instance of a subclassed data structure to another, just call `__finalize__(self, method='inherit')`, e.g.

{% highlight python %}
>>> another_s.__finalize__(s, method='inherit')
{% endhighlight %}
