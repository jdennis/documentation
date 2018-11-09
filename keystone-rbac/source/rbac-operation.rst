Author:
    John Dennis <jdennis@redhat.com>

Version:
    1.0

Date:
    November 9, 2018
    
Introduction
============

Increasingly OpenStack deployments are now authoring their own custom
policy as opposed to using the stock default policy that ships with
OpenStack. Policy can be complex and the full interactions difficult
to comprehend. As with any complex technology there is a need to be
able to validate the behavior and debug when policy does not work as
expected. Preferably the validation and debugging of policy would be
done standalone and in isolation as opposed to inside a running
OpenStack deployment where the task is significantly more difficult.

In November of 2018 an attempt was made to debug custom policy in
OpenStack version 13 (Queens) where the Keystone policy was not
operating as expected. Policy enforcement is now performed through the
oslo.policy component which includes a command line tool
`oslopolicy-checker`. This tool seemed like the ideal tool to test and
validate policy in isolation. However it was discovered
`oslopolicy-checker` was unable to perform the task.

Policy Evaluation Issues
------------------------

Policy evaluation requires 3 pieces of information:

1. Policy Rules
2. User Credentials (auth context)
3. Run Time Object Data (target)

Unfortunately `oslopolicy-checker` only allows you to specify items 1
and 2. There is no mechanism to provide target data, a crucial
component of policy evaluation. The absence of target data means it's
is impossible to validate policy rules.

What is Target Data?
--------------------

The policy documentation never explains:

* What target data is.
* What information needs to be in the target data.
* How the target data needs to be formatted.
* Where the target data comes from at run time.

The problem is further compounded by the fact the policy documentation
uses different and conflicting terminology than the policy
implementation.

What is in a name?
``````````````````

Most users are introduced to OpenStack Policy by reading the
documentation on policies files which defines a policy rule like this: 

::

  Each policy is defined by a one-line statement in the form
  "<target>": "<rule>".

  The policy target, also named “action”, represents an API call like
  “start an instance” or “attach a volume”.

  A simple rule might look like this:

  "compute:get_all" : ""

  The target is "compute:get_all", the “list all instances” API of the
  Compute service.

  Targets are APIs and are written "service:API" or simply "API". For
  example, "compute:create" or "add_image".

  Two values are compared in the following way:

  "value1 : value2"

  Possible values are:

  * constants: Strings, numbers, true, false
  * API attributes
  * target object attributes
  * the flag is_admin

  API attributes can be project_id, user_id or domain_id.

  Target object attributes are fields from the object description in
  the database.

The reader of the documentation is clearly left with the impression
that `target` is the name of an API entry point. Furthermore the term
`target object attribute` is introduced with no explanation of how the
policy enforcer in oslo.policy gets access to these data values when
it evaluates a policy rule.

The oslo.policy method that evaluates policy is defined this way:

.. code-block:: python

  def enforce(self, rule, target, creds, do_raise=False, exc=None,
              *args, **kwargs):

**The problem is the rule and target parameters in the `enforce`
method have entirely different meanings than what is described in the
policy documentation.**

*At the code level a target is a dict of key/value pairs and a rule is
the name of an API entry point.*

Until you understand the terms `target` and `rule` are used completely
differently depending upon whether you're reading the policy
documentation vs. reading the implementation in code you'll be
hopelessly confused.

The following table illustrates how the terms are used in each
context. An empty table entry means the item is not used in that
context. 

+---------------------+----------------------------------------------+
|     Description     |                Context                       |
|                     +-------------+-----------------+--------------+
|                     | Policy File | policy.Enforcer | Keystone     |
+---------------------+-------------+-----------------+--------------+
|  rule definition    | rule        | rule [1]_       |              |
+---------------------+-------------+-----------------+--------------+
|  rule name          | target      | rule [1]_       | action       |
+---------------------+-------------+-----------------+--------------+
|  target dict        |             | target          | target       |
+---------------------+-------------+-----------------+--------------+
|  auth context       | credentials | creds           | auth context |
+---------------------+-------------+-----------------+--------------+

