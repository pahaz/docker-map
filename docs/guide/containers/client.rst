.. _container_client:

Enhanced client functionality
=============================
The library comes with an enhanced client for some added functionality. Docker-Map is relying on that for managing
container creation and startup. One part of the client is :class:`~dockermap.map.base.DockerClientWrapper`, a wrapper
around `docker-py`'s client; another is the application of :class:`~dockermap.map.container.ContainerMap` instances to
this client, which is handled by :class:`~dockermap.map.client.MappingDockerClient`.

Wrapped functionality
---------------------
In a few methods, the original arguments and behavior of `docker-py` has been modified in
:class:`~dockermap.map.base.DockerClientWrapper`:

Building images
^^^^^^^^^^^^^^^
On the build method :meth:`~dockermap.map.base.DockerClientWrapper.build`, it is mandatory to give the new image a
name (short example in :ref:`build_image_run`). Optionally ``add_latest_tag`` can be set to ``True`` for tagging the
image additionally with `latest`. Whereas `docker-py` returns a stream, the wrapped method sends that stream to a log
(see :ref:`client-logging`) and returns the new image id, if the build has been
successful. Unsuccessful builds return ``None``.

Registry access
^^^^^^^^^^^^^^^
A login to a registry server with :meth:`~dockermap.map.base.DockerClientWrapper.login` only returns ``True``, if it
has been successful, or ``False`` otherwise. Registry :meth:`~dockermap.map.base.DockerClientWrapper.pull` and
:meth:`~dockermap.map.base.DockerClientWrapper.push` actions process the stream output using
:meth:`~dockermap.map.base.DockerClientWrapper.push_log`; they return ``True`` or ``False`` depending on whether the
operation succeeded.

Added functionality
-------------------
The following methods are not part of the original `docker-py` implementation:

.. _client-logging:

Logging
^^^^^^^
Feedback from the service is processed with :meth:`~dockermap.map.base.DockerClientWrapper.push_log`. The
implementation writes to `stdout`, but this may easily be changed in a subclass. Logs for a running container can be
shown with :meth:`~dockermap.map.base.DockerClientWrapper.push_container_logs`. Each message is prefixed with the
container name.

.. NOTE::
   This implementation structure is likely to be changed in the future, in order to provide better support for different
   logging needs (e.g. using the default logger or showing progress bars).

Building from DockerFile and DockerContext
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
In order to build files directly from :class:`~dockermap.build.dockerfile.DockerFile` and
:class:`~dockermap.build.context.DockerContext` instances,
:meth:`~dockermap.map.base.DockerClientWrapper.build_from_file` and
:meth:`~dockermap.map.base.DockerClientWrapper.build_from_context` are available. For details, see
:ref:`build_images`.

Managing images and containers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
On development machines, containers often have to be stopped, removed, and restarted. Furthermore, when repeatedly
building images, there may be a lot of unused images around.

Calling :meth:`~dockermap.map.base.DockerClientWrapper.cleanup_containers` removes all stopped containers from the
remote host. Containers that have never been started are not deleted.
:meth:`~dockermap.map.base.DockerClientWrapper.remove_all_containers` stops and removes all containers on the remote.
Use this with care outside of the development environment.

For removing images without names and tags (i.e. that show up as `none`), use
:meth:`~dockermap.map.base.DockerClientWrapper.cleanup_images`. Optionally, setting ``remove_old`` to ``True``
additionally removes images that do have names and tags, but not one with `latest`::

    client.cleanup_images(remove_old=True)

All current container names are available through :meth:`~dockermap.map.base.DockerClientWrapper.get_container_names`,
for checking if they exist. Similarly :meth:`~dockermap.map.base.DockerClientWrapper.get_image_tags` returns all
named images, but in form of a dictionary with a name-id assignment.

Storing images and resources
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The original implementations of ``copy`` (copying a resource from a container) and ``get_image`` (retrieving an image
as a tarball) are available directly, but they return a stream. Implementations of
:meth:`~dockermap.map.base.DockerClientWrapper.copy_resource` and
:meth:`~dockermap.map.base.DockerClientWrapper.save_image` allow for writing the data directly to a local file.
However, this has turned out to be very slow and may not be practical.


