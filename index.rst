:tocdepth: 1

.. sectnum::

Abstract
========

The Rubin Science Platform will store various metadata about each user, either created by the Science Platform (such as some identifiers) or collected from the relevant identity provider.
This document describes the metadata associated with users and its sources and constraints, such as numeric ranges for UIDs and GIDs and valid patterns for usernames and group names.

While this document is primarily about general users of the Science Platform deployment at the :abbr:`IDF (Interim Data Facility)` and :abbr:`CDF (Cloud Data Facility)`, it also records the differences for other Science Platform deployments (such as ones internal to the project) where known.

User metadata
=============

This is the user metadata that we are storing so far.
We expect to add additional metadata, such as whether a user has accepted the Acceptable Use Policy, before the production release.
This tech note will be updated when we add additional metadata.

Username
--------

**Source**: User chooses their (unique) username during enrollment.
For internal deployments using GitHub as the authentication source, the username is the same as their GitHub username converted to all lowercase.
For internal deployments using a local authentication provider, their username is whatever username is returned by the local authentication provider (usually via the ``sub`` claim in the OpenID Connect JWT).

**Storage**: The username is used as a unique key for the user in all identity management systems except for COmanage.
In COmanage, it is stored as an identifier associated with the user's record.

**Constraints**: Must consist solely of lowercase ASCII letters, numbers, and dash (``-``), must not start or end with a dash, and must not contain two consecutive dashes. [#]_
Must not consist entirely of numbers.

.. [#] Regular expression: ``^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*$``

Numeric UID
-----------

**Source**: Assigned by the Science Platform on first use of an account.
All IDF and CDF Science Platform deployments share the same UID assignment pool and map the same CILogon identity to the same UID.
Internal deployments using GitHub as the authentication source use the UID from GitHub.
Internal deployments using a local authentication provider may use either a configurable JWT claim from OpenID Connect or a configurable LDAP attribute from a local LDAP server.

**Storage**: For public deployments, stored within the Science Platform user database, which is shared between all IDF and CDF deployments.
For all deployments, the UID number is also stored in the token and retrieved via the token API.

**Constraints**: See :ref:`UID and GID assignment <uid-gid-assignment>`.
UIDs are unique and are intended to never be reused.
Once assigned, the UID for a given account never changes, even if the username is changed.

Full name
---------

**Source**: Taken from the user's federated identity provider during enrollment.
The user may choose to enter a new name.
Insofar as possible, the Rubin Science Platform will only record the user's entire name of choice as a single text field, not divided into components such as given name and family name.
COmanage currently does not properly support this, and may represent the name in components, but the Science Platform will attempt to use the combined form only.
For internal deployments using GitHub as the authentication source, the name is taken from the GitHub account metadata.
For internal deployments using a local authentication provider, the name is taken from the ``name`` claim in the issued OpenID Connect JWT.

**Storage**: The names from each associated federated identity are stored in COmanage, along with any name the user chooses to enter.
Whatever name they choose as primary is stored as the ``displayName`` attribute in the user's LDAP record as maintained by COmanage, and is retrieved from there by the Science Platform using the token API.
For deployments that do not use COmanage, the name is determined during authentication, stored with the user's authentication token, and retrieved as needed via the token API.

**Constraints**: Any valid UTF-8 string of reasonable length without control characters.
No assumptions are made about the structure of the name.

Email address
-------------

**Source**: Taken from the user's federated identity provider during enrollment.
The user may choose to enter a new email address.
For internal deployments using GitHub as the authentication source, the email address is taken from the GitHub account metadata.
For internal deployments using a local authentication provider, the name is taken from the ``email`` claim in the issued OpenID Connect JWT.

**Storage**: The email addresses from each associated federated identity are stored in COmanage, along with any email the user chooses to enter.
Whatever email address they choose as primary is stored as the ``mail`` attribute in the user's LDAP record as maintained by COmanage, and is retrieved from there by the Science Platform using the token API.
For deployments that do not use COmanage, the email address is determined during authentication, stored with the user's authentication token, and retrieved as needed via the token API.

**Constraints**: Must be a syntactically-valid `RFC 5322 addr-spec <https://datatracker.ietf.org/doc/html/rfc5322#section-3.4.1>`__.
COmanage will confirm the validity of the email address during enrollment by sending the user an email and having them follow a link in the email.

Group membership
----------------

**Source**: COmanage records the user's group membership (except in their default group).
Users are added to groups by group owners, and may be added to groups based on automated rules triggering off of their affiliation data.
For internal deployments using GitHub as the authentication source, the user's group membership is derived from their organization and team memberships.
For internal deployments using a local authentication mechanism, the user's group membership may be taken from an ``isMemberOf`` claim in the OpenID Connect JWT, or by querying a local LDAP server.

**Storage**: COmanage stores the user's group membership information and provides it in the LDAP server it maintains, as ``member`` attributes in a groups tree.
The group membership for a given user can be retrieved via the token API.
For internal deployments, either a local LDAP server is used, in which case group membership is handled much the same way as it is with COmanage, or it can be determined during authentication.
In the latter case, it is stored in the user's tokens and can be retrieved via the token API.

**Constraints**: There is no inherent limit in the number of groups a user may be a member of, but be aware that NFS only allows a user to be a member of 16 groups, one of which is the user's default group.
Group memberships above 16 may be ignored by the NFS server.

Group metadata
==============

Group name
----------

(The below rules only apply to additional groups.
The user's default group has the same name as the username.)

**Source**: Groups managed in COmanage are named when created manually.
Groups that come from a local authentication provider use whatever names the local authentication provider provides.
For internal deployments that use GitHub is used as the authentication provider, each team that the user is a member of corresponds to one group.
The name of the group is the lowercase form of the organization, a dash (``-``), and the "slug" of the team as retrieved from the GitHub API.
If the resulting group name is longer than 32 characters, it is truncated at 25 characters and the first six characters of a hash of the full name will be appended.

**Storage**: Group names are stored where user group membership is stored.

**Constraints**: For IDF and CDF deployments, all group names must begin with ``g_``.
Group names must consist of lowercase ASCII letters and numbers, period (``.``), dash (``-``), and underscore (``_``), must begin with a letter, and must be at most 32 characters long.
For internal deployments, uppercase letters are also allowed.

Numeric GID
-----------

**Source**: Assigned by the Science Platform on first use of a group.
All IDF and CDF Science Platform deployments share the same GID assignment pool and map the same CILogon group to the same GID.
Internal deployments using GitHub as the authentication source use the team ID from GitHub.
Internal deployments using a local authentication provider may use either the ``isMemberOf`` JWT claim from OpenID Connect or the ``gidNumber`` attribute from a local LDAP server.

**Storage**: For public deployments, stored within the Science Platform user database, which is shared between all IDF and CDF deployments.
For all deployments, the GID number is also stored in the token and retrieved via the token API.

**Constraints**: See :ref:`UID and GID assignment <uid-gid-assignment>`.
GIDs are unique and are intended to never be reused.
Once assigned, the GID for a given group never changes, even if the group name is changed.

.. _uid-gid-assignment:

UID and GID assignment
======================

The following section applies only to Science Platform deployments that use COmanage.

The Science Platform uses a POSIX file system for some storage.
Access control in that file system is done via numeric UIDs and GIDs.
Each user must therefore be assigned a unique UID, and each group must be assigned a unique GID.

Each user must also have a default group.
Following the now-standard Linux convention, that default group will have the same name as the user and will contain only the user.
That group must also have a unique GID.

For convenience, the GID of the user's default group will always match the user's UID.

The Science Platform requires support for 31-bit or 32-bit UIDs and GIDs and makes no attempt to support platforms with 16-bit UIDs or GIDs.
We can therefore take advantage of the increased UID and GID space up to 2,147,483,648.

UID and GID space is divided into the following ranges:

0-99
    Reserved for the container operating system.

100-999
    Reserved for users created by packages installed in containers, and for the use of some containers that use default UIDs in the high 900s.

1000-999999
    Reserved for future use.
    Note that 65534 is reserved by the operating system.

1000000-1999999
    GIDs for groups other than the user's default group.

2000000-2147483646
    User UIDs and the corresponding GID for the user's default group.

UIDs and GIDs are assigned on first use of a given user or group in any Science Platform deployment that shares the same UID and GID assignment database.
All IDF and CDF deployments will use the same UID and GID assignments so that UIDs and GIDs are portable between deployments.
We expect to sometimes want to mount the same POSIX file system on multiple deployments.

Once a given UID or GID has been used, it will never be reused for a different user or group.

COmanage does support assigning UIDs and GIDs, but the configuration complexity required is higher, and our assignment needs are a somewhat awkward fit for COmanage's capabilities.
We therefore will do UID and GID assignment independently of COmanage.
