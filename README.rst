PEP: 458
Title: Surviving a Compromise of PyPI
Author: Trishank Karthik Kuppusamy <tk47@students.poly.edu>,
        Donald Stufft <donald@stufft.io>,
        Justin Cappos <jcappos@poly.edu>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: DistUtils mailing list <distutils-sig@python.org>,
                TUF mailing list <theupdateframework@googlegroups.com>
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 27-Sep-2013


Abstract
========

This PEP describes how the Python Package Index (PyPI [1]_) may be integrated
with The Update Framework [2]_ (TUF).  TUF was designed to be a plug-and-play
security add-on to a software updater or package manager.  TUF provides
end-to-end security like SSL, but for software updates instead of HTTP
connections.  The framework integrates best security practices such as
separating responsibilities, adopting the many-man rule for signing packages,
keeping signing keys offline, and revocation of expired or compromised signing
keys.

The proposed integration will render modern package managers such as pip [3]_
more secure against various types of security attacks on PyPI and protect users
against them.  Even in the worst case where an attacker manages to compromise
PyPI itself, the damage is controlled in scope and limited in duration.

Specifically, this PEP will describe how PyPI processes should be adapted to
incorporate TUF metadata.  It will not prescribe how package managers such as
pip should be adapted to install or update with TUF metadata projects from
PyPI.


Rationale
=========

In January 2013, the Python Software Foundation (PSF) announced [4]_ that the
python.org wikis for Python, Jython, and the PSF were subjected to a security
breach which caused all of the wiki data to be destroyed on January 5 2013.
Fortunately, the PyPI infrastructure was not affected by this security breach.
However, the incident is a reminder that PyPI should take defensive steps to
protect users as much as possible in the event of a compromise.  Attacks on
software repositories happen all the time [5]_.  We must accept the possibility
of security breaches and prepare PyPI accordingly because it is a valuable
target used by thousands, if not millions, of people.

Before the wiki attack, PyPI used MD5 hashes to tell package managers such as
pip whether or not a package was corrupted in transit.  However, the absence of
SSL made it hard for package managers to verify transport integrity to PyPI.
It was easy to launch a man-in-the-middle attack between pip and PyPI to change
package contents arbitrarily.  This can be used to trick users into installing
malicious packages.  After the wiki attack, several steps were proposed (some
of which were implemented) to deliver a much higher level of security than was
previously the case: requiring SSL to communicate with PyPI [6]_, restricting
project names [7]_, and migrating from MD5 to SHA-2 hashes [8]_.

These steps, though necessary, are insufficient because attacks are still
possible through other avenues.  For example, a public mirror is trusted to
honestly mirror PyPI, but some mirrors may misbehave due to malice or accident.
Package managers such as pip are supposed to use signatures from PyPI to verify
packages downloaded from a public mirror [9]_, but none are known to actually
do so [10]_.  Therefore, it is also wise to add more security measures to
detect attacks from public mirrors or content delivery networks [11]_ (CDNs).

Even though official mirrors are being deprecated on PyPI [12]_, there remain a
wide variety of other attack vectors on package managers [13]_.  Among other
things, these attacks can crash client systems, cause obsolete packages to be
installed, or even allow an attacker to execute arbitrary code.  In September
2013, we showed how the latest version of pip then was susceptible to these
attacks and how TUF could protect users against them [14]_.

Finally, PyPI allows for packages to be signed with GPG keys [15]_, although no
package manager is known to verify those signatures, thus negating much of the
benefits of having those signatures at all.  Validating integrity through
cryptography is important, but issues such as immediate and secure key
revocation or specifying a required threshold number of signatures still
remain.  Furthermore, GPG by itself does not immediately address the attacks
mentioned above.

In order to protect PyPI against infrastructure compromises, we propose
integrating PyPI with The Update Framework [2]_ (TUF).


Definitions
===========

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC 2119__.

__ http://www.ietf.org/rfc/rfc2119.txt

In order to keep this PEP focused solely on the application of TUF on PyPI, the
reader is assumed to already be familiar with the design principles of
TUF [2]_.  It is also strongly RECOMMENDED that the reader be familiar with the
TUF specification [16]_.

* Projects: Projects are software components that are made available for
  integration.  Projects include Python libraries, frameworks, scripts, plugins,
  applications, collections of data or other resources, and various
  combinations thereof.  Public Python projects are typically registered on the
  Python Package Index [17]_.

* Releases: Releases are uniquely identified snapshots of a project [17]_.

* Distributions: Distributions are the packaged files which are used to publish
  and distribute a release [17]_.

* Simple index: The HTML page which contains internal links to the
  distributions of a project [17]_.

* Consistent snapshot: A set of TUF metadata and PyPI targets that capture the
  complete state of all projects on PyPI as they were at some fixed point in
  time.

* The *consistent-snapshot* (*release*) role: In order to prevent confusion due
  to the different meanings of the term "release" as employed by PEP 426 [17]_
  and the TUF specification [16]_, we rename the *release* role as the
  *consistent-snapshot* role.

* Continuous delivery: A set of processes with which PyPI produces consistent
  snapshots that can safely coexist and deleted independently [18]_.

* Developer: Either the owner or maintainer of a project who is allowed to
  update the TUF metadata as well as distribution metadata and data for the
  project.

* Online key: A key that MUST be stored on the PyPI server infrastructure.
  This is usually to allow automated signing with the key.  However, this means
  that an attacker who compromises PyPI infrastructure will be able to read
  these keys.

* Offline key: A key that MUST be stored off the PyPI infrastructure.  This
  prevents automated signing with the key.  This means that an attacker who
  compromises PyPI infrastructure will not be able to immediately read these
  keys.

* Developer key: A private key for which its corresponding public key is
  registered with PyPI to say that it is responsible for directly signing for
  or delegating the distributions belonging to a project.  For the purposes of
  this PEP, it is offline in the sense that the private key MUST not be stored
  on PyPI.  However, the project is free to require certain developer keys to
  be online on its own infrastructure.

