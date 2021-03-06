.. include:: references.txt

.. _astropy-coordinates-separations-matching:

Separations, Catalog Matching, and Related Functionality
--------------------------------------------------------

`astropy.coordinates` contains commonly-used tools for comparing or
matching coordinate objects.  Of particular importance are those for
determining separations between coordinates and those for matching a
coordinate (or coordinates) to a catalog.  These are mainly implemented
as methods on the coordinate objects.

Separations
===========

The on-sky separation is easily computed with the
:meth:`astropy.coordinates.BaseCoordinateFrame.separation` or
:meth:`astropy.coordinates.SkyCoord.separation` methods,
which computes the great-circle distance (*not* the small-angle
approximation)::

    >>> import numpy as np
    >>> from astropy import units as u
    >>> from astropy.coordinates import SkyCoord
    >>> c1 = SkyCoord('5h23m34.5s', '-69d45m22s', frame='icrs')
    >>> c2 = SkyCoord('0h52m44.8s', '-72d49m43s', frame='fk5')
    >>> sep = c1.separation(c2)
    >>> sep  # doctest: +FLOAT_CMP
    <Angle 20.74611447604398 deg>

The returned object is an `~astropy.coordinates.Angle` instance, so it
is straightforward to access the angle in any of several equivalent angular
units::

    >>> sep.radian  # doctest: +FLOAT_CMP
    0.36208800460262575
    >>> sep.hour  # doctest: +FLOAT_CMP
    1.3830742984029323
    >>> sep.arcminute  # doctest: +FLOAT_CMP
    1244.7668685626388
    >>> sep.arcsecond  # doctest: +FLOAT_CMP
    74686.01211375833

Also note that the two input coordinates were not in the same frame -
one is  automatically converted to match the other, ensuring that even
though they are  in different frames, the separation is determined
consistently.

In addition to the on-sky separation described above,
:meth:`astropy.coordinates.BaseCoordinateFrame.separation_3d` or
:meth:`astropy.coordinates.SkyCoord.separation_3d` methods will
determine the 3D distance between two coordinates that have ``distance``
defined::

    >>> from astropy.coordinates import SkyCoord
    >>> c1 = SkyCoord('5h23m34.5s', '-69d45m22s', distance=70*u.kpc, frame='icrs')
    >>> c2 = SkyCoord('0h52m44.8s', '-72d49m43s', distance=80*u.kpc, frame='icrs')
    >>> sep = c1.separation_3d(c2)
    >>> sep  # doctest: +FLOAT_CMP
    <Distance 28.743988157814094 kpc>

There is also a :meth:`~astropy.coordinates.SkyCoord.spherical_offsets_to` method
for computing angular offsets (e.g., small shifts like you might give a
telescope operator to move from a bright star to a fainter target.)::

    >>> from astropy.coordinates import SkyCoord
    >>> bright_star = SkyCoord('8h50m59.75s', '+11d39m22.15s', frame='icrs')
    >>> faint_galaxy = SkyCoord('8h50m47.92s', '+11d39m32.74s', frame='icrs')
    >>> dra, ddec = bright_star.spherical_offsets_to(faint_galaxy)
    >>> dra.to(u.arcsec)  # doctest: +FLOAT_CMP
    <Angle -173.78873354064126 arcsec>
    >>> ddec.to(u.arcsec)  # doctest: +FLOAT_CMP
    <Angle 10.605103417374696 arcsec>

.. _astropy-skyoffset-frames:

"Sky Offset" Frames
===================

