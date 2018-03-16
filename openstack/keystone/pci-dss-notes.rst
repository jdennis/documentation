PCI-DSS
=======

Introduction
============

Questions often arise as to whether OpenStack is PCI-DSS
compliant. Because of these following factors the answer is "it
depends".

* What version of OpenStack?
* What version of PCI-DSS?
* How has OpenStack been deployed and configured?
* Which of multiple OpenStack identity backends are in use?
* Which Keystone authentication plugins are in use?

To complicate matters the necessary components to achieve PCI-DSS
compliance have been added to OpenStack over the course of multiple
releases and have evolved (and will continue to evolve).

This document attempts to pull together in one place all the necessary
elements to answer these questions. We start with some general pieces
of relevant information and then list each PCI-DSS requirement in
section 8 of the current PCI-DSS specification enumerating as best we
can these details:

* The PCI-DSS requirement.
* Does OpenStack satisfy the requirement?
* How OpenStack satisfies the requirement?
* When was support for the requirement added to OpenStack?
* What was the Gerrit review that introduced the support requirement?
* Any expository material relevant to the requirement unique to OpenStack. 

.. IMPORTANT::

  PCI-DSS compliance in OpenStack applies **only** to the Keystone SQL
  identity backend. If any other identity backend is used it is the
  responsibility of that driver to implement the PCI-DSS
  compliance. This includes using federated identity, even when
  federation is used in conjunction with shadow users.


.. NOTE::

   Some OpenStack projects can be run outside of OpenStack and thus
   have their authentication systems independent of OpenStack. Swift
   is an an example. Typically Swift when used in an OpenStack
   environment will be deployed configured to use Keystone
   authentication, however this is not mandatory. The discussions
   concerning PCI-DSS apply only to those components using Keystone
   authentication.

   See `Swift Authentication`_ for more details regarding Swift.


.. _os_mfa:

OpenStack Multi-factor authentication
=====================================

Several of the PCI-DSS requirements refer to multi-factor
authentication (MFA). To avoid duplicating the discussion of MFA the
material on MFA is gathered here in one section.

Keystone added Multi-factor authentication (MFA) support in the Ocata
release. The `Per-User Auth Plugin Requirements`_ blueprint describes
how MFA is intended to work.

The `Keystone Ocata Release Notes
<https://docs.openstack.org/releasenotes/keystone/ocata.html>`_
describe how to configure and use the per-user MFA functionality, see
the section titled `blueprint per-user-auth-plugin-reqs`.

If the per-user-auth functionality implements MFA then why isn't it
called MFA instead? Because the MFA rules **must** be explicitly set
for *every* user. As of the Queens release there is **no site-wide
global MFA rules that applies to all users by default**. Hence the
initial MFA blueprint is *per-user*. If and when site-wide default MFA
rules are added it is anticipated the site-wide MFA rules will follow
the same model as the per-user rules.

.. IMPORTANT::
   Keystone MFA satisfies the PCI-DSS MFA requirement. But because
   Keystone MFA is managed on a *per-user* basis it adds an additional
   workflow burden. A failure to set MFA rules on a user (either
   previously provisioned or newly provisioned) invalidates
   PCI-DSS MFA requirements 8.3, 8.3.1, and 8.3.2.

.. NOTE::
   The only *traditional* 2nd factor auth method currently supported by
   Keystone is `Time-based One-time Password` (TOTP). TOTP was added
   in Mitaka and is described in `Keystone Time-based One-time
   Password`_

.. NOTE::
   PCI-DSS MFA 8.3 requirements specify MFA for *non-console* and
   *remote network* access. Keystone is unable to distinguish the
   origin of the authentication request therefore MFA must be enabled
   in all contexts.

PCI DSS Requirements
====================

Below are the all the PCI-DSS requirements from section 8 of the
PCI-DSS specification along with information on if, how and when
OpenStack satisfies the requirement.

Requirement 8.1
---------------

*Define and implement policies and procedures to ensure proper user
identification management for non-consumer users and administrators
on all system components.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

This is a site management task.

**CONCLUSION:** Not Applicable.

Requirement 8.1.1
-----------------

