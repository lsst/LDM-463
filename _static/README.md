The `_static/` directory should be used for all images and downloads associated with the document.
Anything included in here is [automatically copied](http://www.sphinx-doc.org/en/stable/config.html?highlight=static#confval-html_static_path) to `_build/html/_static/` without otherwise being modified by Sphinx.

To include an image in reStructuredText,

```
.. image:: /_static/image_file_name.png
```

or a figure:

```
.. figure:: /_static/image_file_name.png

   Caption
```

See the [Developer Guide](http://developer.lsst.io/en/latest/docs/rst_styleguide.html#images-and-figures) for more information.