* Threshold signature scheme: A role could increase its resilience to key
  compromises by requiring that at least t out of n keys are REQUIRED to sign
  its metadata.  This means that a compromise of t-1 keys is insufficient to
  compromise the role itself.  We denote this property by saying that the role
  requires (t, n) keys.


Overview
========

.. image:: https://raw.github.com/theupdateframework/pep-on-pypi-with-tuf/master/figure1.png

Figure 1: A simplified overview of the roles in PyPI with TUF

Figure 1 shows a simplified overview of the roles that TUF metadata assume on
PyPI.  The top-level *root* role signs for the keys of the top-level
*timestamp*, *consistent-snapshot*, *targets* and *root* roles.  The
*timestamp* role signs for a new and consistent snapshot.  The *consistent-
snapshot* role signs for the *root*, *targets* and all delegated targets
metadata.  The *claimed* role signs for all projects that have registered their
own developer keys with PyPI.  The *recently-claimed* role signs for all
projects that recently registered their own developer keys with PyPI.  Finally,
the *unclaimed* role signs for all projects that have not registered developer
keys with PyPI.  The *claimed*, *recently-claimed* and *unclaimed* roles are
numbered 1, 2, 3 respectively because a project will be searched for in each of
those roles in that descending order: first in *claimed*, then in
*recently-claimed* if necessary, and finally in *unclaimed* if necessary.

Every year, PyPI administrators are going to sign for *root* role keys.  After
that, automation will continuously sign for a timestamped, consistent snapshot
of all projects.  Every few months, PyPI administrators will move projects with
vetted developer keys from the *recently-claimed* role to the *claimed* role.
As we will soon see, they will sign for *claimed* with projects with offline
keys.

This PEP does not require project developers to use TUF to secure their
packages from attacks on PyPI.  By default, all projects will be signed for by
the *unclaimed* role.  If a project wishes stronger security guarantees, then
the project is strongly RECOMMENDED to register developer keys with PyPI so
that it may sign for its own distributions.  By doing so, the project must
remain as a *recently-claimed* project until PyPI administrators have had an
opportunity to vet the developer keys of the project, after which the project
will be moved to the *claimed* role.

This PEP has **not** been designed to be backward-compatible for package
managers that do not use the TUF security protocol to install or update a
project from the PyPI described here.  Instead, it is RECOMMENDED that PyPI
maintain a backward-compatible API of itself that does NOT offer TUF so that
older package managers that do not use TUF will be able to install or update
projects from PyPI as usual but without any of the security offered by TUF.
For the rest of this PEP, we will assume that PyPI will simultaneously maintain
a backward-incompatible API of itself for package managers that MUST use TUF to
securely install or update projects.  We think that this approach represents a
reasonable trade-off: older package managers that do not TUF will still be able
to install or update projects without any TUF security from PyPI, and newer
package managers that do use TUF will be able to securely install or update
projects.  At some point in the future, PyPI administrators MAY choose to
permanently deprecate the backward-compatible version of itself that does not
offer TUF metadata.

Unless a mirror, CDN or the PyPI repository has been compromised, the end-user
will not be able to discern whether or not a package manager is using TUF to
install or update a project from PyPI.


Responsibility Separation
=========================

Recall that TUF requires four top-level roles: *root*, *timestamp*,
*consistent-snapshot* and *targets*.  The *root* role specifies the keys of all
the top-level roles (including itself).  The *timestamp* role specifies the
latest consistent snapshot.  The *consistent-snapshot* role specifies the
latest versions of all TUF metadata files (other than *timestamp*).  The
*targets* role specifies available target files (in our case, it will be all
files on PyPI under the /simple and /packages directories).  In this PEP, each
of these roles will serve their responsibilities without exception.

Our proposal offers two levels of security to developers.  If developers opt in
to secure their projects with their own developer keys, then their projects
will be very secure.  Otherwise, TUF will still protect them in many cases:

1. Minimum security (no action by a developer): protects *unclaimed* and
   *recently-claimed* projects without developer keys from CDNs [19]_ or public
   mirrors, but not from some PyPI compromises.  This is because continuous
   delivery requires some keys to be online.  This level of security protects
   projects from being accidentally or deliberately tampered with by a mirror
   or a CDN because the mirror or CDN will not have any of the PyPI or
   developer keys required to sign for projects. However, it would not protect
   projects from attackers who have compromised PyPI because they will be able
   to manipulate the TUF metadata for *unclaimed* projects with the appropriate
   online keys.

2. Maximum security (developer signs their project): protects projects with
   developer keys not only from CDNs or public mirrors, but also from some PyPI
   compromises.  This is because many important keys will be offline.  This
   level of security protects projects from being accidentally or deliberately
   tampered with by a mirror or a CDN for reasons identical to the minimum
   security level.  It will also protect projects (or at least mitigate
   damages) from the most likely attacks on PyPI.  For example: given access to
   online keys after a PyPI compromise, attackers will be able to freeze the
   distributions for these projects, but they will not be able to serve
   malicious distributions for these projects (not without compromising other
   offline keys which would entail more risk, time and energy).  Details for
   the exact level of security offered is discussed in the section on key
   management.

In order to complete support for continuous delivery, we propose three
delegated targets roles:

1. *claimed*: Signs for the delegation of PyPI projects to their respective
   developer keys.

2. *recently-claimed*: This role is almost identical to the *claimed* role and
   could technically be performed by the *unclaimed* role, but there are two
   important reasons why it exists independently: the first reason is to
   improve the performance of looking up projects in the *unclaimed* role (by
   moving metadata to the *recently-claimed* role instead), and the second
   reason is to make it easier for PyPI administrators to move
   *recently-claimed* projects to the *claimed* role.

3. *unclaimed*: Signs for PyPI projects without developer keys.

