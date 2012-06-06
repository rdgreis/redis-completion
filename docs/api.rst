.. _api:

API
===

.. py:module:: redis_completion.engine

.. py:class:: RedisEngine(gram_lengths=None, min_length=3, prefix='ac', stop_words=None, terminator='^', **conn_kwargs)

    :param gram_lengths: a 2-tuple containing the min and max lengths of n-grams
        to generate.  The default is (2, 3), meaning that the phrase "Autocomplete with redis"
        would become ``[autocomplete with], [autocomplete with redis], [with redis], [redis]``
    :param min_length: the minimum length a phrase has to be to return meaningful
        search results
    :param prefix: a prefix used for all keys stored in Redis to allow multiple
        "indexes" to exist and to make deletion easier.
    :param stop_words: a ``set()`` of stop words to remove from index/search data
    :param terminator: a character used internally to represent that a given ``ZSET``
        matches a phrase in the index
    :param conn_kwargs: any named parameters that should be used when connecting
        to Redis, e.g. ``host='localhost', port=6379``

    :py:class:`RedisEngine` is responsible for storing and searching data suitable
    for autocompletion.  There are many different options you can use to configure
    how autocomplete behaves, but the defaults are intended to provide good general
    performance for searching things like blog post titles and the like.

    The underlying data structure used to provide autocompletion is a sorted set,
    details are described `in this post <http://antirez.com/post/autocomplete-with-redis.html>`_.

    .. py:method:: store(obj_id[, title=None[, data=None]])

        :param obj_id: a unique identifier for the object
        :param title: a string to store in the index and allow autocompletion on,
            which, if not provided defaults to the given ``obj_id``
        :param data: any data you wish to store and return when a given title is
            searched for.  If not provided, defaults to the given ``title`` (or ``obj_id``)

        Store an object in the index and allow it to be searched for.

        Examples:

        .. code-block:: python

            engine.store('some phrase')
            engine.store('some other phrase')

        In the following example a list of blog entries is being stored in the
        index.  Note that arbitrary data can be stored in the index.  When a search
        is performed this data will be returned.

        .. code-block:: python

            for entry in Entry.select():
                engine.store(entry.id, entry.title, json.dumps({
                    'id': entry.id,
                    'published': entry.published,
                    'title': entry.title,
                    'url': entry.url,
                })

    .. py:method:: store_json(obj_id, title, data)

        Like :py:meth:`store` except ``data`` is automatically serialized as JSON
        before being stored in the index.  Best when used in conjunction with
        :py:meth:`search_json`.

    .. py:method:: remove(obj_id)

        :param obj_id: a unique identifier for the object

        Removes the given object from the index.

    .. py:method:: search(phrase[, limit=None[, filters=None[, mappers=None]]])

        :param phrase: search the index for the given phrase
        :param limit: an integer indicating the number of results to limit the
            search to.
        :param filters: a list of callables which will be used to filter the search
            results before returning to the user.  Filters should take a single parameter,
            the ``data`` associated with a given object and should return either
            ``True`` or ``False``.  A ``False`` value returned by any of the filters
            will prevent a result from being returned.
        :param mappers: a list of callables which will be used to transform the
            raw data returned from the index.
        :rtype: A list containing data returned by the index

        .. note:: Mappers act upon data before it is passed to the filters

        Assume we have stored some interesting blog posts, encoding some metadata
        using JSON:

        .. code-block:: python

            >>> engine.search('python', mappers=[json.loads])
            [{'published': True, 'title': 'an entry about python', 'url': '/blog/1/'},
             {'published': False, 'title': 'using redis with python', 'url': '/blog/3/'}]

    .. py:method:: search(phrase[, limit=None[, filters=None[, mappers=None]]])

        Like :py:meth:`search` except ``json.loads`` is inserted as the very first
        mapper.  Best when used in conjunction with :py:meth:`store_json`.