*Assign all users a unique ID before allowing them to access
system components or cardholder data.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

The core SQL identity backend has always assured a unique user id.

**CONCLUSION:** Satisfied

Requirement 8.1.2
-----------------

*Control addition, deletion, and modification of user IDs,
credentials, and other identifier objects.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

The `Keystone v3 Users API
<https://developer.openstack.org/api-ref/identity/v3/#users>`_ provides:

* List Users
* Create User
* Show user details
* Update User
* Delete User
* List groups to which a user belongs
* List projects for user
* Change password for user

The `Keystone v3 Credentials API
<https://developer.openstack.org/api-ref/identity/v3/#credentials>`_ provides:

* Create credential
* List credentials
* Show credential details
* Update credential
* Delete credential

**CONCLUSION:** Satisfied

Requirement 8.1.3
-----------------

*Immediately revoke access for any terminated users.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

The Keystone v3 Token API provides `revoke-token
<https://developer.openstack.org/api-ref/identity/v3/#revoke-token>`_
which immediately revokes a token. However there are a few caveats to
consider. 

Validating the identity of every client on every request can impact
performance for both the OpenStack service and the identity
service. As a result, keystonemiddleware is configurable to cache
authentication responses from the identity service. Tokens invalidated
after they’ve been stored in the cache may continue to
work. See this configuration option::

  # default 300 seconds
  # Set to -1 to disable caching completely.
  token_cache_time = 300 

Also note that service accounts may allow a token to continue to be
used past it's expiration. However this PCI-DSS requirement addresses
the revocation of users, not services therefore this does not apply.

**CONCLUSION:** Satisfied (with appropriate configuration)

Requirement 8.1.4
-----------------

*Remove/disable inactive user accounts within 90 days.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

See
`Disabling inactive users <https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#disabling-inactive-users>`_
for more information.

**OpenStack Change:** https://review.openstack.org/#/c/328447/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied

Requirement 8.1.5
-----------------

*Manage IDs used by third parties to access, support, or maintain
system components via remote access as follows:*

* *Enabled only during the time period needed and disabled when not in use.*

* *Monitored when in use.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

OpenStack provides several ways to disable a user:

* set the user `enabled` flag to false
* remove all roles from the user
* remove all projects from the user

The user `enabled` flag enables or disables the user. An enabled user can
authenticate and receive authorization. A disabled user cannot
authenticate or receive authorization. Additionally, all tokens that
the user holds become no longer valid. If you re-enable this user,
pre-existing tokens do not become valid. To enable the user, set to
true. To disable the user, set to false. Default is true.

User activity can be monitored via the `Audit Middleware
<https://docs.openstack.org/keystonemiddleware/latest/audit.html>`_
OpenStack component.

**CONCLUSION:** Satisfied

Requirement 8.1.6
-----------------

*Limit repeated access attempts by locking out the user ID after
not more than six attempts.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

See `Setting an account lockout threshold
<https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#setting-an-account-lockout-threshold>`_
for more information.

**OpenStack Change:** https://review.openstack.org/#/c/340074/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied


Requirement 8.1.7
-----------------

*Set the lockout duration to a minimum of 30 minutes or until an
administrator enables the user ID.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

See `lockout_duration` in `Setting an account lockout threshold
<https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#setting-an-account-lockout-threshold>`_
for more information.

**OpenStack Change:** https://review.openstack.org/#/c/340074/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied


Requirement 8.1.8
-----------------

*If a session has been idle for more than 15 minutes, require the
user to re-authenticate to re-activate the terminal or session.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

Keystone has had the token.expiration config value for a long time.
From the Keystone Token doc::

  # The amount of time that a token should remain valid (in seconds). Drastically
  # reducing this value may break "long-running" operations that involve multiple
  # services to coordinate together, and will force users to authenticate with
  # keystone more frequently. Drastically increasing this value will increase
  # load on the `[token] driver`, as more tokens will be simultaneously valid.
  # Keystone tokens are also bearer tokens, so a shorter duration will also
  # reduce the potential security impact of a compromised token. (integer value)
  # Minimum value: 0
  # Maximum value: 9223372036854775807
  #expiration = 3600


