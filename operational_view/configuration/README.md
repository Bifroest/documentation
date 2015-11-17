
# Configuring the Bifroest Service

Bifroest Services use a JSON object for their configuration. This
JSON Object is acquired in the following manner:

 - Start with the `configuration path` given in the command line arguments.
 - Traverse the directory tree, following symlinks and collect all files
   ending in `.conf`.
 - Parse each file as JSON, while *ignoring files not containing a single
   valid JSON object*. If the log of the service complains about missing
   configuration, check if the relevant files were ignored.
 - Merge all of these json objects according to the following rules:
     - If we merge two JSON objects:
        - Create an empty JSON Object
        - Add all key-value pairs contained in only one JSON object to
          the new object.
        - Merge all conflicting key-value pairs recursively and add the
          the result to the new object.
     - If we merge two JSON arrays, append the arrays in any order.
     - Otherwise, we have a merge conflict. *The service won't start in this
       case*.

This config merging allows us to split up the configuration into managable 
chunks and arrange them freely on the filesystem.

At the moment, bifroest configurations follow a similar structure: Toplevel 
elements of the configuration object correspond to subsystems. These subsystems
are rougly documented in the commons project. Thus, if this documentation talks
about changing the configuration of the `statistics-system`, or the 
`multi-server-system`, it's a good idea to look for a toplevel key "statistics" or
"multi-server" in the configuration.