The *targets* role MUST delegate all PyPI projects to the three delegated
targets roles in the order of appearance listed above.  This means that when
pip downloads with TUF a distribution from a project on PyPI, it will first
consult the *claimed* role about it.  If the *claimed* role has delegated the
project, then pip will trust the project developers (in order of delegation)
about the TUF metadata for the project.  Otherwise, pip will consult the
*recently-claimed* role about the project.  If the *recently-claimed* role has
delegated the project, then pip will trust the project developers (in order of
delegation) about the TUF metadata for the project.  Otherwise, pip will
consult the *unclaimed* role about the TUF metadata for the project.  If the
*unclaimed* role has not delegated the project, then the project is considered
to be non-existent on PyPI.

A PyPI project MAY begin without registering a developer key.  Therefore, the
project will be signed for by the *unclaimed* role.  After registering
developer keys, the project will be removed from the *unclaimed* role and
delegated to the *recently-claimed* role.  After a probation period and a
vetting process to verify the developer keys of the project, the project will
be removed from the *recently-claimed* role and delegated to the *claimed*
role.

The *claimed* role offers maximum security, whereas the *recently-claimed* and
*unclaimed* role offer minimum security.  All three roles support continuous
delivery of PyPI projects.

The *unclaimed* role offers minimum security because PyPI will sign for
projects without developer keys with an online key in order to permit
continuous delivery.

The *recently-claimed* role offers minimum security because while the project
developers will sign for their own distributions with offline developer keys,
PyPI will sign with an online key the delegation of the project to those
offline developer keys.  The signing of the delegation with an online key
allows PyPI administrators to continuously deliver projects without having to
continuously sign the delegation whenever one of those projects registers
developer keys.

Finally, the *claimed* role offers maximum security because PyPI will sign with
offline keys the delegation of a project to its offline developer keys.  This
means that every now and then, PyPI administrators will vet developer keys and
sign the delegation of a project to those developer keys after being reasonably
sure about the ownership of the developer keys.  The process for vetting
developer keys is out of the scope of this PEP.


Metadata Management
===================

In this section, we examine the TUF metadata that PyPI must manage by itself,
and other TUF metadata that must be safely delegated to projects.  Examples of
the metadata described here may be seen at our testbed mirror of
`PyPI-with-TUF`__.

__ http://mirror1.poly.edu/

The metadata files that change most frequently will be *timestamp*,
*consistent-snapshot* and delegated targets  (*claimed*, *recently-claimed*,
*unclaimed*, project) metadata.  The *timestamp* and *consistent-snapshot*
metadata MUST be updated whenever *root*, *targets* or delegated targets
metadata are updated.  Observe, though, that *root* and *targets* metadata are
much less likely to be updated as often as delegated targets metadata.
Therefore, *timestamp* and *consistent-snapshot* metadata will most likely be
updated frequently (possibly every minute) due to delegated targets metadata
being updated frequently in order to drive continuous delivery of projects.

Consequently, the processes with which PyPI updates projects will have to be
updated accordingly, the details of which are explained in the following
subsections.


Why Do We Need Consistent Snapshots?
------------------------------------

In an ideal world, metadata and data should be immediately updated and
presented whenever a project is updated.  In practice, there will be problems
when there are many readers and writers who access the same metadata or data at
the same time.

An important example at the time of writing is that, mirrors are very likely,
as far as we can tell, to update in an inconsistent manner from PyPI as it is
without TUF.  Specifically, a mirror would update itself in such a way that
project A would be from time T, whereas project B would be from time T+5,
project C would be from time T+3, and so on where T is the time that the mirror
first begun updating itself.  There is no known way for a mirror to update
itself such that it captures the state of all projects as they were at time T.

Adding TUF to PyPI will not automatically solve the problem.  Consider what we
call the `"inverse replay" or "fast-forward" problem`__.  Suppose that PyPI has
timestamped a consistent snapshot at version 1.  A mirror is later in the
middle of copying PyPI at this snapshot.  While the mirror is copying PyPI at
this snapshot, PyPI timestamps a new snapshot at, say, version 2.  Without
accounting for consistency, the mirror would then find itself with a copy of
PyPI in an inconsistent state which is indistinguishable from arbitrary
metadata or target attacks.  The problem would also apply when the mirror is
substituted with a pip user.

__ https://groups.google.com/forum/#!topic/theupdateframework/8mkR9iqivQA

Therefore, the problem can be summarized as such: there are problems of
consistency on PyPI with or without TUF.  TUF requires its metadata to be
consistent with the data, but how would the metadata be kept consistent with
projects that change all the time?

As a result, we will solve for PyPI the problem of producing a consistent
snapshot that captures the state of all known projects at a given time.  Each
consistent snapshot can safely coexist with any other consistent snapshot and
deleted independently without affecting any other consistent snapshot.

The gist of the solution is that every metadata or data file written to disk
MUST include in its filename the `cryptographic hash`__ of the file.  How would
this help clients which use the TUF protocol to securely and consistently
install or update a project from PyPI?

__ https://en.wikipedia.org/wiki/Cryptographic_hash_function

Recall that the first step in the TUF protocol requires the client to download
the latest *timestamp* metadata.  However, the client would not know in advance
the hash of the *timestamp* metadata file from the latest consistent snapshot.
Therefore, PyPI MUST redirect all HTTP GET requests for *timestamp* metadata to
the *timestamp* metadata file from the latest consistent snapshot.  Since the
*timestamp* metadata is the root of a tree of cryptographic hashes pointing to
every other metadata or target file that are meant to exist together for
consistency, the client is then able to retrieve any file from this consistent
snapshot by deterministically including, in the request for the file, the hash
of the file in the filename.  Assuming infinite disk space and no `hash
collisions`__, a client may safely read from one consistent snapshot while PyPI
produces another consistent snapshot.

__ https://en.wikipedia.org/wiki/Collision_(computer_science)

In this simple but effective manner, we are able to capture a consistent
snapshot of all projects and the associated metadata at a given time.  The next
subsection will explicate the implementation details of this idea.


Producing Consistent Snapshots
------------------------------