**OpenStack Change:** https://review.openstack.org/#/c/3942/

**First Appeared in Release:** essex-4

**CONCLUSION:** Satisfied

Requirement 8.2
---------------

*In addition to assigning a unique ID, ensure proper
user-authentication management for non-consumer users and
administrators on all system components by employing at least one of
the following methods to authenticate all users:*

* *Something you know, such as a password or passphrase.*

* *Something you have, such as a token device or smart card.*

* *Something you are, such as a biometric.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

OpenStack requires both authentication and authorization for virutally
all operations.

**CONCLUSION:** Satisfied


Requirement 8.2.1
-----------------

*Using strong cryptography, render all authentication credentials
(such as passwords/phrases) unreadable during transmission and storage
on all system components.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

Requirement 8.2.1 is actually two independent requirements:

1. encrypt credentials in transit
2. encrypt credentials at rest

The first requirement to encrypt credentials in transit is satisfied
when Keystone requires the TLS protocol for client connection. The use
of TLS is not a function of OpenStack per se, rather TLS and it's
configuration properties are the province of OpenStack deployment
tools. As of this writing most of the deployment tools support
TLS. However, it is important to note even if a client is required to
use TLS to connect to Keystone it may not be an end-to-end TLS
connection if Keystone is deployed in a high availability (HA)
environment. This is because in most HA deployments TLS is terminated
at the front end *public* endpoint which then forwards the *unencrypted*
communication on a private internal network to one of many backend
servers via a load balancing arrangement. Thus the public
communication is encrypted but unencrypted data is exposed on the
private internal network. If the private internal network is
considered vulnerable then most load balances can be configured to use
TLS on the private network. All of this is managed by the deployment
and not OpenStack.

The second requirement to encrypt credentials at rest was implemented
in Newton, see the `Credential Encryption`_ documentation for
details. As noted above for the first requirement enabling this
feature requires configuration outside the scope of OpenStack proper,
most OpenStack deployment tools now support managing the encryption
keys necessary to support credential encryption.

**OpenStack Change:** https://review.openstack.org/#/c/355618/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied (with appropriate configuration)

Requirement 8.2.2
-----------------

*Verify user identity before modifying any authentication
credential—for example, performing password resets, provisioning new
tokens, or generating new keys.*

This is a site management task. OpenStack provides the necessary tools
but it is up to the site to implement the necessary workflow.

**CONCLUSION:** Not Applicable.

Requirement 8.2.3
-----------------

*Passwords/passphrases must meet the following:*

* *Require a minimum length of at least seven characters.*

* *Contain both numeric and alphabetic characters.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

See the `Password Strength`_ documentation for details.

**OpenStack Change:** https://review.openstack.org/#/c/320586/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied

Requirement 8.2.4
-----------------

*Change user passwords/passphrases at least once every 90 days.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^
See the `Password Expiration`_ documentation for details.

**OpenStack Change:** https://review.openstack.org/#/c/333360/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied

Requirement 8.2.5
-----------------

*Do not allow an individual to submit a new password/passphrase
that is the same as any of the last four passwords/passphrases he or
she has used.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

See the `Password History`_ documentation for details.

**OpenStack Change:** https://review.openstack.org/#/c/328339/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied

Requirement 8.2.6
-----------------

*Set passwords/passphrases for first-time use and upon reset to a
unique value for each user, and change immediately after the first
use.*

OpenStack Implementation
^^^^^^^^^^^^^^^^^^^^^^^^
See the `Password First Use`_ documentation for details.

**OpenStack Change:** https://review.openstack.org/#/c/403916/

**First Appeared in Release:** Newton

**CONCLUSION:** Satisfied

Requirement 8.3
---------------

*Secure all individual non-console administrative access and all
remote access to the CDE using multi-factor authentication.*

See `OpenStack Multi-factor authentication`_ for details and make note
of the console vs. non-console comment in that sections.

**CONCLUSION:** Satisfied

Requirement 8.3.1
-----------------

*Incorporate multi-factor authentication for all non-console
access into the CDE for personnel with administrative access.*