Reconciling policy documentation to policy implementation
`````````````````````````````````````````````````````````

The policy `enforce` method requires 3 pieces of information:

rule
    This is the name of the rule. [1]_ It is the same thing as
    `target` in the policy documentation
    (e.g. `identity:list_projects`).
    See `rule name determination <rule_name_determination_>`_ to learn
    how API entry points obtain their matching rule name.

target
    This is a dict the policy enforcer uses to look up run-time values
    referenced by the policy rule. It contains the `target object
    attributes` the policy documentation refers to under the `target`
    key. The target_dict contains other run-time values in addition to
    target resource values.

auth context
    Despite the policy documentation calling this credentials it
    isn't. Credentials are used to obtain a Keystone token and the
    token is exchanged to obtain auth context data such as 
    user id, user domain, user project user roles, etc.

Obtaining Target Data from Keystone
-----------------------------------

`oslopolicy-checker` can easily be extended to accept target
data. Target data as passed to the policy enforcer is a `flattened
<dict_flattening_>`_ dict of key/value pairs, henceforth this will be
called the target dict. The target dict can easily be represented by a
JSON object and passed into `oslopolicy-checker`. One would expect you
could obtain the token auth context (contains the roles which is
essential for policy evaluation) **and** the target dict from
Keystone's debug log. Unfortunately that is not the case, neither
Keystone nor the policy enforcer code emits the critical target
information.

Keystone calls the olso policy enforcer in
keystone/policy/backends/rules.py like this:

.. code-block:: python

    class Policy(base.PolicyDriverBase):
        def enforce(self, credentials, action, target):
            msg = 'enforce %(action)s: %(credentials)s'
            LOG.debug(msg, {
                'action': action,
                'credentials': credentials})
            policy.enforce(credentials, action, target)

Notice how the critical target data is not logged.


How Keystone Enforces Policy in V3
==================================

The above issues led me to try to understand how Keystone computes
it's target data and more importantly how Keystone implements it's
policy enforcement. I was not able to find any comprehensive
documentation on the Keystone policy implementation. It remained
mysterious and I felt if I needed to debug Keystone policy problems at
a minimum I needed to understand how it worked. My investigation was
based on the Queens release. Since that time (mostly during 2018) the
master branch has seen significant modification of the Keystone policy
implementation. Some of what is in this document will likely carry over
but this document is also useful to those who have to support prior
releases. 

.. _single_resource_loading:

How is resource data loaded into the target?
--------------------------------------------

Keystone manages several distinct collections of data such as users, projects,
etc. Each member of the collection is identified by it's unique `id`
and is called a resource. Resources are looked up by the collection
provider using it's `id`. The returned resource is a dict and is often
called a `ref <ref_>`_ in the source code.

Policy rules can utilize an API attribute such as::

  "os_compute_api:servers:start" : "project_id:%(project_id)s"

Here the `project_id` is the API attribute and it is compared for
equality against the `project_id` in the resource. To evaluate the
rule the policy enforcer needs access to the resource's
`project_id`. This is done by looking up the resource by it's `id` and
adding it to the target_dict passed to the policy enforcer.

Each resource collection is represented by a subclass of the
`V3Controller` class which defines these class attributes:

`collection_name`
    The name of the collection, e.g. 'projects'

`member_name`
    The name of an item in the collection, e.g. 'project'

`get_member_from_driver`
    Function that takes an `id` as a parameter, looks up the resource
    in the collection and returns it as a `ref`_

`_public_parameters`
    Set of parameters that are exposed to the user.
    Usually used by cls.filter_params()

The flow works like this:

1. Each API entry point is decorated with a `protected()` or
   `filterprotected()` decorator whose purpose is to enforce
   RBAC. The decorator eventually calls
   `authorization.check_policy()`

2. `check_policy()` calls `_handle_member_from_driver()` to load the
   resource.

3. `_handle_member_from_driver()` uses the `get_member_from_driver`
   and `member_name` class attributes. It forms a lookup key name by
   appending the string "_id" to the `member_name` attribute (e.g. the
   projects collection has a member_name of "project" thus it's lookup
   key name is "project_id"). If the class does not have a
   `get_member_from_driver` attribute no resource is loaded.

4. The lookup key name is looked up in the kwargs of the API entry
   point to obtain the collection id key. (e.g. kwargs =
   {'project_id': 'abcd1234567890'}, thus the collection key is
   'abcd1234567890')

5. `get_member_from_driver()` is passed the collection key which
   returns a ref_ to the resource data.

6. The target_dict is updated with a 'target' key whose value is
   a dict with a `member_name` key whose value is the ref_. For
   example:
     
.. code-block:: python

   target_dict['target'] = {self.member_name: ref}

Thus continuing with the project example the target_dict will have a
key/value pair that looks like this:

.. code-block:: python

   {'target': {'project': project_ref}}

And after `flattening <dict_flattening_>`_ the target_dict looks like this::

  {'target.project': {project_ref}}

Resource Definitions
````````````````````

To better understand how `collection_name`, `member_name` and
`get_member_from_driver` interact it is instructional to look at the
resource class definitions (this table is based on the code in Queens).

+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| Class                            | collection_name         | member_name            | get_member_from_driver                      | 
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ApplicationCredentialV3          | application_credentials | application_credential |                                             |
|                                  |                         |                        |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ProjectAssignmentV3              | projects                | project                | PROVIDERS.resource_api.get_project          |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| RoleV3                           | roles                   | role                   | PROVIDERS.role_api.get_role                 |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| GrantAssignmentV3                | roles                   | role                   | PROVIDERS.role_api.get_role                 |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| RoleAssignmentV3                 | role_assignments        | role_assignment        |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| Auth                             | tokens                  | token                  |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| RegionV3                         | regions                 | region                 |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ServiceV3                        | services                | service                | PROVIDERS.catalog_api.get_service           |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| EndpointV3                       | endpoints               | endpoint               | PROVIDERS.catalog_api.get_endpoint          |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| EndpointGroupV3Controller        | endpoint_groups         | endpoint_group         |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ProjectEndpointGroupV3Controller | project_endpoint_groups | project_endpoint_group |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| V3Controller                     | entities                | entity                 |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| Ec2ControllerV3                  | credentials             | credential             |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| CredentialV3                     | credentials             | credential             | PROVIDERS.credential_api.get_credential     |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| EndpointPolicyV3Controller       | endpoints               | endpoint               |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| IdentityProvider                 | identity_providers      | identity_provider      |                                             |
|                                  |                         |                        |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| FederationProtocol               | protocols               | protocol               |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| MappingController                | mappings                | mapping                |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| DomainV3                         | domains                 | domain                 | PROVIDERS.resource_api.get_domain           |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ProjectAssignmentV3              | projects                | project                | PROVIDERS.resource_api.get_project          |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ServiceProvider                  | service_providers       | service_provider       |                                             |
|                                  |                         |                        |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| UserV3                           | users                   | user                   | PROVIDERS.identity_api.get_user             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| GroupV3                          | groups                  | group                  | PROVIDERS.identity_api.get_group            |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| RegisteredLimitV3                | registered_limits       | registered_limit       | self.unified_limit_api.get_registered_limit |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| LimitV3                          | limits                  | limit                  | self.unified_limit_api.get_limit            |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ConsumerCrudV3                   | consumers               | consumer               |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| AccessTokenCrudV3                | access_tokens           | access_token           |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| AccessTokenRolesV3               | roles                   | role                   |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| OAuthControllerV3                | not_used                | not_used               |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| PolicyV3                         | policies                | policy                 |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| DomainV3                         | domains                 | domain                 | PROVIDERS.resource_api.get_domain           |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ProjectV3                        | projects                | project                | PROVIDERS.resource_api.get_project          |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| ProjectTagV3                     | projects                | tags                   | PROVIDERS.resource_api.get_project_tag      |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+
| TrustV3                          | trusts                  | trust                  |                                             |
+----------------------------------+-------------------------+------------------------+---------------------------------------------+ 

.. _multiple_resource_loading:

Multiple resource data loading
``````````````````````````````

The process described above only works for the common case of a single
resource which is implicitly defined by the API entry point. For
example the API entry points dealing with projects implicitly use
project resources looked up by the project_id in the API call. But
some API entry points use multiple resource types in addition to their
implicit resource type. How are these other resources loaded into the
target_dict?

This is handled by the `callback` arg of the `protected` and
`filterprotected` decorators. The callback is responsible for adding
the resource data before `check_protection()` is invoked. In essence
it overrides the target resource data added by
`get_member_from_driver()`. Let's look at an example from the `UserV3`
class.

.. code-block:: python

    @controller.protected(callback=_check_user_and_group_protection)
    def add_user_to_group(self, request, user_id, group_id):
                
The callback implementation looks like this:

.. code-block:: python

    def _check_user_and_group_protection(self, request, prep_info,
                                         user_id, group_id):
        ref = {}
        ref['user'] = PROVIDERS.identity_api.get_user(user_id)
        ref['group'] = PROVIDERS.identity_api.get_group(group_id)
        self.check_protection(request, prep_info, ref)

When `check_protection()` is called it sets the `target` key in the
target_dict to the 3rd `check_protection()` parameter
`target_attr`. In the above example that is the `ref`. Thus the
target_dict ends up looking like this:

.. code-block:: python

    {'target': {'user': user_ref,
                'group': group_ref}}


.. _rule_name_determination:

How does an API entry point determine it's rule name?
`````````````````````````````````````````````````````

