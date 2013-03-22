Symfony2 versus zwykły PHP
==========================

**Dlaczego Symfony2 jest lepszy niż otworzenie pliku i pisanie zwykłego PHP?**

Jeśli nigdy jeszcze nie używałeś frameworka PHP, nie jesteś zaznajomiony z filozofią
MVC lub po prostu zastanawiasz się dlaczego tyle rozgłosu jest wokół Symfony2, to
ten rozdział jest dla Ciebie. Zamiast wmawiać Ci, że Symfony2 pozwala tworzyć
oprogramowanie szybciej i lepiej niż przez pisanie skryptów w zwykłym PHP, pokażemy
Ci to na przykładach.

W tym rozdziale napiszemy prostą aplikację w zwykłym PHP a następnie przerobimy ją
tak, aby była lepiej zorganizowana. Będziemy podróżować w czasie, obserwując powody
dla których tworzenie stron internetowych ewaluowało na przestrzeni ostatnich lat
i gdzie jest teraz.

Na koniec zobaczysz jak Symfony2 może uwolnić Cię od prozaicznych zadań i pozwala
uzyskać kontrolę nad kodem.

Prosty blog w zwykłym PHP
-------------------------

W tym rozdziale zbudujemy symboliczną aplikacje bloga używając tylko zwykłego PHP.
Aby rozpocząć, utworzymy plik, który będzie wyświetlał wpisy bloga, które zostały
utrwalona w bazie danych. Kod napisany w zwykłym PHP jest szybki, lecz pogmatwany:

.. code-block:: html+php

    <?php
    // index.php
    $link = mysql_connect('localhost', 'myuser', 'mypassword');
    mysql_select_db('blog_db', $link);

    $result = mysql_query('SELECT id, title FROM post', $link);
    ?>

    <!DOCTYPE html>
    <html>
        <head>
            <title>List of Posts</title>
        </head>
        <body>
            <h1>List of Posts</h1>
            <ul>
                <?php while ($row = mysql_fetch_assoc($result)): ?>
                <li>
                    <a href="/show.php?id=<?php echo $row['id'] ?>">
                        <?php echo $row['title'] ?>
                    </a>
                </li>
                <?php endwhile; ?>
            </ul>
        </body>
    </html>

    <?php
    mysql_close($link);
    ?>

Jest to szybkie w napisaniu i szybkie w wykonaniu, ale co będzie, gdy aplikacja
zacznie się rozrastać – jej utrzymanie stanie się niemożliwe. Z tym kodem wiąże
się kilka problemów, które należy wziąć pod uwagę:

* **Brak sprawdzania błędów**. Co gdy połączenie z bazą danych nie uda się?

* **Zła organizacja**. Jeżeli aplikacja zacznie się rozrastać, to kod zawarty w
   jednym pliku będzie nie do utrzymania. Gdzie należy umieścić kod do obsługi
   formularzy i ich zgłoszeń? Jak można sprawdzać dane? Gdzie powinien się
   znajdować kod do wysyłania wiadomości e-mail?

* **Trudno wielokrotnie wykorzystywać fragmenty kodu**. Ponieważ wszystko jest w
   jednym pliku, to nie ma możliwości ponownego wykorzystania fragmentów kodu
   aplikacji dla innych „stron” blogu.

.. note::

     Innym problemem, tutaj nie wymienionym, jest fakt, że baza danych
     jest w tej aplikacji ograniczona do MySQL. Choć o tym tutaj nie mówimy,
     to Symfony2 w pełni integruje `Doctrine`_, bibliotekę dostarczającą
     abstakcyjną warstwę dostępu do baz danych i `mapowanie obiektowo-relacyjne`_
     (*ang. Object-Relational Mapping - ORM*).

Bierzmy się więc do pracy nad rozwiązaniem tych problemów.

Isolating the Presentation
~~~~~~~~~~~~~~~~~~~~~~~~~~

The code can immediately gain from separating the application "logic" from
the code that prepares the HTML "presentation":

