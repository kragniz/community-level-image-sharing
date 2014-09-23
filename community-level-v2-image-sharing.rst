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
to the image is large, the explicit sharing can become a burden on the owner of
the image.


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
