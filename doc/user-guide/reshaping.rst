.. _reshape:

###############################
Reshaping and reorganizing data
###############################

Reshaping and reorganizing data refers to the process of changing the structure or organization of data by modifying dimensions, array shapes, order of values, or indexes. Xarray provides several methods to accomplish these tasks.

These methods are particularly useful for reshaping xarray objects for use in machine learning packages, such as scikit-learn, that usually require two-dimensional numpy arrays as inputs. Reshaping can also be required before passing data to external visualization tools, for example geospatial data might expect input organized into a particular format corresponding to stacks of satellite images.

Importing the library
---------------------

.. jupyter-execute::
    :hide-code:

    import numpy as np
    import pandas as pd
    import xarray as xr

    np.random.seed(123456)

    # Use defaults so we don't get gridlines in generated docs
    import matplotlib as mpl

    mpl.rcdefaults()

Reordering dimensions
---------------------

To reorder dimensions on a :py:class:`~xarray.DataArray` or across all variables
on a :py:class:`~xarray.Dataset`, use :py:meth:`~xarray.DataArray.transpose`. An
ellipsis (`...`) can be used to represent all other dimensions:

.. jupyter-execute::

    ds = xr.Dataset({"foo": (("x", "y", "z"), [[[42]]]), "bar": (("y", "z"), [[24]])})
    ds.transpose("y", "z", "x") # equivalent to ds.transpose(..., "x")

.. jupyter-execute::

    ds.transpose()  # reverses all dimensions

Expand and squeeze dimensions
-----------------------------

To expand a :py:class:`~xarray.DataArray` or all
variables on a :py:class:`~xarray.Dataset` along a new dimension,
use :py:meth:`~xarray.DataArray.expand_dims`

.. jupyter-execute::

    expanded = ds.expand_dims("w")
    expanded

This method attaches a new dimension with size 1 to all data variables.

To remove such a size-1 dimension from the :py:class:`~xarray.DataArray`
or :py:class:`~xarray.Dataset`,
use :py:meth:`~xarray.DataArray.squeeze`

.. jupyter-execute::

    expanded.squeeze("w")

Converting between datasets and arrays
--------------------------------------

To convert from a Dataset to a DataArray, use :py:meth:`~xarray.Dataset.to_dataarray`:

.. jupyter-execute::

    arr = ds.to_dataarray()
    arr

This method broadcasts all data variables in the dataset against each other,
then concatenates them along a new dimension into a new array while preserving
coordinates.

To convert back from a DataArray to a Dataset, use
:py:meth:`~xarray.DataArray.to_dataset`:

.. jupyter-execute::

    arr.to_dataset(dim="variable")

The broadcasting behavior of ``to_dataarray`` means that the resulting array
includes the union of data variable dimensions:

.. jupyter-execute::

    ds2 = xr.Dataset({"a": 0, "b": ("x", [3, 4, 5])})

    # the input dataset has 4 elements
    ds2

.. jupyter-execute::

    # the resulting array has 6 elements
    ds2.to_dataarray()

Otherwise, the result could not be represented as an orthogonal array.

If you use ``to_dataset`` without supplying the ``dim`` argument, the DataArray will be converted into a Dataset of one variable:

.. jupyter-execute::

    arr.to_dataset(name="combined")

.. _reshape.stack:

Stack and unstack
-----------------

As part of xarray's nascent support for :py:class:`pandas.MultiIndex`, we have
implemented :py:meth:`~xarray.DataArray.stack` and
:py:meth:`~xarray.DataArray.unstack` method, for combining or splitting dimensions:

.. jupyter-execute::

    array = xr.DataArray(
        np.random.randn(2, 3), coords=[("x", ["a", "b"]), ("y", [0, 1, 2])]
    )
    stacked = array.stack(z=("x", "y"))
    stacked

.. jupyter-execute::

    stacked.unstack("z")

As elsewhere in xarray, an ellipsis (`...`) can be used to represent all unlisted dimensions:

.. jupyter-execute::

    stacked = array.stack(z=[..., "x"])
    stacked

