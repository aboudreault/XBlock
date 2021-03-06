=======
XBlocks
=======

XBlocks are Python classes that implement a small web application. Like full
applications, they have state and methods, and operate on both the server and
the client.


Python Structure
----------------

XBlocks are implemented as Python code, and packaged using standard Python
techniques.  They have Python code, and other file resources, including CSS and
Javascript needed to fully render themselves in a browser.


.. _guide_fields:

Fields
------

XBlock state (or data) is arbitrary JSON-able data stored in Python attributes
called **fields**.

Fields declare their relationship to both blocks and users, by specifying their
**scope**.

- By user: fields declare how they relate to the user:

  * No user: the data is related to no users, for example, the content of the
    course which is independent of any users.

  * One user: the data is particular to a single user, such as an answer to a
    problem.

  * All users: the data shared for all users.  An example is the total
    number of students answering a question.  Note that this is *not* the
    same as aggregate or query data.  The same value will be shared for
    all users, and there's no way to link specific actions to specific users.

- By XBlock: fields declare how they relate to the block:

  * Block usage: the fields are related *only* to that instance, or usage, of
    the XBlock in a particular course.

  * Block definition: the fields are related to the *definition* of the XBlock.
    This definition is specified by the content creator.  A definition can be
    shared across one or more *usages*.  For instance, you could create a single
    XBlock definition with many usages, and those usages can appear across
    courses or within the same course.

  * Block type: the fields are related to the *Python type* of the XBlock, and
    thus shared across all instances of the XBlock in all courses.

  * All: the fields are related to all XBlocks, of all types (thus, any XBlock
    can access the data).

These two aspects, user and block, are independent.  A field scope specifies
both.  For example:

* A user's progress through a particular set of problems would be stored in a
  scope with User=One and XBlock=Usage.

* The content to display in an XBlock would be stored in a scope with
  User=None and XBlock=Definition.

* A user's preferences for a type of XBlock, such as the preferences for a
  circuit editor, would be stored in a scope with User=One and XBlock=Type.

* Information about the user, such as language or timezone, would be stored in
  a scope with User=One and XBlock=All.

For convenience, we also provide six predefined scopes: ``Scope.content``,
``Scope.settings``, ``Scope.user_state``, ``Scope.preferences``,
``Scope.user_info``, and ``Scope.user_state_summary``:

+---------------------------+----------------+-------------------+--------------------------+
|                           | UserScope.NONE | UserScope.ONE     | UserScope.ALL            |
+===========================+================+===================+==========================+
| **BlockScope.USAGE**      | Scope.settings | Scope.user_state  | Scope.user_state_summary |
+---------------------------+----------------+-------------------+--------------------------+
| **BlockScope.DEFINITION** | Scope.content  |                   |                          |
+---------------------------+----------------+-------------------+--------------------------+
| **BlockScope.TYPE**       |                | Scope.preferences |                          |
+---------------------------+----------------+-------------------+--------------------------+
| **BlockScope.ALL**        |                | Scope.user_info   |                          |
+---------------------------+----------------+-------------------+--------------------------+

**Examples of Situations.**  The following list shows examples of cases in which
you may want to use a particular (UserScope, BlockScope) pairing.  There are
many other situations in which you may want to use these scopes.

*BlockScope.USAGE:*

- UserScope.NONE: Settings.  You want something to be true only for a specific
  instance of an XBlock.
- UserScope.ONE: User state.  You want to keep track of questions a user has
  answered correctly and incorrectly for a specific instance of an XBlock.
- UserScope.ALL: User state summary.  You want to show how all the users respond
  to a question on a specific instance of an XBlock.

*BlockScope.DEFINITION:*

- UserScope.NONE: Content.  You have some content you would like to share across
  several courses (i.e., a table of math formulas), but it is not specific to
  any user.  All content is in this scope.
- UserScope.ONE: Multi-course user state.  You want to keep track of the user's
  state across multiple instances of this XBlock.  For instance, you could have
  a Test XBlock and then show the user their comparative scores on pre-tests and
  post-tests.
- UserScope.ALL: Multi-course user state summary.  For instance, you could show
  the average score of users on all Tests and display this average to users.

