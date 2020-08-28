# docker-cloudsqlproxy

Google [Cloud SQL](https://cloud.google.com/sql) proxy service.

This repo creates a wrapper (entrypoint) script to be used in the Google Cloud SQL [proxy container](https://cloud.google.com/sql/docs/mysql/connect-docker). The `entrypoint.sh` script autoruns when you run the docker container created using the Dockerfile in this repo.  All the settings for the Cloud SQL proxy are provided using environment variables - enabling a much simplier configuration.

## Contents

1. [Authentication](#authentication)
2. [Connection settings](#connection-settings)
    1. [Connection string list environment variable](#1-connection-string-list-environment-variable)
    1. [Individual settings variables](#2-individual-settings-variables)
3. [Additional settings](#additional-settings)
4. [Example usage](#example-usage)

## Authentication

The script assumes one of three forms of **service account authentication**:

1) A **service account json file** that is **mounted into the docker container** either under the default path (`/etc/sqlproxy-service-account.json`) or under the path specified by the environment variable `CLOUDSQL_CREDENTIAL_FILE`.

2) A **service account json string** provided in the `CLOUDSQL_CREDENTIALS` environment variable. The credentials are saved to the same path as the paramenter above (either the default `/etc/sqlproxy-service-account.json` or the path specified by `CLOUDSQL_CREDENTIAL_FILE`). Thus, any credentials provided by in the file specified by `CLOUDSQL_CREDENTIAL_FILE` will be overwritten by the credentials provided in the `CLOUDSQL_CREDENTIALS` variable.

3) Using the "application" **default service account**. This is either the service account used to create the GCE compute instance the docker is running on **or** what ever the current user is authorized as if they were running it on a non-GCE host.

> **Remember:** the service account used to create the proxy must have a role that includes the `cloudsql.instances.connect` permission. The predefined Cloud SQL roles that include this permission are: `Cloud SQL Client`, `Cloud SQL Editor` and `Cloud SQL Admin`.

## Connection settings

There are also two methods of **setting the connection string** used by the proxy.

### 1) Connection string list environment variable

Specify an explicit comma-separated list of one or more database _connection strings_ in the environment variable `CLOUDSQL_CONNECTION_LIST`. The list **must** contain **at least one** connection string in the following format:

```text
CLOUDSQL_INSTANCE_CONNECTION_NAME=0.0.0.0:PORT
```

which is equivalent to:

```text
GOOGLE_PROJECT:CLOUDSQL_ZONE:CLOUDSQL_INSTANCE=0.0.0.0:PORT
```

where `INSTANCE_CONNECTION_NAME` is the instances connection name, which can be retrieved from the [Cloud SQL Console](https://console.cloud.google.com/sql/instances), `GOOGLE_PROJECT` is Google Cloud project where the Cloud SQL instance resides, `CLOUDSQL_ZONE` is the instance's zone, `CLOUDSQL_INSTANCE` is the instance's ID name, and `PORT` is the TCP port number that the Cloud SQL proxy will open for connections to the instance.

> **Note:** the port set by the `PORT` environment variable is _inside_ the docker container. To expose the service on a port on the host machine, the `publish` option must be used with the [`docker run`](https://docs.docker.com/engine/reference/commandline/run) command. For example `docker run --env PORT=$PORT -p 127.0.0.1:$HOST_PORT:$PORT ...`, where `$PORT` contains the container port number and `$HOST_PORT` contains the host port.

### 2) Individual settings variables

By specifying all of the environment variables bellow. This method supports only a single Cloud SQL instance.

1. `GOOGLE_PROJECT`: the Google project where the instance resides.
2. `CLOUDSQL_ZONE`: the instance's zone.
3. `CLOUDSQL_INSTANCE`: the instance's ID name.
4. `PORT`: the TCP port number that the Cloud SQL proxy will open for connections to the instance.

## Additional settings

1. `CLOUDSQL_MAXCONNS`: the maximum number of database connections the proxy will support. The default is unlimited.
2. `CLOUDSQL_LOGGING`: logging level. The default is verbose.

## Example usage

To build the image:

```bash
docker build . -t cloud-sql-proxy
```

To start the proxy:

```bash
docker run --env-file=.env -p 127.0.0.1:5432:5432 cloud-sql-proxy
```

where `.env` contains the configuration variables specified in the sections above. For example:

```env
CLOUDSQL_CREDENTIALS={"type":"service_account", ...}
GOOGLE_PROJECT=my_project
CLOUDSQL_ZONE=us-east1
CLOUDSQL_INSTANCE=my_instance_name
PORT=5432
```

It might be more confortable to run the proxy as a detached container (`-d` flag):

```bash
docker run --env-file=.env -p 127.0.0.1:5432:5432 -d cloud-sql-proxy
```