.. code-block:: html+php

    <?php
    // index.php
    $link = mysql_connect('localhost', 'myuser', 'mypassword');
    mysql_select_db('blog_db', $link);

    $result = mysql_query('SELECT id, title FROM post', $link);

    $posts = array();
    while ($row = mysql_fetch_assoc($result)) {
        $posts[] = $row;
    }

    mysql_close($link);

    // include the HTML presentation code
    require 'templates/list.php';

The HTML code is now stored in a separate file (``templates/list.php``), which
is primarily an HTML file that uses a template-like PHP syntax:

.. code-block:: html+php

    <!DOCTYPE html>
    <html>
        <head>
            <title>List of Posts</title>
        </head>
        <body>
            <h1>List of Posts</h1>
            <ul>
                <?php foreach ($posts as $post): ?>
                <li>
                    <a href="/read?id=<?php echo $post['id'] ?>">
                        <?php echo $post['title'] ?>
                    </a>
                </li>
                <?php endforeach; ?>
            </ul>
        </body>
    </html>

By convention, the file that contains all of the application logic - ``index.php`` -
is known as a "controller". The term :term:`controller` is a word you'll hear
a lot, regardless of the language or framework you use. It refers simply
to the area of *your* code that processes user input and prepares the response.

In this case, the controller prepares data from the database and then includes
a template to present that data. With the controller isolated, you could
easily change *just* the template file if you needed to render the blog
entries in some other format (e.g. ``list.json.php`` for JSON format).

Isolating the Application (Domain) Logic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So far the application contains only one page. But what if a second page
needed to use the same database connection, or even the same array of blog
posts? Refactor the code so that the core behavior and data-access functions
of the application are isolated in a new file called ``model.php``:

.. code-block:: html+php

    <?php
    // model.php
    function open_database_connection()
    {
        $link = mysql_connect('localhost', 'myuser', 'mypassword');
        mysql_select_db('blog_db', $link);

        return $link;
    }

    function close_database_connection($link)
    {
        mysql_close($link);
    }

    function get_all_posts()
    {
        $link = open_database_connection();

        $result = mysql_query('SELECT id, title FROM post', $link);
        $posts = array();
        while ($row = mysql_fetch_assoc($result)) {
            $posts[] = $row;
        }
        close_database_connection($link);

        return $posts;
    }

.. tip::

   The filename ``model.php`` is used because the logic and data access of
   an application is traditionally known as the "model" layer. In a well-organized
   application, the majority of the code representing your "business logic"
   should live in the model (as opposed to living in a controller). And unlike
   in this example, only a portion (or none) of the model is actually concerned
   with accessing a database.

The controller (``index.php``) is now very simple:

.. code-block:: html+php

    <?php
    require_once 'model.php';

    $posts = get_all_posts();

    require 'templates/list.php';

Now, the sole task of the controller is to get data from the model layer of
the application (the model) and to call a template to render that data.
This is a very simple example of the model-view-controller pattern.

Isolating the Layout
~~~~~~~~~~~~~~~~~~~~

At this point, the application has been refactored into three distinct pieces
offering various advantages and the opportunity to reuse almost everything
on different pages.

The only part of the code that *can't* be reused is the page layout. Fix
that by creating a new ``layout.php`` file:

.. code-block:: html+php

    <!-- templates/layout.php -->
    <!DOCTYPE html>
    <html>
        <head>
            <title><?php echo $title ?></title>
        </head>
        <body>
            <?php echo $content ?>
        </body>
    </html>

The template (``templates/list.php``) can now be simplified to "extend"
the layout:

.. code-block:: html+php

    <?php $title = 'List of Posts' ?>

    <?php ob_start() ?>
        <h1>List of Posts</h1>
        <ul>
            <?php foreach ($posts as $post): ?>
            <li>
                <a href="/read?id=<?php echo $post['id'] ?>">
                    <?php echo $post['title'] ?>
                </a>
            </li>
            <?php endforeach; ?>
        </ul>
    <?php $content = ob_get_clean() ?>

    <?php include 'layout.php' ?>