Given a project, PyPI is responsible for updating, depending on the project,
either the *claimed*, *recently-claimed* or *unclaimed* metadata as well as
associated delegated targets metadata.  Every project MUST upload its set of
metadata and targets in a single transaction.  We will call this set of files
the project transaction.  We will discuss later how PyPI MAY validate the files
in a project transaction.  For now, let us focus on how PyPI will respond to a
project transaction.  We will call this response the project transaction
process.  There will also be a consistent snapshot process that we will define
momentarily; for now, it suffices to know that project transaction processes
and the consistent snapshot process must coordinate with each other.

Also, every metadata and target file MUST include in its filename the `hex
digest`__ of its `SHA-256`__ hash.  For this PEP, it is RECOMMENDED that PyPI
adopt a simple convention of the form filename.digest.ext, where filename is
the original filename without a copy of the hash, digest is the hex digest of
the hash, and ext is the filename extension.

__ http://docs.python.org/2/library/hashlib.html#hashlib.hash.hexdigest
__ https://en.wikipedia.org/wiki/SHA-2

When an *unclaimed* project uploads a new transaction, a project transaction
process MUST add  all new targets and relevant delegated *unclaimed* metadata.
(We will see later in this section why the *unclaimed* role will delegate
targets to a number of delegated *unclaimed* roles.)  Finally, the project
transaction process MUST inform the consistent snapshot process about new
delegated *unclaimed* metadata.

When a *recently-claimed* project uploads a new a transaction, a project
transaction process MUST add all new targets and delegated targets metadata for
the project.  If the project is new, then the project transaction process MUST
also add new *recently-claimed* metadata with public keys and threshold number
(which MUST be part of the transaction) for the project.  Finally, the project
transaction process MUST inform the consistent snapshot process about new
*recently-claimed* metadata as well as the current set of delegated targets
metadata for the project.

The process for a *claimed* project is slightly different.  The difference is
that PyPI administrators will choose to move the project from the
*recently-claimed* role to the *claimed* role.  A project transaction process
MUST then add new *recently-claimed* and *claimed* metadata to reflect this
migration.  As is the case for a *recently-claimed* project, the project
transaction process MUST always add all new targets and delegated targets
metadata for the *claimed* project.  Finally, the project transaction process
MUST inform the consistent snapshot process about new *recently-claimed* or
*claimed* metadata as well as the current set of delegated targets metadata for
the project.

Project transaction processes SHOULD be automated, except when PyPI
administrators move a project from the *recently-claimed* role to the *claimed*
role.  Project transaction processes MUST also be applied atomically: either
all metadata and targets, or none of them, are added.  The project transaction
processes and consistent snapshot process SHOULD work concurrently.  Finally,
project transaction processes SHOULD keep in memory the latest *claimed*,
*recently-claimed* and *unclaimed* metadata so that they will be correctly
updated in new consistent snapshots.

All project transactions MAY be placed in a single queue and processed
serially.  Alternatively, the queue MAY be processed concurrently in order of
appearance provided that the following rules are observed:

1. No pair of project transaction processes must concurrently work on the same
   project.

2. No pair of project transaction processes must concurrently work on
   *unclaimed* projects that belong to the same delegated *unclaimed* targets
   role.

3. No pair of project transaction processes must concurrently work on new
   *recently-claimed* projects.

4. No pair of project transaction processes must concurrently work on new
   *claimed* projects.

5. No project transaction process must work on a new *claimed* project while
   another project transaction process is working on a new *recently-claimed*
   project and vice versa.

These rules MUST be observed so that metadata is not read from or written to
inconsistently.

The consistent snapshot process is fairly simple and SHOULD be automated.  The
consistent snapshot process MUST keep in memory the latest working set of
*root*, *targets* and delegated targets metadata.  Every minute or so, the
consistent snapshot process will sign for this latest working set.  (Recall
that project transaction processes continuously inform the consistent snapshot
process about the latest delegated targets metadata in a concurrency-safe
manner.  The consistent snapshot process will actually sign for a copy of the
latest working set while the actual latest working set in memory will be
updated with information continuously communicated by project transaction
processes.)  Next, the consistent snapshot process MUST generate and sign new
*timestamp* metadata that will vouch for the *consistent-snapshot* metadata
generated in the previous step.  Finally, the consistent snapshot process MUST
add new *timestamp* and *consistent-snapshot* metadata representing the latest
consistent snapshot.

A few implementation notes are now in order.  So far, we have seen only that
new metadata and targets are added, but not that old metadata and targets are
removed.  Practical constraints are such that eventually PyPI will run out of
disk space to produce a new consistent snapshot.  In that case, PyPI MAY then
use something like a "mark-and-sweep" algorithm to delete sufficiently old
consistent snapshots: in order to preserve the latest consistent snapshot, PyPI
would walk objects beginning from the root (*timestamp*) of the latest
consistent snapshot, mark all visited objects, and delete all unmarked
objects.  The last few consistent snapshots may be preserved in a similar
fashion.  Deleting a consistent snapshot will cause clients to see nothing
thereafter but HTTP 404 responses to any request for a file in that consistent
snapshot.  Clients SHOULD then retry their requests with the latest consistent
snapshot.

We do **not** consider updates to any consistent snapshot because `hash
collisions`__ are out of the scope of this PEP.  In case a hash collision is
observed, PyPI MAY wish to check that the file being added is identical to the
file already stored.  (Should a hash collision be observed, it is far more
likely the case that the file is identical rather than being a genuine
`collision attack`__.)  Otherwise, PyPI MAY either overwrite the existing file
or ignore any write operation to an existing file.

__ https://en.wikipedia.org/wiki/Collision_(computer_science)
__ https://en.wikipedia.org/wiki/Collision_attack

All clients, such as pip using the TUF protocol, MUST be modified to download
every metadata and target file (except for *timestamp* metadata) by including,
in the request for the file, the hash of the file in the filename.  Following
the filename convention recommended earlier, a request for the file at
filename.ext will be transformed to the equivalent request for the file at
filename.digest.ext.

Finally, PyPI SHOULD use a `transaction log`__ to record project transaction
processes and queues so that it will be easier to recover from errors after a
server failure.