To extend the concept of spherical offsets, `~astropy.coordinates` has
a frame class :class:`~astropy.coordinates.builtin_frames.skyoffset.SkyOffsetFrame`
which creates distinct frames that are centered on a specific point.
These are known as "sky offset frames", as they are a convenient way to create
a frame centered on an arbitrary position on the sky, suitable for computing
positional offsets (e.g., for astrometry)::

    >>> from astropy.coordinates import SkyOffsetFrame, ICRS
    >>> center = ICRS(10*u.deg, 45*u.deg)
    >>> center.transform_to(SkyOffsetFrame(origin=center))  # doctest: +SKIP
    <SkyOffsetICRS Coordinate (rotation=0.0 deg, origin=<ICRS Coordinate: (ra, dec) in deg
        (10.0, 45.0)>): (lon, lat) in deg
        (0.0, 0.0)>
    >>> target = ICRS(11*u.deg, 46*u.deg)
    >>> target.transform_to(SkyOffsetFrame(origin=center))  # doctest: +FLOAT_CMP
    <SkyOffsetICRS Coordinate (rotation=0.0 deg, origin=<ICRS Coordinate: (ra, dec) in deg
        (10.0, 45.0)>): (lon, lat) in deg
        (0.69474685, 1.00428706)>


Alternatively, the convenience method
:meth:`~astropy.coordinates.SkyCoord.skyoffset_frame` lets you create an skyoffset
frame from an already-existing |SkyCoord|::

    >>> center = SkyCoord(10*u.deg, 45*u.deg)
    >>> aframe = center.skyoffset_frame()
    >>> target.transform_to(aframe)  # doctest: +FLOAT_CMP
    <SkyOffsetICRS Coordinate (rotation=0.0 deg, origin=<ICRS Coordinate: (ra, dec) in deg
        (10.0, 45.0)>): (lon, lat) in deg
        (0.69474685, 1.00428706)>
    >>> other = SkyCoord(9*u.deg, 44*u.deg, frame='fk5')
    >>> other.transform_to(aframe)  # doctest: +FLOAT_CMP
    <SkyCoord (SkyOffsetICRS: rotation=0.0 deg, origin=<ICRS Coordinate: (ra, dec) in deg
        (10.0, 45.0)>): (lon, lat) in deg
        (-0.71943945, -0.99556216)>

.. note ::
    While sky offset frames *appear* to be all the same class, this not the
    case: the sky offset frame for each different type of frame for ``origin`` is
    actually a distinct class.  E.g., ``SkyOffsetFrame(origin=ICRS(...))``
    yields an object of class ``SkyOffsetICRS``, *not* ``SkyOffsetFrame``.
    While this is not important for most uses of this class, it is important for
    things like type-checking, because something like
    ``SkyOffsetFrame(origin=ICRS(...)).__class__ is SkyOffsetFrame`` will
    *not* be ``True``, as it would be for most classes.

.. _astropy-coordinates-matching:

Matching Catalogs
=================

`~astropy.coordinates` supports leverages the coordinate framework to make it
straightforward to find the closest coordinates in a catalog to a desired set
of other coordinates. For example, assuming ``ra1``/``dec1`` and
``ra2``/``dec2`` are numpy arrays loaded from some file::

    >>> from astropy.coordinates import SkyCoord
    >>> from astropy import units as u
    >>> c = SkyCoord(ra=ra1*u.degree, dec=dec1*u.degree)  # doctest: +SKIP
    >>> catalog = SkyCoord(ra=ra2*u.degree, dec=dec2*u.degree)  # doctest: +SKIP
    >>> idx, d2d, d3d = c.match_to_catalog_sky(catalog)  # doctest: +SKIP

You can also find the nearest 3d matches, different from the on-sky
separation shown above only when the coordinates were initialized with
a ``distance``::

    >>> c = SkyCoord(ra=ra1*u.degree, dec=dec1*u.degree, distance=distance1*u.kpc)  # doctest: +SKIP
    >>> catalog = SkyCoord(ra=ra2*u.degree, dec=dec2*u.degree, distance=distance2*u.kpc)  # doctest: +SKIP
    >>> idx, d2d, d3d = c.match_to_catalog_3d(catalog)  # doctest: +SKIP