You've now introduced a methodology that allows for the reuse of the
layout. Unfortunately, to accomplish this, you're forced to use a few ugly
PHP functions (``ob_start()``, ``ob_get_clean()``) in the template. Symfony2
uses a ``Templating`` component that allows this to be accomplished cleanly
and easily. You'll see it in action shortly.

Adding a Blog "show" Page
-------------------------

The blog "list" page has now been refactored so that the code is better-organized
and reusable. To prove it, add a blog "show" page, which displays an individual
blog post identified by an ``id`` query parameter.

To begin, create a new function in the ``model.php`` file that retrieves
an individual blog result based on a given id::

    // model.php
    function get_post_by_id($id)
    {
        $link = open_database_connection();

        $id = intval($id);
        $query = 'SELECT date, title, body FROM post WHERE id = '.$id;
        $result = mysql_query($query);
        $row = mysql_fetch_assoc($result);

        close_database_connection($link);

        return $row;
    }

Next, create a new file called ``show.php`` - the controller for this new
page:

.. code-block:: html+php

    <?php
    require_once 'model.php';

    $post = get_post_by_id($_GET['id']);

    require 'templates/show.php';

Finally, create the new template file - ``templates/show.php`` - to render
the individual blog post:

.. code-block:: html+php

    <?php $title = $post['title'] ?>

    <?php ob_start() ?>
        <h1><?php echo $post['title'] ?></h1>

        <div class="date"><?php echo $post['date'] ?></div>
        <div class="body">
            <?php echo $post['body'] ?>
        </div>
    <?php $content = ob_get_clean() ?>

    <?php include 'layout.php' ?>

Creating the second page is now very easy and no code is duplicated. Still,
this page introduces even more lingering problems that a framework can solve
for you. For example, a missing or invalid ``id`` query parameter will cause
the page to crash. It would be better if this caused a 404 page to be rendered,
but this can't really be done easily yet. Worse, had you forgotten to clean
the ``id`` parameter via the ``intval()`` function, your
entire database would be at risk for an SQL injection attack.

Another major problem is that each individual controller file must include
the ``model.php`` file. What if each controller file suddenly needed to include
an additional file or perform some other global task (e.g. enforce security)?
As it stands now, that code would need to be added to every controller file.
If you forget to include something in one file, hopefully it doesn't relate
to security...

A "Front Controller" to the Rescue
----------------------------------

The solution is to use a :term:`front controller`: a single PHP file through
which *all* requests are processed. With a front controller, the URIs for the
application change slightly, but start to become more flexible:

.. code-block:: text

    Without a front controller
    /index.php          => Blog post list page (index.php executed)
    /show.php           => Blog post show page (show.php executed)

    With index.php as the front controller
    /index.php          => Blog post list page (index.php executed)
    /index.php/show     => Blog post show page (index.php executed)

.. tip::
    The ``index.php`` portion of the URI can be removed if using Apache
    rewrite rules (or equivalent). In that case, the resulting URI of the
    blog show page would be simply ``/show``.

When using a front controller, a single PHP file (``index.php`` in this case)
renders *every* request. For the blog post show page, ``/index.php/show`` will
actually execute the ``index.php`` file, which is now responsible for routing
requests internally based on the full URI. As you'll see, a front controller
is a very powerful tool.

Creating the Front Controller
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You're about to take a **big** step with the application. With one file handling
all requests, you can centralize things such as security handling, configuration
loading, and routing. In this application, ``index.php`` must now be smart
enough to render the blog post list page *or* the blog post show page based
on the requested URI:

.. code-block:: html+php

    <?php
    // index.php

    // load and initialize any global libraries
    require_once 'model.php';
    require_once 'controllers.php';

    // route the request internally
    $uri = $_SERVER['REQUEST_URI'];
    if ('/index.php' == $uri) {
        list_action();
    } elseif ('/index.php/show' == $uri && isset($_GET['id'])) {
        show_action($_GET['id']);
    } else {
        header('Status: 404 Not Found');
        echo '<html><body><h1>Page Not Found</h1></body></html>';
    }

