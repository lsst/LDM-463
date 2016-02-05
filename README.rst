#######
LDM-463
#######

==================
Data Access Design
==================

This is a working repository for *LDM-463: Data Access Design*.

* Read the living document on the web: http://ldm-463.lsst.io 

Working with this document
--------------------------

.. code::

   git clone git@github.com:lsst/LDM-463.git
   cd LDM-463
   pip install -r requirements.txt
   make html

The built site can be viewed by opening ``_build/html/index.html`` in
your web browser.

Whenever you push to ``master``, readthedocs.org will build and host the
document at http://ldm-463.lsst.io

Editing metadata
----------------

Metadata, such as document version, title, date of last edit, and
authors, are maintained in the ``metadata.yaml`` file
