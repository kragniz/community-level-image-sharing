..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Add community-level image sharing
=================================

https://blueprints.launchpad.net/glance/+spec/community-level-v2-image-sharing

Add a new feature to the v2 API which allows a user to share an image with all
other tenants.  These "community" images do not appear in image listings for a
user until that user has accepted the image.

This new feature adds a new value for the visibility of an image to the
underlying data model, named ``community``.


Problem description
===================

Currently, there is no provision for an image to be available for use by users
other than the owner of the image, unless the image is either made public or
explicitly shared by the owner of the image with other users. If the number of
users who require access to the image is large, the overhead of explicit
sharing can become a burden on the owner of the image.

Public images appear in ``image-list`` for all users, which can be undesirable:

1. An open source project wants to make a prepared VM image available so users
   simply boot an instance from that image to use the project's software. The
   developers of the project are too busy writing software to worry about the
   hassle of maintaining a list of "customers" as they'd have to do with current
   v2 image sharing. At the same time, the cloud provider hosting the prepared
   image doesn't want to make this image public, as that would imply a support
   relation that doesn't exist between image consumers and the cloud provider.

2. A cloud provider wants to discourage users from using a public image that is
   getting old, for example an image which is missing some security patches or is
   no longer supported by the vendor, but doesn't want to delete the image because
   some users may still require it. Users could need the old image to rebuild a
   server, or because they have custom patches for that, particular image.

   In either case, the vendor could make the image a "community' image and
   alleviate the challenges presented above. This means that:

   a) The community image won't appear in user's default image lists. This
   means they won't know about it unless they are motivated to seek it out, for
   example by asking other users for the UUID.

   b) Since the image is no longer "public," it doesn't imply the same level of
   support from the vendor as a provider-supplied public image.


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

* **private**: Users with ``tenant_id == owner_tenant_id`` only:

  - have this image in the default ``image-list``

  - see ``image-detail`` for this image

  - can boot from this image

* **shared**:

  - Users with ``tenant_id == owner_tenant_id``

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

Adding image aliases
~~~~~~~~~~~~~~~~~~~~

A completely different way of solving the usecase for cloud providers
(discouraging users from using an older version of a public image) could be to
create a mechanism to make an image alias, which could point at the newest
version of the public image. There is a abandoned blueprint for this feature
[#]_. This, however, is much harder to implement and does not fit with the
other use cases.

.. [#] https://blueprints.launchpad.net/glance/+spec/glance-image-aliases


Adding a special case of image sharing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Another method of implementing this functionality is to add a membership record
for an image that has a target of ``"community"`` (i.e. it is shared with all
tenants) with ``membership_status = "community"``. This marks it as a community
image very simply and requires few modifications to existing code.

This respects the current anti-spam provisions in the glance v2 api; when an
image owner makes an image a "community" image, any other tenant should be able
to boot an instance from that image, but the image will not show up in any
tenant's default image-list.

The downsides to using this method are a few corner cases which result in
surprising API calls and some less than desirable mappings between api level
and data model level values of visibility.


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

REST API impact
---------------

Image discovery
~~~~~~~~~~~~~~~

Community images will be displayed in an image listing when the visibility:w

filter is set to ``community``. ::

    GET /v2/images?visibility=community


All other appropriate filters will be respected. Of note is the use of an
``owner`` parameter. This, when supplied together with the
``visibility=community`` filter, allows a user to request only those community
images owned by that particular tenant: ::

    GET /v2/images?visibility=community&owner={owner_tenant_id}


Accepting a community Image
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Having to filter the results each time to find a particular image is
inconvenient, particularity if that image is often used. To remove this
inconvenience, users are allowed to "bookmark" a particular community image.
This image then appears in the user's image-list. This functionality operates
in the same way that v2 image sharing works, but the owner of the image does
not have to explicitly maintain relationships with each of the new image
consumers.

If a consumer of a community image would like to bookmark it, they must accept
the image so it will appear in their image-list

For this, the existing image-sharing API calls will be used:

Accept the image: ::

       PUT /v2/images/{image_id}/members/{consumer_tenant_id}

Request body:

.. code:: json

   {"status": "accepted"}

The response and other behaviour remains the same as was previously defined for
this call.


Making an image a "community image"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An admin or the owner of an image (depending on policy settings) can use the
existing image-update call, changing the image's visibility to ``'community'``: ::

    PATCH /v2/images/{image_id}

Request body: ::

    [{ "op": "replace", "path": "/visibility", "value": "community" }]

The response and other behaviour remains the same as was previously defined for
this call.


Removing a community image
~~~~~~~~~~~~~~~~~~~~~~~~~~

A community image can be removed from community-level access by also using the
image-update call. Instead of setting it to ``'community'`` as before, we set
it to ``'private'``: ::

    PATCH /v2/images/{image_id}

Request body: ::

    [{ "op": "replace", "path": "/visibility", "value": "private" }]

As in all the above cases, the response and other behaviour remains the same as
was previously defined for this call.

Security impact
---------------

See "other deployer impact".

Notifications impact
--------------------

None

Other end user impact
---------------------


Client changes
~~~~~~~~~~~~~~

Python-glanceclient will be updated to expose this feature. An option to
``glance image-update`` will be added named ``--visibility
<VISIBILITY_STATUS>``, where ``VISIBILITY_STATUS`` may be one of ``{public,
private, community}``.

For example, to make an image a community image:

.. code:: bash

    $ glance image-update --visibility community <IMAGE>

To make the image private again:

.. code:: bash

    $ glance image-update --visibility private <IMAGE>


Membership behaviour
~~~~~~~~~~~~~~~~~~~~

Moving from community to public retains the list of members the image currently
contains. This maintains constancy with the current membership behaviour.

Performance Impact
------------------

None

Other deployer impact
---------------------

The ability to provide images to other users has the potential for abuse. A
user could provide a malicious image to a large audience. For this reason, the
ability to create community images is moderated using policy.json. A new rule
will be created, which has a default configuration of ``[role:admin]``:

- ``publicize_community_image`` - Share image with all tenants

  + ``POST /v2/images/{image_id}/members`` with ``member`` = ``community``

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kragniz

Work Items
----------

- Refactor db api to use ``visibility`` rather than ``is_public``

- Add functionality for storing the community state in the interfaces to both db
  backends:

  + sqlalchemy

  + simple

- Add functionality to enable this and accept the image using the api

- Add unit tests to test various inputs to the api

- Add functional tests for the lifecycle of community images

- Update glanceclient to use the new api functionality


Dependencies
============

None

Testing
=======

A tempest test must be added to cover creating a community image and it
transitioning between public and private states.


Documentation Impact
====================

New features must be documented in both glance and python-glanceclient.

References
==========

None
