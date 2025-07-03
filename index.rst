######################################
User metadata for the Science Platform
######################################

.. abstract::

   The Rubin Science Platform will store various metadata about each user, either created by the Science Platform (such as some identifiers) or collected from the relevant identity provider.
   This document describes the metadata associated with users and its sources and constraints, such as numeric ranges for UIDs and GIDs and valid patterns for usernames and group names.

This document is divided into three sections, one for the :abbr:`IDF (Interim Data Facility)` and :abbr:`USDAC (United States Data Access Center)`, one for Telescope and Site deployments, and one for the :abbr:`USDF (United States Data Facility)`.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are DMTN-234_, which describes the high-level design; DMTN-224_, which describes the implementation; and SQR-069_, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/
.. _SQR-069: https://sqr-069.lsst.io/

IDF and USDAC
=============

User metadata
-------------

We expect to add additional metadata, such as whether a user has accepted the Acceptable Use Policy, before the production release.
This tech note will be updated when we add additional metadata.

Username
^^^^^^^^

**Source**: User chooses their (unique) username during enrollment.

**Storage**: In COmanage, it is stored as an identifier associated with the user's record.
COmanage stores it in LDAP as ``voPersonApplicationUID``.
During authentication, CILogon maps the user's identity to an LDAP record, retrieves the username from that attribute, and stores it in the ``username`` claim of the resulting OpenID Connect ID token.
Gafaelfawr reads it from there and stores it in the user's token in Redis and its SQL database.

