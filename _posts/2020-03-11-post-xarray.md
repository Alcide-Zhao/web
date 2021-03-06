---
title: 'Xarray notes'
date: 2020-03-11
permalink: /posts/2020/03/post-xarray/
tags:
  - Xarray
  - python
  - programming
---

People are all talking about Xarray, but God knows how hesitating I am to switch from Python 2 to Python 3. Unfortunately, **Xarray was developed with Python 3, so you cannot use it in Python 2.**

## Why Xarray
If you look at their [official web](xarray.pydata.org/en/latest/why-xarray.html), what they are trying to convince you simply is the importance of labelling the axis of N-dimenssional data with more meaningful names (Like pandas.DataFrame, but extending dimenssion (ndim>2)). This makes sense because it is less easier to make mistakes manipulating N-dimenssional data using labels than using axis numbers (axis=0,1,2,...). 

It is also true that, by labelling axis, it is easier to pick up a script after a long time without touching. However, this does not convince me, because it is always better to put comments and nots nwhatever programming languages you use (Jupyter notebook is highly recommended!) 

To learn from scratch, you must first understand these self-defined [terminologies](xarray.pydata.org/en/latest/terminology.html). I know, physicists always find it difficult communicating with artists, and vice versa!

Just a few words on dimenssions and coordinates. This seems complicated, but it is atcually just 'axis' and 'lables along the axis'. Dimenssions are easy to understand, while you can think coordinates as ticklables on axis. Therefore, one axis can have multiple coordinates, but a cordinate can only be assigned uniquelly to one axis. 

Whatsoever, let's start !

## 0. Data structure
**Xarray is just a dic-like N-D generalisation of pandas.series!**

*DataArray*: An N-D generalization of a pandas.Series.

*Dataset*: A dict-like collection of DataArray objects with aligned dimensions (like a necdf file).

### DataArray

The `DataArray` constructor takes:
   - data: a multi-dimensional array of values
   - coords: a list or dictionary of coordinates.
   - dims: a list of dimension names.
   - attrs: a dictionary of attributes to add to the instance
   - name: a string that names the instance
 To dump these fields, one can do `DA.values, DA.coords, DA.dims, DA.atrs, DA.name`
 <pre>
import Xarray as xr
import numpas as np
import pandas as pd

data = np.random.rand(4, 3)
locs = ['IA', 'IL', 'IN']
times = pd.date_range('2000-01-01', periods=4)
foo = xr.DataArray(data, coords=[times, locs], dims=['time', 'space'])
 </pre>

  - To modify data inside DataArray: `foo.values = 1.0 * foo.values`
  - To add attributes: `foo.attrs['units'] = 'meters'`
  - To add/change coordinates: `foo.coords['time']=pd.date_range('2000-01-01', periods=4)`

### Dataset
**Just think `DataSet` as a netcdf file. it contains many variables inside which are like DataArray**
To make a `Dataset` from scratch, supply dictionaries for any variables (data_vars), coordinates (coords) and attributes (attrs). coordinates and variables are all dict-lik in `dataset`, and can be accessed by their keys.

<pre>
temp = 15 + 8 * np.random.randn(2, 2, 3)
precip = 10 * np.random.rand(2, 2, 3)
lon = [[-99.83, -99.32], [-99.79, -99.23]]
lat = [[42.25, 42.21], [42.63, 42.59]]

ds = xr.Dataset({'temperature': (['x', 'y', 'time'],  temp),
                 'precipitation': (['x', 'y', 'time'], precip)},
                 coords={'lon': (['x', 'y'], lon),
                         'lat': (['x', 'y'], lat),
                         'time': pd.date_range('2014-09-06', periods=3),
                         'reference_time': pd.Timestamp('2000-01-01')})

</pre>
  -  To access content in dataset: `ds[temperature]`
  -  To show content: `ds.coords, ds.data_vars, ds.attrs`
  -  To change content: `ds['temperature_double'] = (['x', 'y', 'time'], temp * 2 )`
  -  To remove variables: `ds.drop_vars[temperature]`
  -  To remove dimenssions: `ds.drop_dims('time')`
  -  To rename variables: `ds.rename({'temperature': 'temp', 'precipitation': 'precip'})`


## 1. Indexing and slicing

Dataarray can be indexed as in numpy. However, we want to meke use of the advantage of labelling. That is, label-based indexing, 

Like pandas, label based indexing in xarray is inclusive of both the start and stop bounds.

<pre>
ds['temperature'].loc[-99.79,42.63,'2000-01-01':'2000-01-02']
# index by integer array indices
da.isel(space=0, time=slice(None, 2))

# index by dimension coordinate labels
da.sel(time=slice('2000-01-01', '2000-01-02'))
</pre>