The `protected()` and `filterprotected()` decorators are used to
enforce RBAC policy on API entry points. The decorators eventually
call `protected_wrapper()` which obtains the name of the function
being decorated and stores it in the target_dict under the key
`f_name`. The `check_protection()` function extracts the `f_name` key
and forms the rule name by concatenating the identity service name, a
dot and the `f_name`. This is then passed to the `check_policy()`
function as the `action` parameter. Thus the exact same data item is
called `target` in the policy documentation, an `action` in Keystone,
and a `rule` in the oslo.policy `enforce()` function. Confused yet?

.. _check_protection_function:

The authorization.check_protection() function
`````````````````````````````````````````````

The `check_protection()` function is the main function in Keystone to
evaluation RBAC policy. It is called by API entry point decorators as
well as other functions when a policy authorization decision must be
made. 

`check_protection()` does very little work of it's own, it calls
`check_policy()` where the bulk of the work is done.

.. _check_policy_function:

The authorization.check_policy() function
`````````````````````````````````````````

The `check_policy()` function signature looks like this:

.. code-block:: python

  def check_policy(controller, request, action,
                   filter_attr=None, input_attr=None, target_attr=None,
                   *args, **kwargs):

Unfortunately the parameters are not documented in the code, here is
their description:


controller
    This is really `self`. All classes using RBAC are derived from
    the `V3Controller` class.