These methods are modeled on the :py:class:`pandas.DataFrame` methods of the
same name, although in xarray they always create new dimensions rather than
adding to the existing index or columns.

Like :py:meth:`DataFrame.unstack<pandas.DataFrame.unstack>`, xarray's ``unstack``
always succeeds, even if the multi-index being unstacked does not contain all
possible levels. Missing levels are filled in with ``NaN`` in the resulting object:

.. jupyter-execute::

    stacked2 = stacked[::2]
    stacked2

.. jupyter-execute::

    stacked2.unstack("z")

However, xarray's ``stack`` has an important difference from pandas: unlike
pandas, it does not automatically drop missing values. Compare:

.. jupyter-execute::

    array = xr.DataArray([[np.nan, 1], [2, 3]], dims=["x", "y"])
    array.stack(z=("x", "y"))

.. jupyter-execute::

    array.to_pandas().stack()

We departed from pandas's behavior here because predictable shapes for new
array dimensions is necessary for :ref:`dask`.

.. _reshape.stacking_different:

Stacking different variables together
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

These stacking and unstacking operations are particularly useful for reshaping
xarray objects for use in machine learning packages, such as `scikit-learn
<https://scikit-learn.org>`_, that usually require two-dimensional numpy
arrays as inputs. For datasets with only one variable, we only need ``stack``
and ``unstack``, but combining multiple variables in a
:py:class:`xarray.Dataset` is more complicated. If the variables in the dataset
have matching numbers of dimensions, we can call
:py:meth:`~xarray.Dataset.to_dataarray` and then stack along the the new coordinate.
But :py:meth:`~xarray.Dataset.to_dataarray` will broadcast the dataarrays together,
which will effectively tile the lower dimensional variable along the missing
dimensions. The method :py:meth:`xarray.Dataset.to_stacked_array` allows
combining variables of differing dimensions without this wasteful copying while
:py:meth:`xarray.DataArray.to_unstacked_dataset` reverses this operation.
Just as with :py:meth:`xarray.Dataset.stack` the stacked coordinate is
represented by a :py:class:`pandas.MultiIndex` object. These methods are used
like this:

.. jupyter-execute::

    data = xr.Dataset(
        data_vars={"a": (("x", "y"), [[0, 1, 2], [3, 4, 5]]), "b": ("x", [6, 7])},
        coords={"y": ["u", "v", "w"]},
    )
    data

.. jupyter-execute::

    stacked = data.to_stacked_array("z", sample_dims=["x"])
    stacked

.. jupyter-execute::

    unstacked = stacked.to_unstacked_dataset("z")
    unstacked

In this example, ``stacked`` is a two dimensional array that we can easily pass to a scikit-learn or another generic
numerical method.

.. note::

    Unlike with ``stack``,  in ``to_stacked_array``, the user specifies the dimensions they **do not** want stacked.
    For a machine learning task, these unstacked dimensions can be interpreted as the dimensions over which samples are
    drawn, whereas the stacked coordinates are the features. Naturally, all variables should possess these sampling
    dimensions.


.. _reshape.set_index:

Set and reset index
-------------------

Complementary to stack / unstack, xarray's ``.set_index``, ``.reset_index`` and
``.reorder_levels`` allow easy manipulation of ``DataArray`` or ``Dataset``
multi-indexes without modifying the data and its dimensions.

You can create a multi-index from several 1-dimensional variables and/or
coordinates using :py:meth:`~xarray.DataArray.set_index`:

.. jupyter-execute::

    da = xr.DataArray(
        np.random.rand(4),
        coords={
            "band": ("x", ["a", "a", "b", "b"]),
            "wavenumber": ("x", np.linspace(200, 400, 4)),
        },
        dims="x",
    )
    da

.. jupyter-execute::

    mda = da.set_index(x=["band", "wavenumber"])
    mda

These coordinates can now be used for indexing, e.g.,

.. jupyter-execute::

    mda.sel(band="a")