For organization, both controllers (formerly ``index.php`` and ``show.php``)
are now PHP functions and each has been moved into a separate file, ``controllers.php``:

.. code-block:: php

    function list_action()
    {
        $posts = get_all_posts();
        require 'templates/list.php';
    }

    function show_action($id)
    {
        $post = get_post_by_id($id);
        require 'templates/show.php';
    }

As a front controller, ``index.php`` has taken on an entirely new role, one
that includes loading the core libraries and routing the application so that
one of the two controllers (the ``list_action()`` and ``show_action()``
functions) is called. In reality, the front controller is beginning to look and
act a lot like Symfony2's mechanism for handling and routing requests.

.. tip::

   Another advantage of a front controller is flexible URLs. Notice that
   the URL to the blog post show page could be changed from ``/show`` to ``/read``
   by changing code in only one location. Before, an entire file needed to
   be renamed. In Symfony2, URLs are even more flexible.

By now, the application has evolved from a single PHP file into a structure
that is organized and allows for code reuse. You should be happier, but far
from satisfied. For example, the "routing" system is fickle, and wouldn't
recognize that the list page (``/index.php``) should be accessible also via ``/``
(if Apache rewrite rules were added). Also, instead of developing the blog,
a lot of time is being spent working on the "architecture" of the code (e.g.
routing, calling controllers, templates, etc.). More time will need to be
spent to handle form submissions, input validation, logging and security.
Why should you have to reinvent solutions to all these routine problems?

Add a Touch of Symfony2
~~~~~~~~~~~~~~~~~~~~~~~

Symfony2 to the rescue. Before actually using Symfony2, you need to download
it. This can be done by using Composer, which takes care of downloading the
correct version and all its dependencies and provides an autoloader. An 
autoloader is a tool that makes it possible to start using PHP classes 
without explicitly including the file containing the class.

In your root directory, create a ``composer.json`` file with the following
content:

.. code-block:: json

    {
        "require": {
            "symfony/symfony": "2.2.*"
        },
        "autoload": {
            "files": ["model.php","controllers.php"]
        }
    }
    
Next, `download Composer`_ and then run the following command, which will download Symfony
into a vendor/ directory:

.. code-block:: bash

    $ php composer.phar install

Beside downloading your dependencies, Composer generates a ``vendor/autoload.php`` file,
which takes care of autoloading for all the files in the Symfony Framework as well as 
the files mentioned in the autoload section of your ``composer.json``.

Core to Symfony's philosophy is the idea that an application's main job is
to interpret each request and return a response. To this end, Symfony2 provides
both a :class:`Symfony\\Component\\HttpFoundation\\Request` and a
:class:`Symfony\\Component\\HttpFoundation\\Response` class. These classes are
object-oriented representations of the raw HTTP request being processed and
the HTTP response being returned. Use them to improve the blog:

.. code-block:: html+php

    <?php
    // index.php
    require_once 'vendor/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $uri = $request->getPathInfo();
    if ('/' == $uri) {
        $response = list_action();
    } elseif ('/show' == $uri && $request->query->has('id')) {
        $response = show_action($request->query->get('id'));
    } else {
        $html = '<html><body><h1>Page Not Found</h1></body></html>';
        $response = new Response($html, 404);
    }

    // echo the headers and send the response
    $response->send();

The controllers are now responsible for returning a ``Response`` object.
To make this easier, you can add a new ``render_template()`` function, which,
incidentally, acts quite a bit like the Symfony2 templating engine:

.. code-block:: php

    // controllers.php
    use Symfony\Component\HttpFoundation\Response;

    function list_action()
    {
        $posts = get_all_posts();
        $html = render_template('templates/list.php', array('posts' => $posts));

        return new Response($html);
    }

    function show_action($id)
    {
        $post = get_post_by_id($id);
        $html = render_template('templates/show.php', array('post' => $post));

        return new Response($html);
    }

    // helper function to render templates
    function render_template($path, array $args)
    {
        extract($args);
        ob_start();
        require $path;
        $html = ob_get_clean();

        return $html;
    }

