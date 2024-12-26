# XPipe Python API

[![GitHub license](https://img.shields.io/github/license/xpipe-io/xpipe-python-api.svg)](https://github.com/xpipe-io/xpipe-python-api/blob/master/LICENSE)
[![PyPI version](https://img.shields.io/pypi/v/xpipe_api)](https://pypi.org/project/xpipe_api/)

Python client for the XPipe API. This library is a wrapper for the raw [HTTP API](https://github.com/xpipe-io/xpipe/blob/master/openapi.yaml) and intended to make working with it more convenient.

## Installation
```
python3 -m pip install xpipe_api
```

## Usage

```python
from xpipe_api import Client

# By default, Client() will read an access key from the file xpipe_auth on the local filesystem
# and talk to the XPipe HTTP server on localhost.  To connect to a remote instance with an API
# key, use Client(token="foo", base_url = "http://servername:21721")
client = Client()

# connection_query accepts glob-based filters on the category, connection name, and connection type
all_connections = client.connection_query()

# Each connection is just a UUID
first_connection_uuid = all_connections[0]

# Before any shell commands can be run, a shell session must be started on a connection
client.shell_start(first_connection_uuid)

# Prints {'exitCode': 0, 'stdout': 'hello world', 'stderr': ''}
print(client.shell_exec(first_connection_uuid, "echo hello world"))

# Clean up after ourselves by stopping the shell session
client.shell_stop(first_connection_uuid)
```

There's also an async version of the client that can be accessed as AsyncClient:

```python
import asyncio
from xpipe_api import AsyncClient

async def main():
    # By default, Client() will read an access key from the file xpipe_auth on the local filesystem
    # and talk to the XPipe HTTP server on localhost.  To connect to a remote instance with an API
    # key, use Client(token="foo", base_url = "http://servername:21721")
    client = AsyncClient()

    # connection_query accepts glob-based filters on the category, connection name, and connection type
    all_connections = await client.connection_query()

    # Each connection includes uuid, category, connection, and type information
    first_connection_uuid = all_connections[0]["uuid"]

    # Before any shell commands can be run, a shell session must be started on a connection
    await client.shell_start(first_connection_uuid)

    # Prints {'exitCode': 0, 'stdout': 'hello world', 'stderr': ''}
    print(await client.shell_exec(first_connection_uuid, "echo hello world"))

    # Clean up after ourselves by stopping the shell session
    await client.shell_stop(first_connection_uuid)


if __name__ == "__main__":
    asyncio.run(main())
```

## Getting connection information

Since each connection reference is just a UUID string, you have to retrieve information for it using another API call.

Reference: http://localhost:21721/#connection-information


```python
from xpipe_api import Client

client = Client()

# connection_query accepts glob-based filters on the category, connection name, and connection type
connections = client.connection_query(categories="Default/**")

# Works in bulk, so it expects an array of connections
infos = client.connection_info(connections)
# Get the first info
info = infos[0]

uuid = info["connection"]
# This is an array containing all category hierarchy names
category = info["category"]
# This is an array containing all connection hierarchy names
name = info["name"]
# This is the connection type name that you can use in the query
type = info["type"]
# There is also some other data for internal state management, e.g. if a tunnel is running for example
```

## Querying connections


```python
from xpipe_api import Client

client = Client()

# connection_query accepts glob-based filters on the category, connection name, and connection type
connections = client.connection_query(categories="Default/**")

# Find only ssh config host connections in the "MyCategory" category that has the word "MyCompany" in its name
# - Searching for strings is case-insensitive here
# - This will not include subcategories of "MyCategory"
# - You can find all the available connection type names by just querying all connections and taking a look add the individual type names of the connections.
connections = client.connection_query(categories="MyCategory", types="sshConfigHost", connections="*MyCompany*")

# Query a specific connection by name in a category
# This will only return one element, so we can just access that
connection = client.connection_query(categories="MyCategory", connections="MyConnectionName")[0]

# A connection reference is just a UUID string, so you can also specifiy it fixed
# If you have the HTTP API setting enabled in XPipe, you can copy the API UUID of a connection in the context menu
connection = "f0ec68aa-63f5-405c-b178-9a4454556d6b"
```

## GUI actions

You can perform some actions for the desktop application from the CLI as well.

```python
from xpipe_api import Client

client = Client()

connection = "f0ec68aa-63f5-405c-b178-9a4454556d6b"

# Opens the file browser in a specified starting directory for a connection
client.connection_browse(connection, "/etc")

# Opens a terminal session in a specified starting directory for a connection
client.connection_terminal(connection, "/etc")
```

## Connection actions

You can perform some actions for the desktop application from the CLI as well.

```python
from xpipe_api import Client

client = Client()

connection = "?"

# Toggles the session state for a connection. If this connection is a tunnel, then this operation will start or stop the tunnel
client.connection_toggle(connection, True)

# Refreshes a connection. This operation is the same as for validating a connection when creating it by attempting whether the connection is functioning.
client.connection_refresh(connection)
```

This is only a short summary of the library. You can find more supported functionalities in the source itself.

## Shells

You can perform some actions for the desktop application from the CLI as well.

```python
from xpipe_api import Client
import re

client = Client()

connection = "f0ec68aa-63f5-405c-b178-9a4454556d6b"

# This will start a shell session for the connection
# Note that this session is non-interactive, meaning that password prompts are not supported
shell_info = client.shell_start(connection)

# The shell dialect of the shell. For example cmd, powershell, bash, etc.
dialect = shell_info["shellDialect"]
# The operating system type of the system. For example, windows, linux, macos, bsd, solaris
osType = shell_info["osType"]
# The display name of the operating system. For example Windows 10 Home 22H2
osName = shell_info["osName"]

# Prints {'exitCode': 0, 'stdout': 'hello world', 'stderr': ''}
print(client.shell_exec(connection, "echo hello world"))

# Prints {'exitCode': 0, 'stdout': '<user>', 'stderr': ''}
print(client.shell_exec(connection, "echo $USER"))

# Prints {'exitCode': 127, 'stdout': 'invalid: command not found', 'stderr': ''}
print(client.shell_exec(connection, "invalid"))

# Extract ssh version from system
version_format = re.compile(r'[0-9.]+')
ret = client.shell_exec(connection, "ssh -V")
ssh_version = version_format.findall(ret["stderr"])[0]

# Clean up after ourselves by stopping the shell session
client.shell_stop(connection)
```

## File system operations

You can perform some actions for the desktop application from the CLI as well.

```python
from xpipe_api import Client
import re

client = Client()

connection = "f0ec68aa-63f5-405c-b178-9a4454556d6b"

# We need a shell for the file system
client.shell_start(connection)

# You are responsible for decoding files in the correct file encoding
file_bytes = client.fs_read(connection, "~/.ssh/config")
file_string = file_bytes.decode('utf-8')

file_write_content = "test\nfile"
blob_id = client.fs_blob(file_write_content)
client.fs_write(connection, blob_id, "~/test_file.txt")

script_write_content = "echo hello\necho world"
blob_script_id = client.fs_blob(file_write_content)
script_path = client.fs_script(connection, blob_script_id)

# Prints {'exitCode': 0, 'stdout': 'hello\nworld', 'stderr': ''}
print(client.shell_exec(connection, "\"" + script_path + "\""))

# Clean up after ourselves by stopping the shell session
client.shell_stop(connection)
```


This is only a short summary of the library. You can find more supported functionalities in the source itself.


## Tests

To run the test suite, you'll need to define the XPIPE_APIKEY env var.  This will allow the two "log in with the ApiKey 
rather than Local method" tests to work.  Here's the recommended method for running the tests with poetry:

```commandline
cd /path/to/xpipe-python-api
poetry install
XPIPE_APIKEY=<api_key> poetry run pytest
```