request
    The WebOb request object. It contains all the HTML information
    pertaining to the request, e.g. the HTML method, HTML headers, URL
    query parameters, etc.

action
    This is the policy rule name. See `rule name determination
    <rule_name_determination_>`_ for an explantion of how Keystone
    obtains this value.

filter_attr
    This is a dict of URL query parameters supplied with API entry
    points used to list data. They are used to control the data
    returned by the list operation. See this `explanation of filters
    <filters_>`_ for more detail.

input_attr
    This is the dict of kwargs from the policy decorator. It appears
    to be identical to the kwargs of this function.

target_attr
    This is the dict computed when there are
    `multiple resource targets <multiple_resource_loading_>`_ or when
    the `implicit single resource target <single_resource_loading_>`_
    needs to be overridden.

args
    This is the array of positional args passed to the API entry point.

kwargs
    This is the dict of keyword args passed to the API entry point.

Here is `check_policy()` implementation from Queens. This is where all
material discussed in this document comes together. 

.. code-block:: python

  def check_policy(controller, request, action,
                   filter_attr=None, input_attr=None, target_attr=None,
                   *args, **kwargs):
      # Makes the arguments from check protection explicit.
      request.assert_authenticated()
      if request.context.is_admin:
          LOG.warning('RBAC: Bypassing authorization')
          return

      # TODO(henry-nash) need to log the target attributes as well
      creds = _build_policy_check_credentials(
          action, request.context_dict, input_attr)
      # Build the dict the policy engine will check against from both the
      # parameters passed into the call we are protecting plus the target
      # attributes provided.
      policy_dict = {}
      _handle_member_from_driver(controller, policy_dict, **kwargs)
      _handle_subject_token_id(controller, request, policy_dict)

      if target_attr:
          policy_dict = {'target': target_attr}
      if input_attr:
          policy_dict.update(input_attr)
      if filter_attr:
          policy_dict.update(filter_attr)

      for key in kwargs:
          policy_dict[key] = kwargs[key]
      controller.policy_api.enforce(creds,
                                    action,
                                    utils.flatten_dict(policy_dict))
      LOG.debug('RBAC: Authorization granted')