__ https://en.wikipedia.org/wiki/Transaction_log


Metadata Validation
-------------------

A *claimed* or *recently-claimed* project will need to upload in its
transaction to PyPI not just targets (a simple index as well as distributions)
but also TUF metadata.  The project MAY do so by uploading a ZIP file
containing two directories, /metadata/ (containing delegated targets metadata
files) and /targets/ (containing targets such as the project simple index and
distributions which are signed for by the delegated targets metadata).

Whenever the project uploads metadata or targets to PyPI, PyPI SHOULD check the
project TUF metadata for at least the following properties:

* A threshold number of the developers keys registered with PyPI by that
  project MUST have signed for the delegated targets metadata file that
  represents the "root" of targets for that project (e.g. metadata/targets/
  project.txt).

* The signatures of delegated targets metadata files MUST be valid.

* The delegated targets metadata files MUST NOT be expired.

* The delegated targets metadata MUST be consistent with the targets.

* A delegator MUST NOT delegate targets that were not delegated to itself by
  another delegator.

* A delegatee MUST NOT sign for targets that were not delegated to itself by a
  delegator.

* Every file MUST contain a unique copy of its hash in its filename following
  the filename.digest.ext convention recommended earlier.

If PyPI chooses to check the project TUF metadata, then PyPI MAY choose to
reject publishing any set of metadata or targets that do not meet these
requirements.

PyPI MUST enforce access control by ensuring that each project can only write
to the TUF metadata for which it is responsible.  It MUST do so by ensuring
that project transaction processes write to the correct metadata as well as
correct locations within those metadata.  For example, a project transaction
process for an *unclaimed* project MUST write to the correct target paths in
the correct delegated *unclaimed* metadata for the targets of the project.

On rare occasions, PyPI MAY wish to extend the TUF metadata format for projects
in a backward-incompatible manner.  Note that PyPI will NOT be able to
automatically rewrite existing TUF metadata on behalf of projects in order to
upgrade the metadata to the new backward-incompatible format because this would
invalidate the signatures of the metadata as signed by developer keys.
Instead, package managers SHOULD be written to recognize and handle multiple
incompatible versions of TUF metadata so that *claimed* and *recently-claimed*
projects could be offered a reasonable time to migrate their metadata to newer
but backward-incompatible formats.

The details of how each project manages its TUF metadata is beyond the scope of
this PEP.


Mirroring Protocol
------------------

The mirroring protocol as described in PEP 381 [9]_ SHOULD change to mirror
PyPI with TUF.

A mirror SHOULD have to maintain for its clients only one consistent snapshot
which would represent the latest consistent snapshot from PyPI known to the
mirror.  The mirror would then serve all HTTP requests for metadata or targets
by simply reading directly from this consistent snapshot directory.

The mirroring protocol itself is fairly simple.  The mirror would ask PyPI for
*timestamp* metadata from the latest consistent snapshot and proceed to copy
the entire consistent snapshot from the *timestamp* metadata onwards.  If the
mirror encounters a failure to copy any metadata or target file while copying
the consistent snapshot, it SHOULD retrying resuming the copy of that
particular consistent snapshot.  If PyPI has deleted that consistent snapshot,
then the mirror SHOULD delete the failed consistent snapshot and try
downloading the latest consistent snapshot instead.

The mirror SHOULD point users to a previous consistent snapshot directory while
it is copying the latest consistent snapshot from PyPI.  Only after the latest
consistent snapshot has been completely copied SHOULD the mirror switch clients
to the latest consistent snapshot.  The mirror MAY then delete the previous
consistent snapshot once it finds that no client is reading from the previous
consistent snapshot.

The mirror MAY use extant file transfer software such as rsync__ to mirror
PyPI. In that case, the mirror MUST first obtain the latest known timestamp
metadata from PyPI. The mirror MUST NOT immediately publish the latest known
timestamp metadata from PyPI. Instead, the mirror MUST first iteratively
transfer all new files from PyPI until there are no new files left to transfer.
Finally, the mirror MUST publish the latest known timestamp it fetched from
PyPI so that package managers such as pip may be directed to the latest
consistent snapshot known to the mirror.

__ https://rsync.samba.org/


Backup Process
--------------

In order to be able to safely restore from static snapshots later in the event
of a compromise, PyPI SHOULD maintain a small number of its own mirrors to copy
PyPI consistent snapshots according to some schedule.  The mirroring protocol
can be used immediately for this purpose.  The mirrors must be secured and
isolated such that they are responsible only for mirroring PyPI.  The mirrors
can be checked against one another to detect accidental or malicious failures.


Metadata Expiry Times
---------------------

The *root* and *targets* role metadata SHOULD expire in a year, because these
metadata files are expected to change very rarely.

The *claimed* role metadata SHOULD expire in three to six months, because this
metadata is expected to be refreshed in that time frame.  This time frame was
chosen to induce an easier administration process for PyPI.

The *timestamp*, *consistent-snapshot*, *recently-claimed* and *unclaimed* role
metadata SHOULD expire in a day because a CDN or mirror SHOULD synchronize
itself with PyPI every day.  Furthermore, this generous time frame also takes
into account client clocks that are highly skewed or adrift.

The expiry times for the delegated targets metadata of a project is beyond the
scope of this PEP.


Metadata Scalability
--------------------

Due to the growing number of projects and distributions, the TUF metadata will
also grow correspondingly.

For example, consider the *unclaimed* role.  In August 2013, we found that the
size of the *unclaimed* role metadata was about 42MB if the *unclaimed* role
itself signed for about 220K PyPI targets (which are simple indices and
distributions).  We will not delve into details in this PEP, but TUF features a
so-called "`lazy bin walk`__" scheme which splits a large targets or delegated
targets metadata file into many small ones.  This allows a TUF client updater
to intelligently download only a small number of TUF metadata files in order to
update any project signed for by the *unclaimed* role.  For example, applying
this scheme to the previous repository resulted in pip downloading between
1.3KB and 111KB to install or upgrade a PyPI project via TUF.