See `OpenStack Multi-factor authentication`_ for details and make note
of the console vs. non-console comment in that sections.

**CONCLUSION:** Satisfied

Requirement 8.3.2
-----------------

*Incorporate multi-factor authentication for all remote network
access (both user and administrator, and including third-party access
for support or maintenance) originating from outside the entity’s
network.*

See `OpenStack Multi-factor authentication`_ for details.

**CONCLUSION:** Satisfied

Requirement 8.4
---------------

*Document and communicate authentication policies and procedures to
all users including:*

* *Guidance on selecting strong authentication credentials.*

* *Guidance for how users should protect their authentication
  credentials.*

* *Instructions not to reuse previously used passwords.*

* *Instructions to change passwords if there is any suspicion the
  password could be compromised.*

This is a site management task.

**CONCLUSION:** Not Applicable.

Requirement 8.5
---------------

*Do not use group, shared, or generic IDs, passwords, or other
authentication methods as follows:*

* *Generic user IDs are disabled or removed.*

* *Shared user IDs do not exist for system administration and other
  critical functions.*

* *Shared and generic user IDs are not used to administer any system
  components.*

This is a site management task.

**CONCLUSION:** Not Applicable.

Requirement 8.5.1
-----------------

*Additional requirement for service providers only: Service
providers with remote access to customer premises (for example, for
support of POS systems or servers) must use a unique authentication
credential (such as a password/phrase) for each customer.*

This is a site management task. OpenStack provides the necessary tools
but it is up to the site to implement the necessary workflow.

**CONCLUSION:** Not Applicable.

Requirement 8.6
---------------

*Where other authentication mechanisms are used (for example,
physical or logical security tokens, smart cards, certificates, etc.),
use of these mechanisms must be assigned as follows:*

* *Authentication mechanisms must be assigned to an individual account
  and not shared among multiple accounts.*

* *Physical and/or logical controls must be in place to ensure only the
  intended account can use that mechanism to gain access.*

This is a site management task.

**CONCLUSION:** Not Applicable.

Requirement 8.7
---------------

*All access to any database containing cardholder data (including
access by applications, administrators, and all other users) is
restricted as follows:*

* *All user access to, user queries of, and user actions on databases
  are through programmatic methods.*

* *Only database administrators have the ability to directly access or
  query databases.*

* *Application IDs for database applications can only be used by the
  applications (and not by individual users or other non-application
  processes).*

This is a deployment issue. OpenStack itself does not enforce
restrictions on database access. However the tools that deploy
OpenStack usually enforce this, but given the number of deployment
scenarios and tools this can only be verified by auditing the
deployment to assure the database is locked down and any credentials
needed to access it are secured.

**CONCLUSION:** Outside of OpenStack & Keystone scope.

Requirement 8.8
---------------

*Ensure that security policies and operational procedures for
identification and authentication are documented, in use, and known to
all affected parties.*

This is a site management task.

**CONCLUSION:** Not Applicable.


.. _OpenStack PCI-DSS Security Compliance:
   https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html

.. _OpenStack Newton PCI-DSS Spec:
   https://specs.openstack.org/openstack/keystone-specs/specs/keystone/newton/pci-dss.html  

.. _Keystone Time-based One-time Password:
   https://docs.openstack.org/keystone/latest/advanced-topics/auth-totp.html

.. _Swift Authentication:
   https://docs.openstack.org/swift/latest/overview_auth.html

.. _Credential Encryption:
   https://docs.openstack.org/keystone/pike/admin/identity-credential-encryption.html

.. _Password Strength:
   https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#configuring-password-strength-requirements

.. _Password Expiration:
   https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#configuring-password-expiration

.. _Password History:
   https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#requiring-a-unique-password-history

.. _Password First Use:
   https://docs.openstack.org/keystone/latest/admin/identity-security-compliance.html#force-users-to-change-password-upon-first-use

.. _Keystone TOTP:
  https://docs.openstack.org/keystone/latest/advanced-topics/auth-totp.html

.. _Per-User Auth Plugin Requirements:
  https://specs.openstack.org/openstack/keystone-specs/specs/keystone/ocata/per-user-auth-plugin-requirements.html