Now ``idx`` are indices into ``catalog`` that are the closest objects to each
of the coordinates in ``c``, ``d2d`` are the on-sky distances between them, and
``d3d`` are the 3-dimensional distances.  Because coordinate objects support
indexing, ``idx`` enables easy access to the matched set of coordinates in
the catalog::

    >>> matches = catalog[idx]  # doctest: +SKIP
    >>> (matches.separation_3d(c) == d3d).all()  # doctest: +SKIP
    True
    >>> dra, ddec = c.spherical_offsets_to(matches)  # doctest: +SKIP

This functionality can also be accessed from the
:func:`~astropy.coordinates.match_coordinates_sky` and
:func:`~astropy.coordinates.match_coordinates_3d` functions. These
will work on either |skycoord| objects *or* the lower-level frame classes::

    >>> from astropy.coordinates import match_coordinates_sky
    >>> idx, d2d, d3d = match_coordinates_sky(c, catalog)  # doctest: +SKIP
    >>> idx, d2d, d3d = match_coordinates_sky(c.frame, catalog.frame)  # doctest: +SKIP

.. _astropy-searching-coordinates:

Searching Around Coordinates
============================

Closely-related functionality can be used to search for *all* coordinates within
a certain distance (either 3D distance or on-sky) of another set of coordinates.
The ``search_around_*`` methods (and functions) provide this functionality,
with an interface very similar to ``match_coordinates_*``::

    >>> idxc, idxcatalog, d2d, d3d = catalog.search_around_sky(c, 1*u.deg)  # doctest: +SKIP
    >>> np.all(d2d < 1*u.deg)  # doctest: +SKIP
    True
    >>> idxc, idxcatalog, d2d, d3d = catalog.search_around_3d(c, 1*u.kpc)  # doctest: +SKIP
    >>> np.all(d3d < 1*u.kpc)  # doctest: +SKIP
    True

The key difference for these methods is that there can be multiple (or no)
matches in ``catalog`` around any locations in ``c``.  Hence, indices into both
``c`` and ``catalog`` are returned instead of just indices into ``catalog``.
These can then be indexed back into the two |skycoord| objects, or, for that
matter, any array with the same order::

    >>> np.all(c[idxc].separation(catalog[idxcatalog]) == d2d)  # doctest: +SKIP
    True
    >>> np.all(c[idxc].separation_3d(catalog[idxcatalog]) == d3d)  # doctest: +SKIP
    True
    >>> print catalog_objectnames[idxcatalog]  # doctest: +SKIP
    ['NGC 1234' 'NGC 4567' ...]

Note, though, that this dual-indexing means that ``search_around_*`` does not
work well if one of the coordinates is a scalar, because the returned index
would not make sense for a scalar::

    >>> scalarc = SkyCoord(1*u.deg, 2*u.deg)  # doctest: +SKIP
    >>> idxscalarc, idxcatalog, d2d, d3d = catalog.search_around_sky(scalarc, 1*u.deg)  # THIS DOESN'T ACTUALLY WORK  # doctest: +SKIP
    >>> scalarc[idxscalarc]  # doctest: +SKIP
    IndexError: 0-d arrays can't be indexed

As a result (and because the ``search_around_*`` algorithm is inefficient in
the scalar case, anyway), the best approach for this scenario is to instead
use the ``separation*`` methods::

    >>> d2d = scalarc.separation(catalog)  # doctest: +SKIP
    >>> catalogmsk = d2d < 1*u.deg  # doctest: +SKIP
    >>> d3d = scalarc.separation_3d(catalog)  # doctest: +SKIP
    >>> catalog3dmsk = d3d < 1*u.kpc  # doctest: +SKIP

The resulting ``catalogmsk`` or ``catalog3dmsk`` variables are boolean arrays
rather than arrays of indices, but in practice they usually can be used in
the same way as ``idxcatalog`` from the above examples.  If you definitely do
need indices instead of boolean masks, you can do:

    >>> idxcatalog = np.where(catalogmsk)[0]  # doctest: +SKIP
    >>> idxcatalog3d = np.where(catalog3dmsk)[0]  # doctest: +SKIP