__ https://github.com/theupdateframework/tuf/issues/39

From our findings as of the time of writing, PyPI SHOULD split all targets in
the *unclaimed* role by delegating it to 1024 delegated targets role, each of
which would sign for PyPI targets whose hashes fall into that "bin" or
delegated targets role.  We found that 1024 bins would result in the
*unclaimed* role metadata and each of its binned delegated targets role
metadata to be about the same size (40-50KB) for about 220K PyPI targets
(simple indices and distributions).

It is possible to make the TUF metadata more compact by representing it in a
binary format as opposed to the JSON text format.  Nevertheless, we believe
that a sufficiently large number of project and distributions will induce
scalability challenges at some point, and therefore the *unclaimed* role will
then still need delegations in order to address the problem.  Furthermore, the
JSON format is an open and well-known standard for data interchange.

Due to the large number of delegated target metadata files, compressed versions
of *consistent-snapshot* metadata SHOULD also be made available.


Key Management
==============

In this section, we examine the kind of keys required to sign for TUF roles on
PyPI.  TUF is agnostic with respect to choices of digital signature algorithms.
For the purpose of discussion, we will assume that most digital signatures will
be produced with the well-tested and tried RSA algorithm [20]_.  Nevertheless,
we do NOT recommend any particular digital signature algorithm in this PEP
because there are a few important constraints: firstly, cryptography changes
over time; secondly, package managers such as pip may wish to perform signature
verification in Python, without resorting to a compiled C library, in order to
be able to run on as many systems as Python supports; finally, TUF recommends
diversity of keys for certain applications, and we will soon discuss these
exceptions.


Number Of Keys
--------------

The *timestamp*, *consistent-snapshot*, *recently-claimed* and *unclaimed*
roles will need to support continuous delivery.  Even though their respective
keys will then need to be online, we will require that the keys be independent
of each other.  This allows for each of the keys to be placed on separate
servers if need be, and prevents side channel attacks that compromise one key
from automatically compromising the rest of the keys.  Therefore, each of the
*timestamp*, *consistent-snapshot*, *recently-claimed* and *unclaimed* roles
MUST require (1, 1) keys.

The *unclaimed* role MAY delegate targets in an automated manner to a number of
roles called "bins", as we discussed in the previous section.  Each of the
"bin" roles SHOULD share the same key as the *unclaimed* role, due
simultaneously to space efficiency of metadata and because there is no security
advantage in requiring separate keys.

The *root* role is critical for security and should very rarely be used.  It is
primarily used for key revocation, and it is the root of trust for all of PyPI.
The *root* role signs for the keys that are authorized for each of the
top-level roles (including itself).  The keys belonging to the *root* role are
intended to be very well-protected and used with the least frequency of all
keys.  We propose that every PSF board member own a (strong) root key.  A
majority of them can then constitute the quorum to revoke or endow trust in all
top-level keys.  Alternatively, the system administrators of PyPI (instead of
PSF board members) could be responsible for signing for the *root* role.
Therefore, the *root* role SHOULD require (t, n) keys, where n is the number of
either all PyPI administrators or all PSF board members, and t > 1 (so that at
least two members must sign the *root* role).

The *targets* role will be used only to sign for the static delegation of all
targets to the *claimed*, *recently-claimed* and *unclaimed* roles.  Since
these target delegations must be secured against attacks in the event of a
compromise, the keys for the *targets* role MUST be offline and independent
from other keys.  For simplicity of key management without sacrificing
security, it is RECOMMENDED that the keys of the *targets* role are permanently
discarded as soon as they have been created and used to sign for the role.
Therefore, the *targets* role SHOULD require (1, 1) keys.  Again, this is
because the keys are going to be permanently discarded, and more offline keys
will not help against key recovery attacks [21]_ unless diversity of keys is
maintained.

Similarly, the *claimed* role will be used only to sign for the dynamic
delegation of projects to their respective developer keys.  Since these target
delegations must be secured against attacks in the event of a compromise, the
keys for the *claimed* role MUST be offline and independent from other keys.
Therefore, the *claimed* role SHOULD require (t, n) keys, where n is the number
of all PyPI administrators (in order to keep it manageable), and t ≥ 1 (so that
at least one member MUST sign the *claimed* role).  While a stronger threshold
would indeed render the role more robust against a compromise of the *claimed*
keys (which is highly unlikely assuming that the keys are independent and
securely kept offline), we think that this trade-off is acceptable for the
important purpose of keeping the maintenance overhead for PyPI administrators
as little as possible.  At the time of writing, we are keeping this point open
for discussion by the distutils-sig community.

The number of developer keys is project-specific and thus beyond the scope of
this PEP.


Online and Offline Keys
-----------------------

In order to support continuous delivery, the *timestamp*,
*consistent-snapshot*, *recently-claimed* and *unclaimed* role keys MUST be
online.

As explained in the previous section, the *root*, *targets* and *claimed* role
keys MUST be offline for maximum security.  Developers keys will be offline in
the sense that the private keys MUST NOT be stored on PyPI, though some of them
may be online on the private infrastructure of the project.


Key Strength
------------

At the time of writing, we recommend that all RSA keys (both offline and
online) SHOULD have a minimum key size of 3072 bits for data-protection
lifetimes beyond 2030 [22]_.


Diversity Of Keys
-----------------

Due to the threats of weak key generation and implementation weaknesses [2]_,
the types of keys as well as the libraries used to generate them should vary
within TUF on PyPI.  Our current implementation of TUF supports multiple
digital signature algorithms such as RSA (with OpenSSL [23]_ or PyCrypto [24]_)
and ed25519 [25]_.  Furthermore, TUF supports the binding of other
cryptographic libraries that it does not immediately support "out of the box",
and so one MAY generate keys using other cryptographic libraries and use them
for TUF on PyPI.

As such, the root role keys SHOULD be generated by a variety of digital
signature algorithms as implemented by different cryptographic libraries.


Key Compromise Analysis
-----------------------

