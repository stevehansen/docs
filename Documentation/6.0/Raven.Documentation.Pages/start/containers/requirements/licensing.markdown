# Licensing Requirements for RavenDB in Containers

When running your database in a containerized or orchestrated environment, ensuring **RavenDB can access its license** is a necessary but straightforward step. Compared to more complex matters like managing persistent storage, securing networking, or optimizing system performance, licensing is one of the simplest tasks to configure.

For developers and DevOps teams, this involves making the license accessible to the database using methods such as configuration settings, environment variables, or secure solutions like secret vaults. The key is to provide the license to the server while maintaining its security.
Common problems include passing the JSON license through to the container - **escaping, quotes, slashes**.
You need to ensure that.

##### Simplify License Formatting with jq
To ensure the license is properly formatted, use the following command:
```
jq -c '.' license.json > formatted-license.json
```

This command reads the license.json file, formats it into a single-line JSON string suitable for environment variables or configuration files, and saves it to formatted-license.json.

------

In most cases, using an environment variable is sufficient, especially in isolated or development environments. For production or sensitive setups, a secret vault can further enhance security. Regardless of the method, the container must have consistent access to the license.

For **detailed guidance** on licensing RavenDB in containerized environments, including examples, **refer to our documentation**: [Licensing RavenDB in Docker](../../licensing/license-under-docker).