*BlockScope.TYPE:*

- UserScope.NONE: XBlock settings.  You have information related to this
  XBlock's functionality that needs to be shared across all XBlocks of its type,
  but it is not user-specific.
- UserScope.ONE: User preferences.  In courses where this XBlock is present, you
  want to keep track of what preferences the user has chosen (for instance, a
  user can set a default speed for the video player, and that default is
  accessible to all video player instances).
- UserScope.ALL: User XBlock-related interaction summary.  For instance, you
  could find the total number of users who use various features of the XBlock
  for every instance of this XBlock in all courses.

*BlockScope.ALL:*

- UserScope.NONE: Global settings.  You want to establish information that all
  XBlocks anywhere should have access to.
- UserScope.ONE: User information.  You want all XBlocks of all types to be able
  to access basic information such as user name, geographic location, language, 
  etc.
- UserScope.ALL: User demographics.  You want to establish user-related
  information that all XBlocks anywhere should have access to.  For instance, a
  count of the total number of users.

**A note about sharing data across XBlocks:** Sharing data between two blocks
that are differently typed is difficult.  In particular, the only way to share
information across differently-typed blocks (say, between a video xblock and a
problem xblock) is by putting that data in ALL.  However, putting data in ALL 
can introduce scoping and name conflict issues.  For instance, if two fields
that are both scoped to ALL have the same field name, both blocks will actually
be pointing to the same data.

For this reason, we encourage developers to be careful when deciding how to
scope their fields.

XBlocks declare their fields as class attributes in the XBlock class
definition.  Each field has at least a name, a type, and a scope::

    class ThumbsBlock(XBlock):

        upvotes = Integer(help="Number of up votes", default=0, scope=Scope.user_state_summary)
        downvotes = Integer(help="Number of down votes", default=0, scope=Scope.user_state_summary)
        voted = Boolean(help="Has this student voted?", default=False, scope=Scope.user_state)

In XBlock code, state is accessed as attributes on self. In our example above,
the data is available as ``self.upvotes``, ``self.downvotes``, and
``self.voted``.  The data is automatically scoped for the current user and
block.  Modifications to the attributes are stored in memory, and persisted to
underlying ``FieldData`` instance when ``save()`` is called on the ``XBlock``.
Runtimes will call ``save()`` after an ``XBlock`` is constructed, and after
every invocation of a handler, view, or method on an XBlock.

**Important note:** Unlike Python classes you may have worked with before, you
may not use an ``init`` method in an XBlock.  This is because XBlocks get called
in many contexts (various views and runtimes), and the ``init`` function may not
be able to do certain things depending on the scope or context in which it is
run.

If you would like to use ``init`` function for some reason, such as to implement
more complicated logic for default field values, consider one of the following
alternatives:

- Use a lazy property decorator, so that when you first access an attribute, a
  function will be called to set that attribute.
- Call the default-field-value logic in the view, instead of in ``init``.

**Important note:** At present, XBlocks does not support storing a very large
amount of data in a single field.  This is because XBlocks fields are written
and retrieved as single entities, reading the whole field into memory.  Thus, a
field that contains, say, a list of one million items would become problematic.
If you need to store very large amounts of data, a possible workaround is to
split the data across many smaller fields.


Children
--------

In contrast to the conceptual view of XBlocks, an XBlock does not refer
directly to its children. Instead, the structure of a tree of XBlocks is
maintained by the runtime, and is made available to the XBlock through a
runtime service.

This allows the runtime to store, access, and modify the structure of a course
without incurring the overhead of the XBlock code itself.  The children will
not be implicitly available.  The runtime will provide a list of child ids, and
a child can be loaded with a get_child() function call.  This means the runtime
can defer loading children until they are actually required (if ever).

.. todo::

    When editing an XBlock, it might want to modify its children. How can it do
    that?


**Accessing Children (Server-Side)**

To access children via the server-side, the best method is:

- Iterate over the XBlock's children attribute (``self.children``), which will
  yield you the usage IDs for each of the children.
- Then, to access a child block, use ``self.runtime.get_block(usage_id)`` for
  your desired usage_id.  You can then modify the child block using its
  ``.save()`` method.