.. image:: https://raw.github.com/theupdateframework/pep-on-pypi-with-tuf/master/table1.png

Table 1: Attacks possible by compromising certain combinations of role keys


Table 1 summarizes the kinds of attacks rendered possible by compromising a
threshold number of keys belonging to the TUF roles on PyPI.  Except for the
*timestamp* and *consistent-snapshot* roles, the pairwise interaction of role
compromises may be found by taking the union of both rows.

In September 2013, we showed how the latest version of pip then was susceptible
to these attacks and how TUF could protect users against them [14]_.

An attacker who compromises developer keys for a project and who is able to
somehow upload malicious metadata and targets to PyPI will be able to serve
malicious updates to users of that project (and that project alone).  Note that
compromising *targets* or any delegated targets role (except for project
targets metadata) does not immediately endow the attacker with the ability to
serve malicious updates.  The attacker must also compromise the *timestamp* and
*consistent-snapshot* roles (which are both online and therefore more likely to
be compromised).  This means that in order to launch any attack, one must be
not only be able to act as a man-in-the-middle but also compromise the
*timestamp* key (or the *root* keys and sign a new *timestamp* key).  To launch
any attack other than a freeze attack, one must also compromise the
*consistent-snapshot* key.

Finally, a compromise of the PyPI infrastructure MAY introduce malicious
updates to *recently-claimed* and *unclaimed* projects because the keys for
those roles are online.  However, attackers cannot modify *claimed* projects in
such an event because *targets* and *claimed* metadata have been signed with
offline keys.  Therefore, it is RECOMMENDED that high-value projects register
their developer keys with PyPI and sign for their own distributions.


In the Event of a Key Compromise
--------------------------------

By a key compromise, we mean that the key as well as PyPI infrastructure has
been compromised and used to sign new metadata on PyPI.

If a threshold number of developer keys of a project have been compromised,
then the project MUST take the following steps:

1. The project metadata and targets MUST be restored to the last known good
   consistent snapshot where the project was not known to be compromised.  This
   can be done by the developers repackaging and resigning all targets with the
   new keys.

2. The project delegated targets metadata MUST have their version numbers
   incremented, expiry times suitably extended and signatures renewed.

Whereas PyPI MUST take the following steps:

1. Revoke the compromised developer keys from the delegation to the project by
   the *recently-claimed* or *claimed* role. This is done by replacing the
   compromised developer keys with newly issued developer keys.

2. A new timestamped consistent snapshot MUST be issued.

If a threshold number of *timestamp*, *consistent-snapshot*, *recently-claimed*
or *unclaimed* keys have been compromised, then PyPI MUST take the following
steps:

1. Revoke the *timestamp*, *consistent-snapshot* and *targets* role keys from
   the *root* role.  This is done by replacing the compromised *timestamp*,
   *consistent-snapshot* and *targets* keys with newly issued keys.

2. Revoke the *recently-claimed* and *unclaimed* keys from the *targets* role
   by replacing their keys with newly issued keys.  Sign the new *targets* role
   metadata and discard the new keys (because, as we explained earlier, this
   increases the security of *targets* metadata).

3. Clear all targets or delegations in the *recently-claimed* role and delete
   all associated delegated targets metadata.  Recently registered projects
   SHOULD register their developer keys again with PyPI.

4. All targets of the *recently-claimed* and *unclaimed* roles SHOULD be
   compared with the last known good consistent snapshot where none of the
   *timestamp*, *consistent-snapshot*, *recently-claimed* or *unclaimed* keys
   were known to have been compromised.  Added, updated or deleted targets in
   the compromised consistent snapshot that do not match the last known good
   consistent snapshot MAY be restored to their previous versions.  After
   ensuring the integrity of all *unclaimed* targets, the *unclaimed* metadata
   MUST be regenerated.

5. The *recently-claimed* and *unclaimed* metadata MUST have their version
   numbers incremented, expiry times suitably extended and signatures renewed.

6. A new timestamped consistent snapshot MUST be issued.

This would preemptively protect all of these roles even though only one of them
may have been compromised.

If a threshold number of the *targets* or *claimed* keys have been compromised,
then there is little that an attacker could do without the *timestamp* and
*consistent-snapshot* keys.  In this case, PyPI MUST simply revoke the
compromised *targets* or *claimed* keys by replacing them with new keys in the
*root* and *targets* roles respectively.

If a threshold number of the *timestamp*, *consistent-snapshot* and *claimed*
keys have been compromised, then PyPI MUST take the following steps in addition
to the steps taken when either the *timestamp* or *consistent-snapshot* keys
are compromised:

1. Revoke the *claimed* role keys from the *targets* role and replace them with
   newly issued keys.

2. All project targets of the *claimed* roles SHOULD be compared with the last
   known good consistent snapshot where none of the *timestamp*,
   *consistent-snapshot* or *claimed* keys were known to have been compromised.
   Added, updated or deleted targets in the compromised consistent snapshot
   that do not match the last known good consistent snapshot MAY be restored to
   their previous versions.  After ensuring the integrity of all *claimed*
   project targets, the *claimed* metadata MUST be regenerated.

3. The *claimed* metadata MUST have their version numbers incremented, expiry
   times suitably extended and signatures renewed.

If a threshold number of the *timestamp*, *consistent-snapshot* and *targets*
keys have been compromised, then PyPI MUST take the union of the steps taken
when the *claimed*, *recently-claimed* and *unclaimed* keys have been
compromised.

If a threshold number of the *root* keys have been compromised, then PyPI MUST
take the steps taken when the *targets* role has been compromised as well as
replace all of the *root* keys.

