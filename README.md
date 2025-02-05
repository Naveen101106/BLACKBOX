# BLACKBOX
This Flask application manages Prometheus Blackbox Exporter configurations by creating and deleting YAML configuration files in the etc directory. The /endpoint POST route allows users to add new configurations by specifying a name and url, ensuring no duplicates exist. The /endpoint/delete DELETE route removes configurations based on name.