- To render a given child, use ``self.runtime.render_child(usage_id)``
- To render *all* children for a given XBlock, use
  ``self.runtime.render_children``
- To ensure things render correctly, template the ``fragment.content`` into the
  parent block's html, and then use ``fragment.add_frag_resources`` (or
  ``.add_frags_resources``, in the case of rendering all children).  This will
  ensure that the javascript and CSS of child elements are included.

**Accessing Children (Client-Side)**

To access children via the client-side (via Javascript), the best method is:

- Call ``runtime.children(element)``, where ``element`` is the DOM node that
  contains the HTML representation of your XBlock's server-side view.
  (``runtime`` is automatically given to you by whichever runtime your XBlock is
  running in.)
- Similarly, you can use ``runtime.childMap(element, name)`` to get a child
  element that has a specific ``name``.
- This `client-side XBlock code`_ provides examples of these usages.

.. _client-side XBlock code: https://github.com/edx/acid-block/blob/master/acid/static/js/acid.js


Methods
-------

The behavior of an XBlock is determined by its methods, which come in a few
categories:

* Views: These are invoked by the runtime to render the XBlock. There can be
  any number of these, written as ordinary Python methods.  Each view has a
  specific name, such as "edit" or "read", specified by the runtime that will
  invoke it.

  A typical use of a view is to produce a :ref:`fragment <fragment>` for
  rendering the block as part of a web page.  The user state, settings, and
  preferences may be used to affect the output in any way the XBlock likes.
  Views can indicate what data they rely on, to aid in caching their output.

  Although views typically produce HTML-based renderings, they can be used for
  anything the runtime wants.  The runtime description of each view should be
  clear about what return type is expected and how it will be used.

* Handlers: Handlers provide server-side logic invoked by AJAX calls from the
  browser. There can be any number of these, written as ordinary Python
  methods.  Each handler has a specific name of your choice, such as "submit"
  or "preview." The runtime provides a mapping from handler names to actual
  URLs so that XBlock Javascript code can make requests to its handlers.
  Handlers can be used with GET requests as well as POST requests.

..
    * Recalculators: (not a great word!) There can be any number of these, written
      as ordinary Python methods. Each has a specific name, and is invoked by the
      runtime when a particular kind of recalculation needs to be done.  An example
      is "regrade", run when a TA needs to adjust a problem, and all the students'
      inputs should be checked again, and their grades republished.

* Methods: XBlocks have access to their children and parent, and can invoke
  methods on them simply by invoking Python methods.

Views and handlers are both inspired by web applications, but have different
uses, and therefore different designs.  Views are invoked by the runtime to
produce a rendering of some course content.  Their results are aggregated
together hierarchically, and so are not expressed as an HTTP response, but as a
structured Fragment.  Handlers are invoked by XBlock code in the browser, so
they are defined more like traditional web applications: they accept an HTTP
request, and produce an HTTP response.


Views
-----

Views are methods on the XBlock that render the block.  The runtime will invoke
a view as part of creating a webpage for part of a course.  The XBlock view
should return data in the form needed by the runtime.  Often, the result will
be a :ref:`fragment <fragment>` that the runtime can compose together into a
complete page.

Views can specify caching information to let runtimes avoid invoking the view
more frequently than needed.  TODO: Describe this.


Handlers
--------

Handlers are methods on the XBlock that process requests coming from the
browser.  Typically, they'll be used to implement ajax endpoints.  They get a
standard request object and return a standard response.  You can have as many
handlers as you like, and name them whatever you like.  Your code asks the
runtime for the URL that corresponds to your handler, and then you can use that
URL to make ajax requests.


Services
--------

XBlocks often need other services to implement full functionality.  As Python
programs, they can import whatever libraries they need.  But some services need
to be provided by the surrounding application in order to work properly as a
unified whole.  Perhaps they need to be implemented specially, or integrated
into the full application.

XBlocks can request services from their runtime to get the best integration.
TODO: finish describing the service() method.

..
    Querying
    --------

    Blocks often need access to information from other blocks in a course.  An
    exam page may want to collect information from each problem on the page, for
    example.

    TODO: Describe how that works.


    Tags
    ----

    TODO: Blocks can have tags and you can use them in querying.