**Constraints**: Must consist solely of lowercase ASCII letters, numbers, and dash (``-``).
Must be at least two characters long.
Must contain at least one lowercase ASCII letter.
Must not start or end with a dash
Must not contain two consecutive dashes. [#]_
Usernames for bot users (users created for automated processes or services, not for human users) must begin with ``bot-``.

.. [#] Regular expression: ``^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$``

Numeric UID
^^^^^^^^^^^

**Source**: Assigned by the Science Platform on first use of an account.
Each deployment IDF or USDAC deployment of the Science Platform has a separate UID assignment pool to map usernames to UIDs.

**Storage**: One document per user is stored in `Google Firestore`_
This currently contains only the UID, but in the future may contain other metadata maintained by the Science Platform rather than COmanage and CILogon.

.. _Google Firestore: https://cloud.google.com/firestore

**Constraints**: See :ref:`UID and GID assignment <uid-gid-assignment>`.
UIDs are unique and are intended to never be reused.
Once assigned, the UID for a given account never changes, even if the username is changed.

Primary GID
^^^^^^^^^^^

**Source**: The user's primary GID will always be the same as their UID.

Full name
^^^^^^^^^

**Source**: Taken from the user's federated identity provider during enrollment.
The user may choose to enter a new name.
Insofar as possible, the Rubin Science Platform will only record the user's entire name of choice as a single text field, not divided into components such as given name and family name.
COmanage currently does not properly support this, and may represent the name in components, but the Science Platform will attempt to use the combined form only.

**Storage**: The names from each associated federated identity are stored in COmanage, along with any name the user chooses to enter.
In COmanage, this is stored as the user's preferred name, and COmanage is configured to write that name to LDAP as the ``displayName`` attribute.
It is retrieved from there by the Science Platform using the token API.

**Constraints**: Any valid UTF-8 string of reasonable length without control characters.
No assumptions are made about the structure of the name.

Email address
^^^^^^^^^^^^^

**Source**: Taken from the user's federated identity provider during enrollment.
The user may choose to enter a new email address.

**Storage**: The email addresses from each associated federated identity are stored in COmanage, along with any email the user chooses to enter.
The user can select which one is preferred, which is the one that will be used by the Science Platform.
This is stored as the ``mail`` attribute in the user's LDAP record as maintained by COmanage, and is retrieved from there by the Science Platform using the token API.

**Constraints**: Must be a syntactically-valid `RFC 5322 addr-spec <https://datatracker.ietf.org/doc/html/rfc5322#section-3.4.1>`__.
COmanage will confirm the validity of the email address during enrollment by sending the user an email and having them follow a link in the email.

Group membership
^^^^^^^^^^^^^^^^

**Source**: COmanage records the user's group membership (except in their default group).
Users are added to groups by group owners, and may be added to groups based on automated rules triggering off of their affiliation data.
Users are also automatically a member of a default group with the same name as the username and the same GID as the user's UID.
This default group is added by the Science Platform and not recorded in COmanage or LDAP.

**Storage**: COmanage stores the user's group membership information and provides it in the LDAP server it maintains, as ``hasMember`` attributes in a groups tree.
The groups of which a user is a member are also stored as ``isMemberOf`` attributes in the person record.
Group membership information is retrieved from LDAP each time it is needed, with the user's default group added before passing the group information along to other systems.

.. warning::

   The scopes of an authentication token are calculated from the group membership at the time of initial user authentication and are not affected by subsequent changes to the user's group membership until that token expires.

**Constraints**: There is no inherent limit in the number of groups a user may be a member of, but be aware that NFS only allows a user to be a member of 16 groups, one of which is the user's default group.
Group memberships above 16 may be ignored by the NFS server.

Group metadata
--------------

Group name
^^^^^^^^^^

(The below rules only apply to additional groups, not the default group with the same name as the username.)

**Source**: Groups are named in COmanage when they are created.

**Storage**: Group names are stored where user group membership is stored.

**Constraints**: All group names must begin with ``g_``.
Group names must consist of lowercase ASCII letters and numbers, period (``.``), dash (``-``), and underscore (``_``), and must be at most 32 characters long. [#]_

.. [#] Regular expression: ``^g_[a-z0-9._-]{1,30}$``

Numeric GID
^^^^^^^^^^^

**Source**: Assigned by the Science Platform on first use of a group.
Each deployment IDF or USDAC deployment of the Science Platform has a separate UID assignment pool to map group names to UIDs.

**Storage**: One document per group is stored in `Google Firestore`_
This currently contains only the GID, but in the future may contain other metadata maintained by the Science Platform rather than COmanage and CILogon.

**Constraints**: See :ref:`UID and GID assignment <uid-gid-assignment>`.
GIDs are unique and are intended to never be reused.
Once assigned, the GID for a given group never changes, even if the group name is changed.

.. _uid-gid-assignment:

UID and GID assignment
----------------------

The Science Platform uses a POSIX file system for some storage.
Access control in that file system is done via numeric UIDs and GIDs.
Each user must therefore be assigned a unique UID, and each group must be assigned a unique GID.

Each user must also have a default group.
Following the now-standard Linux convention, that default group will have the same name as the user and will contain only the user.
That group must also have a unique GID.

For convenience, the GID of the user's default group will always match the user's UID.

The Science Platform requires support for at least 31-bit UIDs and GIDs and makes no attempt to support platforms with 16-bit UIDs or GIDs.
We can therefore take advantage of the increased UID and GID space up to 2,147,483,648.

UID and GID space is divided into the following ranges:

0-99
    Reserved for the container operating system.

100-999
    Reserved for users created by packages installed in containers, and for the use of some containers that use default UIDs in the high 900s.

1000-999999
    Reserved for users created inside the container image.
    Most containers use UID 1000 as a default user.
    Note that 65534 is reserved by the operating system.

100000-199999
    UIDs for bot users and the corresponding GID for the bot user's default group.

200000-299999
    GIDs for groups other than the user's default group.

300000-999999
    User UIDs and the corresponding GID for the user's default group.

1000000-2147483647
    Reserved for future use.

UIDs and GIDs are assigned on first use of a given user or group in a Science Platform deployment.
They are not shared between Science Platform deployments.

Once a given UID or GID has been used, it will never be reused for a different user or group.

COmanage does support assigning UIDs and GIDs, but the configuration complexity required is higher, and our assignment needs are a somewhat awkward fit for COmanage's capabilities.
We therefore will do UID and GID assignment independently of COmanage.

Telescope and Site (IPA)
========================

All Telescope and Site Science Platform instances are being migrated to local IPA as the source of authentication and user metadata.
Until this migration is complete, some will still use GitHub.
See :ref:`ts-github` for rules for those instances.

User metadata
-------------

Username
^^^^^^^^

**Source**: The value of the ``preferred_username`` claim in the ID token returned by the OpenID Connect authentication protocol.
This will correspond to the user's IPA username.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Must consist solely of lowercase ASCII letters, numbers, and dash (``-``), must not start or end with a dash, and must not contain two consecutive dashes. [#]_
Must not consist entirely of numbers.

.. [#] Regular expression: ``^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$``

Numeric UID
^^^^^^^^^^^

**Source**: The ``uidNumber`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Primary GID
^^^^^^^^^^^

**Source**: The ``gidNumber`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Full name
^^^^^^^^^

**Source**: The ``displayName`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.
No assumptions are made about the structure of the name.

Email address
^^^^^^^^^^^^^

**Source**: The ``mail`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Group membership
^^^^^^^^^^^^^^^^

**Source**: All groups in LDAP for which the user's DN is listed as a member.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

.. warning::

   The scopes of an authentication token are calculated from the group membership at the time of initial user authentication and are not affected by subsequent changes to the user's group membership until that token expires.

**Constraints**: There is no inherent limit in the number of groups a user may be a member of, but be aware that NFS only allows a user to be a member of 16 groups, one of which is the user's default group.
Group memberships above 16 may be ignored by the NFS server.

Group metadata
--------------

Group name
^^^^^^^^^^

**Source**: The ``cn`` attribute of the LDAP record for the group.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Group names must consist of ASCII letters (upper- or lowercase) and numbers, period (``.``), dash (``-``), and underscore (``_``), must begin with a letter or number, must contain at least one letter, and must be at most 32 characters long. [#]_

.. [#] Regular expression: ``^[a-zA-Z0-9][a-zA-Z0-9._-]*[a-zA-Z][a-zA-Z0-9._-]*$``

Numeric GID
^^^^^^^^^^^

**Source**: The ``gidNumber`` attribute of the LDAP record for the group.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

.. _ts-github:

Telescope and Site (GitHub)
===========================

Currently, some Telescope and Site deployments use GitHub for authentication.
These are all expected to switch to IPA in the future.

User metadata
-------------

Username
^^^^^^^^

**Source**: The user's GitHub username converted to all lowercase.

**Storage**: The username is used as a unique key for the user in all identity management systems.

**Constraints**: Must consist solely of lowercase ASCII letters, numbers, and dash (``-``), must not start or end with a dash, and must not contain two consecutive dashes. [#]_
Must not consist entirely of numbers.

.. [#] Regular expression: ``^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$``

Numeric UID
^^^^^^^^^^^

**Source**: UID assigned by GitHub.
For bot users that do not exist in GitHub, we make up a UID when an authentication token for the bot user is created and hope it doesn't conflict with a meaningful GitHub user.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Whatever constraints are used by GitHub to assign UIDs.

Primary GID
^^^^^^^^^^^

**Source**: The user's primary GID will always be the same as their UID.

Full name
^^^^^^^^^

**Source**: Taken from the GitHub account metadata if it is released to the Science Platform instance.
If the user does not release their name, no name metadata will be available.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Any valid UTF-8 string of reasonable length without control characters.
No assumptions are made about the structure of the name.

Email address
^^^^^^^^^^^^^

**Source**: Taken from the GitHub account metadata if it is released to the Science Platform instance.
If the user does not release their email address, no email address metadata will be available.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Whatever constraints are used by GitHub when adding email addresses to an account.

Group membership
^^^^^^^^^^^^^^^^

**Source**: Derived from GitHub organization and team memberships, with the exception of the user's default group.
That group will have the same name and GID as the user's username and UID, and is added automatically by the Science Platform.

This is not guaranteed to be safe, since the GitHub user ID and team ID space may overlap and user IDs may therefore conflict with team IDs.
However, in practice, given the small number of users we expect to use these deployments, it is probably safe enough.

**Storage**: Determined during authentication with GitHub API calls and stored as data associated with each token in Redis.

**Constraints**: There is no inherent limit in the number of groups a user may be a member of, but be aware that NFS only allows a user to be a member of 16 groups, one of which is the user's default group.
Group memberships above 16 may be ignored by the NFS server.

Group metadata
--------------

Group name
^^^^^^^^^^

(The below rules only apply to additional groups.
The user's default group has the same name as the username.)

**Source**: Each team that the user is a member of corresponds to one group.
The name of the group is the lowercase form of the organization, a dash (``-``), and the "slug" of the team as retrieved from the GitHub API.
If the resulting group name is longer than 32 characters, it is truncated at 25 characters and the first six characters of a hash of the full name will be appended.

**Storage**: Group names are stored where user group membership is stored.

**Constraints**: Group names must consist of lowercase ASCII letters and numbers, period (``.``), dash (``-``), and underscore (``_``), must begin with a letter or number, must contain a letter, and must be at most 32 characters long. [#]_

.. [#] Regular expression: ``^[a-zA-Z0-9][a-zA-Z0-9._-]*[a-zA-Z][a-zA-Z0-9._-]*$``

Numeric GID
^^^^^^^^^^^

**Source**: The team ID from GitHub.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Whatever constraints GitHub uses to assign team IDs.

.. _usdf:

USDF
====

User metadata
-------------

Username
^^^^^^^^

**Source**: The value of the ``name`` claim in the ID token returned by the OpenID Connect authentication protocol.

**Storage**: Stored as data associated with each token in Redis.

**Constraints**: Must consist solely of lowercase ASCII letters, numbers, and dash (``-``), must not start or end with a dash, and must not contain two consecutive dashes. [#]_
Must not consist entirely of numbers.

.. [#] Regular expression: ``^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$``

Numeric UID
^^^^^^^^^^^

**Source**: The ``uidNumber`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Primary GID
^^^^^^^^^^^

**Source**: The ``gidNumber`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Full name
^^^^^^^^^

**Source**: The ``gecos`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.
No assumptions are made about the structure of the name.

Email address
^^^^^^^^^^^^^

**Source**: The ``mail`` attribute of the user's record in LDAP.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.

Group membership
^^^^^^^^^^^^^^^^

**Source**: All groups in LDAP for which the user is listed as a member (via the ``memberUId`` attribute of the group), plus the group with a GID matching the primary GID of the user.
The user's primary group is not included in their group memberships, so instead it is looked up by GID and then added to the group memberships returned by LDAP.
Unlike the other deployments, the USDF deployment does not put the user in a default group with the same name as their username.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

.. warning::

   The scopes of an authentication token are calculated from the group membership at the time of initial user authentication and are not affected by subsequent changes to the user's group membership until that token expires.

**Constraints**: There is no inherent limit in the number of groups a user may be a member of, but be aware that NFS only allows a user to be a member of 16 groups, one of which is the user's default group.
Group memberships above 16 may be ignored by the NFS server.

Group metadata
--------------

Group name
^^^^^^^^^^

**Source**: The ``cn`` attribute of the LDAP record for the group.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Group names must consist of ASCII letters (upper- or lowercase) and numbers, period (``.``), dash (``-``), and underscore (``_``), must begin with a letter or number, must contain a letter, and must be at most 32 characters long. [#]_

.. [#] Regular expression: ``^[a-zA-Z0-9][a-zA-Z0-9._-]*[a-zA-Z][a-zA-Z0-9._-]*$``

Numeric GID
^^^^^^^^^^^

**Source**: The ``gidNumber`` attribute of the LDAP record for the group.

**Storage**: Retrieved from LDAP when needed and not stored locally in the Science Platform.

**Constraints**: Whatever constraints are used by the local identity management system that populates LDAP.
