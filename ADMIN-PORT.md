ADMIN PORT
==========
Reference the issue https://github.com/valkey-io/valkey/issues/497 (add a management-port) and https://github.com/valkey-io/valkey/issues/469 (Trusted/un-trusted client feature)

Background: In some cases, we can run valkey server on a container and multiply containers could be run in a physical machine simultaneously.
            Adminstator wants to execute some special commands in the server, and general clients are not allowed to run these kinds of commands.
            Thus, we want to set an admin port for adminstator connection, and general clients could still connect with server with port argument.

            Although Valkey invloved ACL concept, but we need to set user first. But for this case, we have no idea the admin clients name and general 
            clients name as well. Thus The better solution is that let admin clients and general clients connect to 2 different port of server.

            We can introduce another parameter in valkey.conf:  admin-port