Conversely, you can use :py:meth:`~xarray.DataArray.reset_index`
to extract multi-index levels as coordinates (this is mainly useful
for serialization):

.. jupyter-execute::

    mda.reset_index("x")

:py:meth:`~xarray.DataArray.reorder_levels` allows changing the order
of multi-index levels:

.. jupyter-execute::

    mda.reorder_levels(x=["wavenumber", "band"])

As of xarray v0.9 coordinate labels for each dimension are optional.
You can also use ``.set_index`` / ``.reset_index`` to add / remove
labels for one or several dimensions:

.. jupyter-execute::

    array = xr.DataArray([1, 2, 3], dims="x")
    array

.. jupyter-execute::

    array["c"] = ("x", ["a", "b", "c"])
    array.set_index(x="c")

.. jupyter-execute::

    array = array.set_index(x="c")
    array = array.reset_index("x", drop=True)

.. _reshape.shift_and_roll:

Shift and roll
--------------

To adjust coordinate labels, you can use the :py:meth:`~xarray.Dataset.shift` and
:py:meth:`~xarray.Dataset.roll` methods:

.. jupyter-execute::

    array = xr.DataArray([1, 2, 3, 4], dims="x")
    array.shift(x=2)

.. jupyter-execute::

    array.roll(x=2, roll_coords=True)

.. _reshape.sort:

Sort
----

One may sort a DataArray/Dataset via :py:meth:`~xarray.DataArray.sortby` and
:py:meth:`~xarray.Dataset.sortby`. The input can be an individual or list of
1D ``DataArray`` objects:

.. jupyter-execute::

    ds = xr.Dataset(
        {
            "A": (("x", "y"), [[1, 2], [3, 4]]),
            "B": (("x", "y"), [[5, 6], [7, 8]]),
        },
        coords={"x": ["b", "a"], "y": [1, 0]},
    )
    dax = xr.DataArray([100, 99], [("x", [0, 1])])
    day = xr.DataArray([90, 80], [("y", [0, 1])])
    ds.sortby([day, dax])

As a shortcut, you can refer to existing coordinates by name:

.. jupyter-execute::

    ds.sortby("x")

.. jupyter-execute::

    ds.sortby(["y", "x"])

.. jupyter-execute::

    ds.sortby(["y", "x"], ascending=False)

.. _reshape.coarsen:

Reshaping via coarsen
---------------------

Whilst :py:class:`~xarray.DataArray.coarsen` is normally used for reducing your data's resolution by applying a reduction function
(see the :ref:`page on computation<compute.coarsen>`),
it can also be used to reorganise your data without applying a computation via :py:meth:`~xarray.computation.rolling.DataArrayCoarsen.construct`.

Taking our example tutorial air temperature dataset over the Northern US

.. jupyter-execute::

    air = xr.tutorial.open_dataset("air_temperature")["air"]

    air.isel(time=0).plot(x="lon", y="lat");

we can split this up into sub-regions of size ``(9, 18)`` points using :py:meth:`~xarray.computation.rolling.DataArrayCoarsen.construct`:

.. jupyter-execute::

    regions = air.coarsen(lat=9, lon=18, boundary="pad").construct(
        lon=("x_coarse", "x_fine"), lat=("y_coarse", "y_fine")
    )
    with xr.set_options(display_expand_data=False):
        regions

9 new regions have been created, each of size 9 by 18 points.
The ``boundary="pad"`` kwarg ensured that all regions are the same size even though the data does not evenly divide into these sizes.

By plotting these 9 regions together via :ref:`faceting<plotting.faceting>` we can see how they relate to the original data.

.. jupyter-execute::

    regions.isel(time=0).plot(
        x="x_fine", y="y_fine", col="x_coarse", row="y_coarse", yincrease=False
    );

We are now free to easily apply any custom computation to each coarsened region of our new dataarray.
This would involve specifying that applied functions should act over the ``"x_fine"`` and ``"y_fine"`` dimensions,
but broadcast over the ``"x_coarse"`` and ``"y_coarse"`` dimensions.