- We can index all variables in a dataset simultaneously, returning a new dataset:
- The drop_sel() method returns a new object with the listed index labels along a dimension dropped: `ds.drop_sel(space=['IN', 'IL'])`
- Use drop_dims() to drop a full dimension from a Dataset. Any variables with these dimensions are also dropped: `drop_dims('time')`. So be careful with it. 
- Masking with where, returning a sama shape but with some lement maksed: `da.where(da.x + da.y < 4)`. You can use the `drop=True` option to drop the masked elements. 
- Selecting values with `isin`: `da.where(da.isin([2, 4]), drop=True)`
- To assign data with indexing
<pre>
# modify one grid point using xr.where()
ds["empty"] = xr.where(
     (ds.coords["lat"] == 20) & (ds.coords["lon"] == 260), 100, ds["empty"])
# or modify a 2D region using xr.where()
mask = (
     (ds.coords["lat"] > 20)
     & (ds.coords["lat"] < 60)
     & (ds.coords["lon"] > 220)
     & (ds.coords["lon"] < 260)
 ) 
ds["empty"] = xr.where(mask, 100, ds["empty"]) 
</pre>

## 2. Interpolation
Arguments like `method` and `fill_values` are the same as numpy routines.

<pre>
# 1-D interpolation  
da = xr.DataArray(np.sin(0.3 * np.arange(12).reshape(4, 3)),
                   [('time', np.arange(4)),
                   ('space', [0.1, 0.2, 0.3])])
da.interp(time=[2.5, 3.5])

# The interpolated data can be merged into the original DataArray by specifying the time periods required.
da_dt64 = xr.DataArray([1, 3],
      [('time', pd.date_range('1/1/2000', '1/3/2000', periods=2))])
da_dt64.interp(time=pd.date_range('1/1/2000', '1/3/2000', periods=3))

# multi-dimenssional interpolation
da.interp(time=[1.5, 2.5], space=[0.15, 0.25])

# Interpolates an xarray object onto the coordinates of another xarray object
interpolated = da.interp_like(other)  # Other is the other xarray object with different coordinates

# Dealing with NaN values
dropped = da.dropna('x')
dropped.interp(x=[0.5, 1.5, 2.5], method='cubic')

#Or, a better solution: fill NaN by interpolate_na()
filled = da.interpolate_na(dim='x') # Fills NaN by interpolating along the specified dimension. After filling NaNs, you can interpolate:
filled.interp(x=[0.5, 1.5, 2.5], method='cubic')
</pre>

## 3. Computation

   1. Arithmetic operations with a single DataArray automatically vectorize (like numpy) over all array values
   2. Numpy’s or scipy’s many [ufunc](https://docs.scipy.org/doc/numpy/reference/ufuncs.html) functions can be directly used on DataArray and DataSet
   3. Use where() to conditionally switch between values: `xr.where(arr > 0, 'positive', 'negative')`
   4.  Aggregation: `arr.sum(dim='x'), arr.min()...`, NaN values are automatically skipped. 
   5. Get axis number: `arr.get_axis_num(dim='y')`
   6. Rolling window operation: `arr.rolling(y=3,center=True,min_periods=2) # along y axis, window=3, center assigned, minimum values required: 2 `
   7. `coarsen()` method supports the block aggregation along multiple dimensions: `da.coarsen(x=7, y=2).mean() # block mean of every 7 rows and two coulmns`
      - If you want to apply a specific function to coordinate, you can pass the function or method name to coord_func option: `da.coarsen(x=7, y=2, coord_func={'x': 'min'}).mean()`
      - `coarsen()` raises an ValueError if the data length is not a multiple of the corresponding window size. You can choose `boundary='trim'` or `boundary='pad'` options for trimming the excess entries or padding nan to insufficient entries.
   8. xarray enforces alignment between index Coordinates (that is, coordinates with the same name as a dimension, marked by `*`) on objects used in binary operations.
   9. Datasets support arithmetic operations by automatically looping over all data variables

## 4. Reshaping and reorganizing data
   1. To reorder dimensions: `ds.transpose('y', 'z', 'x')`. An ellipsis (…) can be use to represent all other dimensions
   2. To expand along a new dimension: `ds.expand_dims('w')`. To remove an axis: `ds.squeeze('w')`
   3. To convert from a Dataset to a DataArray: `arr = ds.to_array()  # broadcasts all data variables in the dataset against each other, then concatenates them along a new dimension into a new array while preserving coordinates.`
   4. To convert from a DataArray to a Dataset: `arr.to_dataset(dim='combined')`

   ## 5. Concatenate, merge, and combine

   1. To combine arrays along existing or new dimension into a larger array, you can use `xr.concat([arr[0], arr[1]], 'x')`. If along a new dimenssion, the arrays will be concatenatd along that new dim which is always inserted as the first dimenssion. `xr.concat([arr[0], arr[1]], 'new_dim')`
   2. To combine variables and coordinates between multiple DataArray and/or Dataset objects, use merge(). 





























