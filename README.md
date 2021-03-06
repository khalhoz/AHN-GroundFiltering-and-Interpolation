# Ground Filtering, Interpolation, Hole Filling and Hydro-flattening Methods and Testing Environment

**This is the Python repo of the ground filtering/interpolation team of the AHN3 GEO1101 (synthesis project) group of 2020.**

## In repo:

* `Classification_F1_score` folder _(Khaled & Manos's quantitative ground filtering evaluation)_
* `Vertical_difference` folder _(Khaled & Manos's working folder for quantitative interpolation evaluation)_
* `gf_target_example` folder _(example folder for LAS input files)_
	* `config.json` _(ground filtering configuration file which you can push updates to)_
	* `config_default.json` _(default ground filtering configuration file which should not be modified)_
	* `config_preprocess.json` _(Khaled & Manos's final pipeline - only pre-processes, does not ground-filter)_
	* `fnames.txt` _(list of input files names for the ground filtering program)_
* `README.md` _(this readme file)_
* `gdal_attempt.py` _(a half-baked test program for direct interpolation via GDAL)_
* `gf_main.py` _(main file for ground filtering)_
* `gf_processing.py` _(ground filtering code)_
* `ip_main.py` _(main file for interpolation)_
* `ip_processing.py` _(interpolation code)_
* `las_prepare.py` _(reads LAS file and establishes raster dimensions, part of main program)_
* `Make_tile_vectors.py` _(Lisa's vector tiling program)_
* `split_bbg_into_subfiles.py` _(Lisa's code for splitting the [BGG](https://www.pdok.nl/introductie/-/article/cbs-bestand-bodemgebruik) into subsets of river, sea, and other water body polygons)_
* `vector_prepare.py` _(extracts a vector tile from a vector file, adapted from `Make_tile_vectors.py` and part of main program)_
* `wfs_prepare.py` _(extracts a vector tile from an OGC WFS service, code based on `vector_prepare.py`)_

The testing environment so far includes multiprocessing pool-based implementations of ground filtering/pre-processing via PDAL, TIN-linear and Laplace interpolation via startin, 
constrained Delaunay-based (CDT) TIN-linear and natural neighbour (NN) interpolation via CGAL, radial IDW via GDAL/PDAL and quadrant-based IDW via scipy cKDTree and our own code.
It also includes these post-processing modules so far: flattening the areas of polygons in the raster, and patching in missing pixels.

**NEW STUFF**

* Second release of proper hydro-flattening. It now produces much nicer results, but is a bit slower.
* GeoTIFF raster export now uses `float32`.
* WFS services can also be used as the basis for polygon-based flattening.
* Fixed some bugs. You can now actually add multiple WFS services, as the layer name is now also configurable.

## Current tasks

Finished all planned tasks of the project. All remaining development concerns the scaling team.
Only bug fixes and potential small changes to the new hydro-flattening algorithm to be expected.

## PDAL pipeline entry point (stand-alone ground filtering/pre-processing)

This is a secondary entry point that only runs a PDAL pipeline. It is intended for testing PDAL's ground filtering and pre-processing capabilities.

1. Copy the example configuration files (`config.json`, `fnames.txt`) to your target folder (in which the LAS files are located).
2. Edit the configuration files.
	1. Specify the names of the LAS files in `fnames.txt` as shown in the example.
	2. Modify the PDAL pipeline configuration in `config.json` as needed.
4. Run `gf_main.py` **from the console**. If you run it from an IDE, it will probably not fork the processes properly. It takes one command line argument,
which should be the target folder. An example call in the Windows Anaconda Prompt would be: `python C:/Users/geo-geek/some_folder/gf_main.py C:/Users/geo-geek/target_folder/`

It will deposit the ground filtered tiles as LAS files tagged with `_gf.las` in the target folder.
You can provide an additional CMD argument to modify the default output tag (which is `_gf`). You don't need the underscore, to export `somefile_dsm.las` for example, you may write
`python [file_path_to_main] [target_folder] dsm`. You can also provide a tag for the input config file. For example, for a pre-processing job you may want to call it `config_pre.json`,
in which case you would use:

`python [file_path_to_main] [target_folder] dsm pre`.

## Primary entry point (pre-processing + interpolation + post-processing)

`ip_main.py` is intended to serve as the primary entry point to the program. It includes CMD arguments to run pre-processing via `gf_main.py` and post-processing, optionally.
If you use it to _only_ interpolate pre-processed LAS files you generated using PDAL pipeline entry point, the program will assume that you used the default `_gf.las` tag
for your ground filtered/pre-processed LAS files. It always needs the `fnames.txt` file to be present (and the file names in it shoud **not** be tagged `_gf.las` even if the
PDAL pipeline had already been run).

If you wish to run ground filtering/pre-processing from this entry point, you need to call your config file `config_preprocess.json`. Note that this entry point transfers data
between the pre-processing and interpolation stages _in memory_, and is hence faster than running `gf_main.py` separately beforehand. The PDAL-IDW method has also been adapted
to work so that pre-processing and interpolation form a single pipeline if you run pre-processing from this entry point.

If you wish to run CGAL-CDT or polygon-based post-processing, you will need to have all the water polygons in your target folder in the same folder structures as released by
Lisa. The BAG data set is automatically fetched through WFS requests from Godzilla, so no need to have a local copy of it. You can add your own vector files and WFS URLs if
you want. The custom resources can be specified in the `poly_fpaths` and `wfs_urls` variables, which are available separately in both the `execute_cgal_cdt` and the `flatten_water`
functions in the `ip_processing.py` script. The vector file file paths are relative to the target folder, as illustrated by the default entries, for instance:

`rest_bodies/bbg_rest_of_the_water.shp`

For the post-processing, BAG is disabled by default as we do not want to flatten building footprints. The flattening script is primarily meant for stationary water bodies (although
rivers are also treated like stationary water bodies at the moment, as the real hydro-flattening scripts are not yet functional).

You are advised to run `ip_main.py` **from the console**, preferably from Anaconda Prompt. If you run it from an IDE, it will probably not fork the processes properly.

A key to the CMD call signature of `ip_main.py`:
1. target folder file path
2. bool to indicate if you would like pre-processing to be run before interpolation _(the default value is `False`)_
3. integer to set the post-processing mode, currently these ones are available:
	* **0** _(default, does not run post-processing)_
	* **1** _(runs missing pixel value patching only)_
	* **2** _(runs basic flattening only)_
	* **3** _(runs both patching and basic flattening)_
	* **4** _(runs patching, basic flattening, and the **river hydro-flattening algorithm**)_
4. pixel size (in metres) for interpolation _(the default value is 1)_
5. interpolation method, one of:
	* startin-TINlinear
	* startin-Laplace _(default)_
	* CGAL-NN
	* CGAL-CDT
	* PDAL-IDW
	* IDWquad
6. output format, one of:
	* ASC
	* GeoTIFF _(default)_
7. IDW argument 0
	* _If using PDAL-IDW:_ IDW interpolation radius in metres
	* _If using IDWquad:_ The _starting_ radius/number of neighbours _k_ to query
8. IDW argument 1: IDW interpolation power (exponent) in metres _(both for PDAL-IDW and IDWquad)_
9. IDW argument 2:
	* _If using PDAL-IDW:_ interpolation fallback window size
	* _If using IDWquad:_ minimum number of points to find per quadrant
10. IDW argument 3: query radius/number of neighbours _k_ to query, increment step value _(only for IDWquad)_
11. IDW argument 4: IDWquad method, one of:
	* radial _(for iterative radius increments)_
	* k-nearest _(for iterative increments of how many neighbours to query)_
12. IDW argument 5: IDWquad KD-tree query tolerance value _eps_
13. IDW argument 6: IDWquad maximum number of iterations before declaring no-data and proceeding to next pixel

An example call in the Windows Anaconda Prompt would be:

`python C:/Users/geo-geek/some_folder/ip_main.py C:/Users/geo-geek/target_folder/ False 0 2 startin-TINlinear ASC`

Or for the PDAL-IDW algorithm (using pre-processing) with radius and power values it would be

`python C:/Users/geo-geek/some_folder/ip_main.py C:/Users/geo-geek/target_folder/ True 0 0.5 PDAL-IDW GeoTIFF 10 2`

For using pre-processing and both post-processing modes (and defaults for everything else):

`python C:/Users/geo-geek/some_folder/ip_main.py C:/Users/geo-geek/target_folder/ True 3`

## A word of caution

If you are using an Anaconda virtual environment for PDAL/CGAL, you should first activate the environment in Anaconda prompt and _then_ run the relevant script
from the same prompt. So, for example:
1. `conda activate [environment_name]`
2. `python [file_path_to_main] [argument_1] [argument_2] [...]`

Relative file paths won't work in virtual environments, so make sure you specify the target folder using a full (absolute) file path.
I implemented the program in such a way, that only those modules are imported which are necessary for the given operation. For instance, if you are using neither pre-processing,
nor PDAL-IDW, then you do not need to have PDAL in your Python environment to run the program.

Another word of caution with the outputs is that they all use a fixed no-data value of -9999. This includes the GeoTIFF exporter. To view the results correctly, you should keep in
mind that while the upper bounds of the data will be determined correctly by the viewer software (e.g. QGIS), the lower bound will be -9999. To see the DTM/DSM the program interpolated,
you need to set the lower bound of the colour scale to a higher value relevant to the data. As AHN3 depicts Dutch terrain, you are advised to use a value somewhere between -20 and 0 metres
depending on the tile. Negative elevation values are perfectly possible in The Netherlands.
In QGIS, you do this by right clicking on your raster layer, and clicking on "Properties...". In the window that pops up, you can change the lower bound of the colour scale by
adjusting the value in the field Symbology --> Band Rendering --> Min.

**Note:** ASC export is not currently supported for the PDAL-IDW algorithm. It also does not yet support post-processing (it does support pre-processing though).

**Another note:** You are advised to configure the IDWquad parametrisation **with performance in mind** when first getting started. Otherwise it might take _veeeeeery long_ to finish.

## More about the IDW algorithms

### PDAL-IDW
The PDAL-IDW workflow is actually built on top of GDAL, but since GDAL does not play well with Python data structures, I used the interface that is provided within PDAL's pipeline framework to
implement it. No part of the program currently uses the Python bindings of GDAL directly, but we might need to eventually start working with it. The ellipsoidal IDW features cannot be accessed
through PDAL's interface for GDAL, hence they cannot be used here (hence PDAL-IDW only accepts one radius). There is a neat extra feature in the PDAL interface though, it allows a fallback method
to be used. If you specify a value for an interpolation window (IDW argument 3 above), wherever radial IDW fails, the program will look for values within a square kernel of pixels around the pixel
that is being interpolated (presumably after the first round of true IDW interpolation). For example, if you provide a value of 10 for this argument, it will look for values in a 10x10 square
kernel around the pixel for values, weighting them based on their distance from the pixel that is being interpolated. This can theoretically make the result more or less continuous (a bit more
like the Voronoi and TIN-based methods).

### IDWquad
This is a quadrant-based IDW implementation that is not built on top of third-party software (apart from scipy, from which cKDTree is used). It builds a KD-tree representation of the points of the
input tile and overlays it with a raster of the desired dimensions. For each pixel, it iteratively queries more and more points until it has enough points **per quadrant** to consider an IDW
interpolation reliable. The algorithm can either base its KD-tree queries on k-nearest neighbours to find, or a query radius to search within.
I'll explain how it works by giving some more detail about the parameters, listing them in the same order as in the list of arguments above.

* **Starting interpolation radius/number of neighbours _k_ to query:** The starting value for the radius or number of neighbours to query. This is the value that will be incremented until enough points per quadrant can be found to interpolate a value for the pixel.
* **Interpolation power:** this is self-explanatory, it is the power that is used _if_ IDW interpolation can take place based on the number of neighbours per quadrant that were found.
* **Minimum number of points to find per quadrant:** the minimum number of neighbours to find in _each_ quadrant to interpolate a value for the pixel. E.g. if it is set to one, the algorithm will be satisfied by finding only one neighbour per quadrant. If you set it to a higher value, it will increase the radius/number of neighbours to find until it finds that many neighbours per quadrant (or reaches the maximum number of iterations).
* **Increment value:** the algorithm will increase the radius/number of closest neighbours queried by this amount in each iteration until there are enough neighbours per quadrant to interpolate.
* **Method:** as already explained, you can either specify "radial" to make the algorithm iteratively expand its search radius, or "k-nearest" to iteratively increase the number of nearest neighbours to query.
* **Tolerance value _eps_:** this is very important in terms of performance. As we discussed with Ravi on Discord, the best way to increase performance here is to use _approximate_ KD-tree queries rather than exact queries. Fine tune this parameter to drastically improve performance. This does not affect continuity, it does however affect quality. You can find more info about this on the following pages:
	* If you use radial queries: https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.cKDTree.query_ball_point.html#scipy.spatial.cKDTree.query_ball_point
	* If you use k-nearest queries: https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.cKDTree.query.html#scipy.spatial.cKDTree.query
* **Iteration limit:** you may further fine-tune performance by setting a limit on how many times the algorithm may increment the radius/number of neighbours to find. In some cases this may also drastically improve performance at the cost of continuity.

## More about the proper hydro-flattening algorithm

It is based on the workflow I originally outlined in the proposal, which in turn was based on [this GitHub post](https://github.com/tudelft3d/geo1101-ahn3-admin/issues/2#issuecomment-620467556).

First, the river polygons (in this case the rivers extracted from BBG by Lisa) are skeletonised in Python. Lisa's code does this, it is not yet online
here as of this readme update (but the end result files are on Stack, so this is not needed for testing).
As it was not possible to prune the polygons a 100% in Python and we were running out of time, we completed the skeletonisation process by taking the
shortest path between the starting and ending points of river segments in QGIS, adding important secondary channels manually. Some further manual
fine-tuning was done in QGIS, but we do not expect this to have had a great impact on the overall quality of the results.

The proper hydro-flattening code is part of the post-processing functionalit of the framework that `ip_main.py` provides. It can be enabled by providing
the appropriate post-processing argument (namely, `4`) in the command line call to `ip_main.py`. The code imports the above skeletons, as well as the
original collection of river polygons that the skeletons were computed from. The vector import is done the usual way, i.e. they are cropped to the
size of the tile that is being processed.

The following procedure then takes place:

1. Lines perpendicular to the skeleton at its vertices are intersected with the river boundary (shore lines), so that each cross-section connects two shoreline points through a skeleton vertex. The shoreline intersections are computed as the closest intersection of the skeleton-perpendicular line with the boundary of the river polygons.
2. Each cross section is then associated with an elevation based on Laplace-interpolating at the two shoreline points and the respective skeleton vertex.
3. The cross-section elevations constitute a 1D elevation profile, which is iteratively refined to ensure monotonously decreasing elevation values.
4. The water polygons are then rasterised to form a mask. All pixels that are masked (identified as river pixels) are re-interpolated in the next step.
5. The closest and second-closest cross-section to each river pixel centre is searched for, and the pixel is given a value based on inverse distance weighting using the distances to these two cross sections.

The last step is crucial. It effectively means that we take the river pixel, find which previous and next cross section it falls between, and then
associate an elevation with it that is somewhere between the elevation of these two cross sections. This guarantees (as far as I can tell) that the
elevation will always decrease downstream under normal circumstances. For this, we need the cross-sections to also have monotously decreasing
elevations in the downstream order, which is guaranteed by the third step above. The second release changed the code so that the program not only
searches for the closest two cross-sections, but also ensures that the interpolation point (the river pixel) indeed falls between these two
cross-sections. It does this by "casting rays" to the closest point on each cross-section while computing the distance, and checking whether the ray
intersects any other cross-sections on the way. If it does, the cross-section is excluded from further consideration because the river pixel is
separated from it by another cross-section and therefore it can be neither the previous, not the next cross-section that the program is looking for.
Calculating the nearest point on each cross-section that is tested, is an operation with a high computational complexity. I tried using rays
to the cross-section centroids instead of to their closest points (to speed up the algorithm), but this approach does not work, I am not sure why.

In theory, this should work perfectly, and it does work quite well where the rivers are relatively straight and the spine vertices are sparse.
However, dense spine vertices, especially in river bends, may result in intersecting cross-sections, which will trick the interpolation mechanism
into drawing a small area with locally reversed flow direction. These are generally small-ish cone-shaped artefacts.
Furthermore, the intersections at river-channels are tricky, and will trigger lots of artefacts to appear. This is the result of lots of cross-sections
being present that do not have a consistent orientation and elevation, and which might intersect randomly.

The solution to these issues is quite straightforward. I do not think it is a good idea to further complicate the code, it is complex enough
as it is. I would be smarter to design the skeletons themselves in such a way that they allow the algorithm to perform better. In particular,
the placement of spine vertices in river bends should prevent the generation of intersecting cross-sections, and the placement in multi-channel
river intersections (where the river forks into channels) should be done in a smart way, so that the intersections do not end up criss-crossing one
another and generating all sorts of weird artefacts.

Another solution would be to modify the program itself to pick points on the spine _itself_ rather than rely on the pre-existing vertices on it.
This would allow the spine construction to be kept simple, and would allow the algorithm to select the locations of cross-sections that work
well with the hydro-flattening algorithm.

We recommend this as future work to the client.

On the other hand, the algorithm is robust in the sense that it can handle really weird rivers shapes (such as offshoots, i.e. dock channels and
hydro-power plants without any issues in most places.

## Future work in this repo

_Development has now ended in this repo. Only bug fixes and small changes to be expected._

The current version of this implementation runs as many processes in parallel as there are input files, hence using all processor cores when there are at least as many files as there are cores in
the given system. This multiprocessing implementation is based on Python built-in multiprocessing pools. A queue-based implementation would probably work better, but this is something for the scaling group to look at. _No final decision yet._

We should probably also experiment around with the `filters.pmf` method that is provided by PDAL. The example parametrisation uses `filters.smrf`. You can start experimenting with this simply by changing the JSON parametrisation, the code does not need to be edited.
There's further guidance in `gf_main.py` on what's what in `config.json`. _(Completed.)_

I actually managed to implement an program that uses the GDAL interface directly (see the file `GDAL_attempt.py`) which is ready to read OGR vector files (for example ESRI shapefiles) and interpolate them
using GDAL's Python bindings. This would give access to the full IDW functionality of GDAL, but there is a problem: I implemented this because I though we could simply set the PDAL ground filtering implementation
to export into OGR files rather than LAS files and then simply use those as input for GDAL (GDAL is not compatible with LAS files). However, it turns out that there is a bug in the OGR writer of PDAL. It crashes
Python randomly while exporting. I can run the same code 10 times in a loop and 3-4 out of 10 attempts it will export correctly, but in the other cases it will crash. Unfortunately this makes it useless to us.
I have been trying to get this to work in Python 3.8, which could potentially be the source of the issues. I'll try to experiment with other combinations of Python and PDAL versions in the future.
_Now that I also implemented quadrant-based IDW, perhaps we might not need this after all?_

A further idea would be to implement no-data mask layers for the GeoTIFF exporter. This would make them easier to visualise, as the no-data pixels would be masked out by the no-data
layer, rather than simply being marked as no-data by the fixed value -9999. _Not being worked on at the moment._