There are a few oddities in the implementation.

1. `_handle_member_from_driver()` and `_handle_subject_token_id()` are
   invoked to add initial data to the `policy_dict`. However if a
   `target_attr` is passed the very next action is to discard it.

2. `input_attr` contains the kwargs, but then the `policy_dict` is
   updated with the kwargs again (why iterate instead of
   update?). Maybe for historical reasons some callers never passed
   the `input_attr` so the kwargs are added to the `policy_dict` just
   to be sure?

What does the `_handle_member_from_driver()` do?
::::::::::::::::::::::::::::::::::::::::::::::::

It loads the resource data associated with the controllers type
(e.g. project, user, etc.). See the `single resource loading
<single_resource_loading_>`_ section for detailed explanation.

The term "member" in the name of the function is perhaps misleading
because for many "member" implies group membership. In this context
"member" has an entirely different meaning. Controllers manage a
collection of data, item in the collection is called a "member". In
fact the `V3Controller` class defines `collection_name` and
`member_name` class level attributes as wells as the
`get_member_from_driver` function pointer. Thus in terms of a
collection "member" makes sense, just don't confuse it with group
membership.

What does the `handle_subject_token_id()` do?
::::::::::::::::::::::::::::::::::::::::::::::

This handles delegation. Mostly used by services who are acting on
behalf of a user (or other principal). The delegate is passed in the
`X-Subject-Token` HTTP header.  If request has `X-Subject-Token` HTTP
header then validate the `X-Subject-Token` and add the `user_id` and
`user_domain_id` to the target_dict:

A simplified version of the code looks like this:

.. code-block:: python

    token_ref = token_model.KeystoneToken(
                    token_id=subject_token_id,
                    token_data=token_provider_api.validate_token(subject_token_id))

    target['target'][member_name]['user_id'] = token_ref.user_id
    if token_ref.user_domain_id:
        target['target']['member_name']['user']['domain']['id'] = user_domain_id

    {'user_id': token_ref.user_id,
     'user': {'domain': {'id': token_ref.user_id}}}




.. _filters:

What are filters and how are they added to the target?
``````````````````````````````````````````````````````

The `filterprotected` decorator allows you to specify a list of filter
names. The API entry points which list data optionally allow you to
indicate what data you want returned. These are referred to as filters
and are passed as URL query parameters to a GET method used to list
the resource data. Let's use the `List Projects
<https://developer.openstack.org/api-ref/identity/v3/?expanded=validate-and-show-information-for-token-detail,create-project-detail,list-projects-detail#list-projects>`_
API as an example. It specifies `domain_id`, `enabled`, `name`,
`parent_id`, and `is_domain` as filters.

The `list_projects` implementation looks like this:

.. code-block:: python

    @controller.filterprotected('domain_id', 'enabled', 'name',
                                'parent_id', 'is_domain')
    def list_projects(self, request, filters):

What the `filterprotected` decorator does is to iterate over the list
of filters and looks to see if that filter name is one of the URL
query parameters, if so it adds it to the target_dict. For example
using this URL::

    GET v3/projects?is_domain=1

The target_dict would be would contain:

.. code-block:: python

    {'is_domain': 1}



Questions and Answers
---------------------

What does the `callback` argument do in the `protected` and `filterprotected` decorators do?
    See the section on `Multiple Resource Loading <multiple_resource_loading_>`_

.. _ref:

What is a `ref`?
    A Python object (usually a dict) returned by a collection provider
    given that resources unique `id`.

.. _dict_flattening:

What is dict flattening?

    Dicts are often complex with arbitrarily deep nesting of other
    dicts (i.e. a key whose value is yet another dict). Flattening
    traverses the key path to a final value joining each path element
    with a dot ('.'). The result is a dict whose nesting has been
    flattened out with just one set of keys of the form
    "foo.bar.blatz" given this dict `{'foo': {'bar': {'blatz': value}}}`. 


.. [1] Actually the `rule` parameter can be either the name of a rule
       or a compiled rule object derived from `BaseCheck`.       
