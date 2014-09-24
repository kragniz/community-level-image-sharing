=================================
Add community-level image sharing
=================================

https://blueprints.launchpad.net/glance/+spec/community-level-v2-image-sharing

Add a new feature which allows a user to share an image with all other tenants.
These 'community' images do no appear in image listings for a user until that
user has accepted the image.


Problem description
===================

Currently, images must either be public, private or shared. There is no
provision for an image to be available for use by users other than the owner of
the image without making the image either public or explicitly shared by the
owner of the image with other users. If the number of users who require access
to the image is large, the overhead of explicit sharing can become a burden on
the owner of the image.


Use cases
---------

 1. An open source project wants to make a prepared VM image available so in
    order to use the project's software, all you have to do is boot an instance
    from their image. They are too busy writing software to worry about the
    hassle of maintaining a list of "customers" as they'd have to do with
    current v2 image sharing. At the same time, the cloud provider hosting the
    prepared image doesn't want to make this image public, as that would imply
    a support relation that doesn't exist between image consumers and the cloud
    provider.

 2. A cloud provider wants to discourage users from using a public image that
    is getting old, for example an image which is missing some security patches
    or has no more vendor support, but doesn't want to delete the image because
    some users may still require it. Reasons for requiring the old image could
    include needing to rebuild a server, or the user has custom patches for
    that particular image. The vendor could make the image a "community' image,
    which means that:

    a) It won't appear in user's default image lists. This means they won't
    know about it unless they are motivated to seek it out, for example by
    asking other users for the UUID.

    b) Since it's not a "public" image any more, it doesn't imply the same
    level of support as a provider-supplied public image.


Proposed change
===============

An additional value for the ``visibility`` enum will be added in the JSON
schema, named ``'community'``.  This makes the possible values of
``visibility``:

.. code:: python

    ['public', 'private', 'shared', 'community']

An image with with a certain value for ``visibility`` has the following
properties:

* **public**: All users:

  - have this image in default ``image-list``

  - can see ``image-detail`` for this image

  - can boot from this image

* **private**: Users with ``tenantId == tenantId(owner)`` only:

  - have this image in the default ``image-list``

  - see ``image-detail`` for this image

  - can boot from this image

* **shared**:

  - Users with ``tenantId == tenantId(owner)``

    + have this image in the default ``image-list``

    + see ``image-detail`` for this image

    + can boot from this image

  - Users with ``tenantId`` in the ``member-list`` of the image

    + can see ``image-detail`` for this image

    + can boot from this image

  - Users with ``tenantId`` in the ``member-list`` with ``member_status == 'accepted'``

    + have this image in their default ``image-list``

* **community**:

  - All users:

    + can see ``image-detail`` for this image

    + can boot from this image

  - Users with ``tenantId`` in the ``member-list`` of the image with ``member_status == 'accepted'``

    + have this image in their default ``image-list``


Alternatives
------------

Adding ``"*"`` as a membership record
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An alternative method of implementing this functionality is to add a membership
record for an image that has a target of ``"*"`` (i.e. it is shared with all
tenants) but with ``membership_status = "community"``.

Community images can then be discovered with the query

::

  GET /v2/images?membership_status=community


The downsides to this is it makes filtering for community images harder and
less explicit.


Adding image aliases
~~~~~~~~~~~~~~~~~~~~

A completely different way of solving the usecase for cloud providers
(discouraging users from using an older version of a public image) could be to
create a mechanism to make an image alias, which could point at the newest
version of the public image. There is a abandoned blueprint for this feature
[#]_. This, however, is much harder to implement and does not fit with the
other usecases.

.. [#] https://blueprints.launchpad.net/glance/+spec/glance-image-aliases


Data model impact
-----------------

Schema changes
~~~~~~~~~~~~~~

The visibility of the image will be will be stored in the database in the
images table in a new column named ``visibility``. This contains one of the
values in the set of ``['public', 'private', 'shared', 'community']``.

The default value for ``visibility`` is ``'private'``.

This change makes the ``is_public`` column redundant, so it will be removed.


Database migrations
~~~~~~~~~~~~~~~~~~~

1. All rows with ``is_public == 1``:

   - ``visibility = 'public'``

2. For all unique ``image_id`` in ``image_members`` where ``deleted != 1``:

   - ``visibility = 'shared'``

3. For all rows with ``visibility == null``:

   - ``visibility = 'private'``