Applying container maps
-----------------------
This section provides some background information of the client functionality. The configuration and an example is
further described in :ref:`container_maps`.

Instances of :class:`~dockermap.map.client.MappingDockerClient` are usually created with a map and a client.
The former is an instance of :class:`~dockermap.map.container.ContainerMap`, the latter is
a :class:`~dockermap.map.base.DockerClientWrapper` object. Both initializing arguments are however optional and may be
changed any time later using the properties :attr:`~dockermap.map.client.MappingDockerClient.map` and
:attr:`~dockermap.map.client.MappingDockerClient.client`::

    map_client = MappingDockerClient(container_map, DockerClientWrapper('unix://var/run/docker.sock'))

:class:`~dockermap.map.client.MappingDockerClient` contains most functions used within a container lifecycle, but
additionally resolves dependencies from the map and applies the resulting parameters to the creation and start.
Calling :meth:`~dockermap.map.client.MappingDockerClient.create`
resolves all dependency containers to be created prior to the current one. First, `attached` volumes are created (see
:ref:`attached-volumes`) of the dependency containers. Then the client creates dependency containers and the requested
container. This functionality can be overridden by setting ``autocreate_dependencies`` and ``autocreate_attached`` to
``False``.

Similarly, :meth:`~dockermap.map.client.MappingDockerClient.start` first launches dependency containers' `attached`
volumes, then dependencies themselves, and finally the requested container; the behavior can also be changed with
``autostart_dependencies``.

In order to see what defines a dependency, see :ref:`shared-volumes-containers` and :ref:`linked-containers`.

Additional keyword arguments to the ``start`` and ``create`` methods of the client are passed through; the order of
precedence towards the :class:`~dockermap.map.config.ContainerConfiguration` is further detailed in
:ref:`additional-options`. Example::

    map_client.start('web_server', restart_policy={'MaximumRetryCount': 0, 'Name': 'always'})

.. note:: Both :meth:`~dockermap.map.client.MappingDockerClient.create` and
          :meth:`~dockermap.map.client.MappingDockerClient.start` in their current implementation will always re-use
          existing containers with the same name. This may be changed to a more sophisticated evaluation in future
          implementations, as partial re-creation of dependency containers with shared volumes may lead to the
          :meth:`~dockermap.map.client.MappingDockerClient.start` method referring to wrong container instances.

:meth:`~dockermap.map.client.MappingDockerClient.stop` stops a container and its dependencies, i.e. containers
that have been started thereafter. The dependency resolution is once again optional and may be deactivated by setting
``autostop_dependent=False``. Removing containers with :meth:`~dockermap.map.client.MappingDockerClient.remove`
does not resolve dependencies, but only removes the specified container. Like in the `docker-py` and command line
client, it only works on stopped containers.

The method :meth:`~dockermap.map.client.MappingDockerClient.wait`, in addition to the original `wait` implementation,
only provides additional (and optional) logging, and prefixes the given container name with the name of the map.
:meth:`~dockermap.map.client.MappingDockerClient.wait_and_remove` removes the container after is has finished running.

For limiting effects to particular :ref:`instances` of a container configuration,
:meth:`~dockermap.map.client.MappingDockerClient.create`,
:meth:`~dockermap.map.client.MappingDockerClient.start`,
:meth:`~dockermap.map.client.MappingDockerClient.stop`, and
:meth:`~dockermap.map.client.MappingDockerClient.remove`, accept an ``instances`` argument, where one or multiple
instance names can be specified. Similarly, :meth:`~dockermap.map.client.MappingDockerClient.wait`
and :meth:`~dockermap.map.client.MappingDockerClient.wait_and_remove` allow for
specifying a single ``instance`` name.

Note that :class:`~dockermap.map.client.MappingDockerClient` caches names of existing containers and images for
speeding up operations. The cache is flushed automatically when the :attr:`~dockermap.map.base.client` property
is set. However, when changes (e.g. creating or removing containers) are made directly, the name cache should be
reset with :meth:`~dockermap.map.client.MappingDockerClient.refresh_names`.