By bringing in a small part of Symfony2, the application is more flexible and
reliable. The ``Request`` provides a dependable way to access information
about the HTTP request. Specifically, the ``getPathInfo()`` method returns
a cleaned URI (always returning ``/show`` and never ``/index.php/show``).
So, even if the user goes to ``/index.php/show``, the application is intelligent
enough to route the request through ``show_action()``.

The ``Response`` object gives flexibility when constructing the HTTP response,
allowing HTTP headers and content to be added via an object-oriented interface.
And while the responses in this application are simple, this flexibility
will pay dividends as your application grows.

The Sample Application in Symfony2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The blog has come a *long* way, but it still contains a lot of code for such
a simple application. Along the way, you've made a simple routing
system and a method using ``ob_start()`` and ``ob_get_clean()`` to render
templates. If, for some reason, you needed to continue building this "framework"
from scratch, you could at least use Symfony's standalone `Routing`_ and
`Templating`_ components, which already solve these problems.

Instead of re-solving common problems, you can let Symfony2 take care of
them for you. Here's the same sample application, now built in Symfony2::

    // src/Acme/BlogBundle/Controller/BlogController.php
    namespace Acme\BlogBundle\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\Controller;

    class BlogController extends Controller
    {
        public function listAction()
        {
            $posts = $this->get('doctrine')->getManager()
                ->createQuery('SELECT p FROM AcmeBlogBundle:Post p')
                ->execute();

            return $this->render(
                'AcmeBlogBundle:Blog:list.html.php',
                array('posts' => $posts)
            );
        }

        public function showAction($id)
        {
            $post = $this->get('doctrine')
                ->getManager()
                ->getRepository('AcmeBlogBundle:Post')
                ->find($id)
            ;

            if (!$post) {
                // cause the 404 page not found to be displayed
                throw $this->createNotFoundException();
            }

            return $this->render(
                'AcmeBlogBundle:Blog:show.html.php',
                array('post' => $post)
            );
        }
    }

The two controllers are still lightweight. Each uses the :doc:`Doctrine ORM library</book/doctrine>`
to retrieve objects from the database and the ``Templating`` component to
render a template and return a ``Response`` object. The list template is
now quite a bit simpler:

.. code-block:: html+php

    <!-- src/Acme/BlogBundle/Resources/views/Blog/list.html.php -->
    <?php $view->extend('::layout.html.php') ?>

    <?php $view['slots']->set('title', 'List of Posts') ?>

    <h1>List of Posts</h1>
    <ul>
        <?php foreach ($posts as $post): ?>
        <li>
            <a href="<?php echo $view['router']->generate(
                'blog_show',
                array('id' => $post->getId())
            ) ?>">
                <?php echo $post->getTitle() ?>
            </a>
        </li>
        <?php endforeach; ?>
    </ul>

The layout is nearly identical:

.. code-block:: html+php

    <!-- app/Resources/views/layout.html.php -->
    <!DOCTYPE html>
    <html>
        <head>
            <title><?php echo $view['slots']->output(
                'title',
                'Default title'
            ) ?></title>
        </head>
        <body>
            <?php echo $view['slots']->output('_content') ?>
        </body>
    </html>

.. note::

    The show template is left as an exercise, as it should be trivial to
    create based on the list template.

When Symfony2's engine (called the ``Kernel``) boots up, it needs a map so
that it knows which controllers to execute based on the request information.
A routing configuration map provides this information in a readable format:

.. code-block:: yaml

    # app/config/routing.yml
    blog_list:
        path:     /blog
        defaults: { _controller: AcmeBlogBundle:Blog:list }

    blog_show:
        path:     /blog/show/{id}
        defaults: { _controller: AcmeBlogBundle:Blog:show }

Now that Symfony2 is handling all the mundane tasks, the front controller
is dead simple. And since it does so little, you'll never have to touch
it once it's created (and if you use a Symfony2 distribution, you won't
even need to create it!)::

    // web/app.php
    require_once __DIR__.'/../app/bootstrap.php';
    require_once __DIR__.'/../app/AppKernel.php';

    use Symfony\Component\HttpFoundation\Request;

    $kernel = new AppKernel('prod', false);
    $kernel->handle(Request::createFromGlobals())->send();