It is also RECOMMENDED that PyPI sufficiently document compromises with
security bulletins.  These security bulletins will be most informative when
users of pip with TUF are unable to install or update a project because the
keys for the *timestamp*, *consistent-snapshot* or *root* roles are no longer
valid.  They could then visit the PyPI web site to consult security bulletins
that would help to explain why they are no longer able to install or update,
and then take action accordingly.  When a threshold number of *root* keys have
not been revoked due to a compromise, then new *root* metadata may be safely
updated because a threshold number of existing *root* keys will be used to sign
for the integrity of the new *root* metadata so that TUF clients will be able
to verify the integrity of the new *root* metadata with a threshold number of
previously known *root* keys.  This will be the common case.  Otherwise, in the
worst case where a threshold number of *root* keys have been revoked due to a
compromise, an end-user may choose to update new *root* metadata with
`out-of-band`__ mechanisms.

__ https://en.wikipedia.org/wiki/Out-of-band#Authentication


Appendix: Rejected Proposals
============================


Alternative Proposals for Producing Consistent Snapshots
--------------------------------------------------------

The complete file snapshot (CFS) scheme uses file system directories to store
efficient consistent snapshots over time.  In this scheme, every consistent
snapshot will be stored in a separate directory, wherein files that are shared
with previous consistent snapshots will be `hard links`__ instead of copies.

__ https://en.wikipedia.org/wiki/Hard_link

The `differential file`__ snapshot (DFS) scheme is a variant of the CFS scheme,
wherein the next consistent snapshot directory will contain only the additions
of new files and updates to existing files of the previous consistent snapshot.
(The first consistent snapshot will contain a complete set of files known
then.)  Deleted files will be marked as such in the next consistent snapshot
directory.  This means that files will be resolved in this manner: First, set
the current consistent snapshot directory to be the latest consistent snapshot
directory.  Then, any requested file will be seeked in the current consistent
snapshot directory.  If the file exists in the current consistent snapshot
directory, then that file will be returned.  If it has been marked as deleted
in the current consistent snapshot directory, then that file will be reported
as missing.  Otherwise, the current consistent snapshot directory will be set
to the preceding consistent snapshot directory and the previous few steps will
be iterated until there is no preceding consistent snapshot to be considered,
at which point the file will be reported as missing.

__ http://dl.acm.org/citation.cfm?id=320484

With the CFS scheme, the trade-off is the I/O costs of producing a consistent
snapshot with the file system.  As of October 2013, we found that a fairly
modern computer with a 7200RPM hard disk drive required at least three minutes
to produce a consistent snapshot with the "cp -lr" command on the ext3__ file
system.  Perhaps the I/O costs of this scheme may be ameliorated with advanced
tools or file systems such as ZFS__ or btrfs__.

__ https://en.wikipedia.org/wiki/Ext3
__ https://en.wikipedia.org/wiki/ZFS
__ https://en.wikipedia.org/wiki/Btrfs

While the DFS scheme improves upon the CFS scheme in terms of producing faster
consistent snapshots, there are at least two trade-offs.  The first is that a
web server will need to be modified to perform the "daisy chain" resolution of
a file.  The second is that every now and then, the differential snapshots will
need to be "squashed" or merged together with the first consistent snapshot to
produce a new first consistent snapshot with the latest and complete set of
files.  Although the merge cost may be amortized over time, this scheme is not
conceptually si




References
==========

.. [1] https://pypi.python.org
.. [2] https://isis.poly.edu/~jcappos/papers/samuel_tuf_ccs_2010.pdf
.. [3] http://www.pip-installer.org
.. [4] https://wiki.python.org/moin/WikiAttack2013
.. [5] https://github.com/theupdateframework/pip/wiki/Attacks-on-software-repositories
.. [6] https://mail.python.org/pipermail/distutils-sig/2013-April/020596.html
.. [7] https://mail.python.org/pipermail/distutils-sig/2013-May/020701.html
.. [8] https://mail.python.org/pipermail/distutils-sig/2013-July/022008.html
.. [9] PEP 381, Mirroring infrastructure for PyPI, Ziadé, Löwis
       http://www.python.org/dev/peps/pep-0381/
.. [10] https://mail.python.org/pipermail/distutils-sig/2013-September/022773.html
.. [11] https://mail.python.org/pipermail/distutils-sig/2013-May/020848.html
.. [12] PEP 449, Removal of the PyPI Mirror Auto Discovery and Naming Scheme, Stufft
        http://www.python.org/dev/peps/pep-0449/
.. [13] https://isis.poly.edu/~jcappos/papers/cappos_mirror_ccs_08.pdf
.. [14] https://mail.python.org/pipermail/distutils-sig/2013-September/022755.html
.. [15] https://pypi.python.org/security
.. [16] https://github.com/theupdateframework/tuf/blob/develop/docs/tuf-spec.txt
.. [17] PEP 426, Metadata for Python Software Packages 2.0, Coghlan, Holth, Stufft
        http://www.python.org/dev/peps/pep-0426/
.. [18] https://en.wikipedia.org/wiki/Continuous_delivery
.. [19] https://mail.python.org/pipermail/distutils-sig/2013-August/022154.html
.. [20] https://en.wikipedia.org/wiki/RSA_%28algorithm%29
.. [21] https://en.wikipedia.org/wiki/Key-recovery_attack
.. [22] http://csrc.nist.gov/publications/nistpubs/800-57/SP800-57-Part1.pdf
.. [23] https://www.openssl.org/
.. [24] https://pypi.python.org/pypi/pycrypto
.. [25] http://ed25519.cr.yp.to/


Acknowledgements
================

Nick Coghlan, Daniel Holth and the distutils-sig community in general for
helping us to think about how to usably and efficiently integrate TUF with
PyPI.

Roger Dingledine, Sebastian Hahn, Nick Mathewson,  Martin Peck and Justin
Samuel for helping us to design TUF from its predecessor Thandy of the Tor
project.

Konstantin Andrianov, Geremy Condra, Vladimir Diaz, Zane Fisher, Justin Samuel,
Tian Tian, Santiago Torres, John Ward, and Yuyu Zheng for helping us to develop
TUF.

Vladimir Diaz, Monzur Muhammad and Sai Teja Peddinti for helping us to review
this PEP.

Zane Fisher for helping us to review and transcribe this PEP.


Copyright
=========

This document has been placed in the public domain.