The front controller's only job is to initialize Symfony2's engine (``Kernel``)
and pass it a ``Request`` object to handle. Symfony2's core then uses the
routing map to determine which controller to call. Just like before, the
controller method is responsible for returning the final ``Response`` object.
There's really not much else to it.

For a visual representation of how Symfony2 handles each request, see the
:ref:`request flow diagram<request-flow-figure>`.

Where Symfony2 Delivers
~~~~~~~~~~~~~~~~~~~~~~~

In the upcoming chapters, you'll learn more about how each piece of Symfony
works and the recommended organization of a project. For now, let's see how
migrating the blog from flat PHP to Symfony2 has improved life:

* Your application now has **clear and consistently organized code** (though
  Symfony doesn't force you into this). This promotes **reusability** and
  allows for new developers to be productive in your project more quickly;

* 100% of the code you write is for *your* application. You **don't need
  to develop or maintain low-level utilities** such as :ref:`autoloading<autoloading-introduction-sidebar>`,
  :doc:`routing</book/routing>`, or rendering :doc:`controllers</book/controller>`;

* Symfony2 gives you **access to open source tools** such as Doctrine and the
  Templating, Security, Form, Validation and Translation components (to name
  a few);

* The application now enjoys **fully-flexible URLs** thanks to the ``Routing``
  component;

* Symfony2's HTTP-centric architecture gives you access to powerful tools
  such as **HTTP caching** powered by **Symfony2's internal HTTP cache** or
  more powerful tools such as `Varnish`_. This is covered in a later chapter
  all about :doc:`caching</book/http_cache>`.

And perhaps best of all, by using Symfony2, you now have access to a whole
set of **high-quality open source tools developed by the Symfony2 community**!
A good selection of Symfony2 community tools can be found on `KnpBundles.com`_.

Better templates
----------------

If you choose to use it, Symfony2 comes standard with a templating engine
called `Twig`_ that makes templates faster to write and easier to read.
It means that the sample application could contain even less code! Take,
for example, the list template written in Twig:

.. code-block:: html+jinja

    {# src/Acme/BlogBundle/Resources/views/Blog/list.html.twig #}
    {% extends "::layout.html.twig" %}

    {% block title %}List of Posts{% endblock %}

    {% block body %}
        <h1>List of Posts</h1>
        <ul>
            {% for post in posts %}
            <li>
                <a href="{{ path('blog_show', {'id': post.id}) }}">
                    {{ post.title }}
                </a>
            </li>
            {% endfor %}
        </ul>
    {% endblock %}

The corresponding ``layout.html.twig`` template is also easier to write:

.. code-block:: html+jinja

    {# app/Resources/views/layout.html.twig #}
    <!DOCTYPE html>
    <html>
        <head>
            <title>{% block title %}Default title{% endblock %}</title>
        </head>
        <body>
            {% block body %}{% endblock %}
        </body>
    </html>

Twig is well-supported in Symfony2. And while PHP templates will always
be supported in Symfony2, the many advantages of Twig will continue to
be discussed. For more information, see the :doc:`templating chapter</book/templating>`.

Learn more from the Cookbook
----------------------------

* :doc:`/cookbook/templating/PHP`
* :doc:`/cookbook/controller/service`

.. _`Doctrine`: http://www.doctrine-project.org
.. _`download Composer`: http://getcomposer.org/download/
.. _`Routing`: https://github.com/symfony/Routing
.. _`Templating`: https://github.com/symfony/Templating
.. _`KnpBundles.com`: http://knpbundles.com/
.. _`Twig`: http://twig.sensiolabs.org
.. _`Varnish`: https://www.varnish-cache.org/
.. _`PHPUnit`: http://www.phpunit.de
.. _`Doctrine`: http://www.doctrine-project.org/
.. _`mapowanie obiektowo-relacyjne`: http://pl.wikipedia.org/wiki/Mapowanie_obiektowo-relacyjne