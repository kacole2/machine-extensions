# Extensions for Docker Machine

### Premise
Docker Machine gets you a "docker ready" host, but what about everything else that integrates with Docker Engine? Docker Engine 1.8 released the pluggable architecture for 3rd party network and storage extensions. However, most of these require installation processes that must be done after the host is provisioned. The Docker Machine Extensibility feature is an interface for specifying the installation of Docker Engine Plugins to install during the host deployment.

Closed [PR #1881](https://github.com/docker/machine/pull/1881) is to address [Proposal / Issue #1765](https://github.com/docker/machine/issues/1765). The discussion will continue on [Issue #773 - Proposal: Machine Declaration](https://github.com/docker/machine/issues/773).

Download the binaries: [kacole2/machine/releases/tag/v0.4.1-ext](https://github.com/kacole2/machine/releases/tag/v0.4.1-ext)
Get the code: [kacole2/machine/tree/feature/extension](https://github.com/kacole2/machine/tree/feature/extension)

### Extensions Supported:
  - [REXray](https://github.com/emccode/rexray) : stable
  - [Weave](https://github.com/weaveworks/weave) : stable
  - Generic - install services through a JSON file without customization

### Summary
Extensions for Docker Machine is a pluggable architecture to install Docker Engine Plugins as an additional command line argument.

```
docker-machine -D create --driver amazonec2 --amazonec2-access-key mykey --amazonec2-secret-key mysecretkey --amazonec2-vpc-id myvpcid --amazonec2-zone b --extension /Users/kcoleman/Desktop/extensions.json dev
```

The name of the `.json` file is arbitrary. This allows the creation of multiple `.json` files for different machines provisioned through Docker Machine. At this time, `.json` files are the only supported file types.

The `.json` file supports the following structure and fields:
```
{
    "extensions": [
        {
            "extension-name": {
                "version": "latest",
                "env": {
                    "any environment": "variables"
                },
                "params": {
                    "specific key values": "for extension customization"
                },
                "copy": {
                    "source path/file": "destination path/file"
                },
                "files": {
                    "filename for extension customization": {
                        "source": "source path/file",
                        "destination": "destination path/file"
                    }
                },
                "run": [
                    "commands",
                    "to",
                    "run"
                ],
                "validOS": [
                    "OSnames",
                    "ubuntu"
                ]
            }
        }
    ]
}
```

A generic installation of a service using only commands lines:
```
{
    "extensions": [
        {
            "generic": {
                "envs": {},
                "copy": {
                    "/Users/kcoleman/Desktop/rexconfig.yaml": "~/.rexray/config.yaml"
                },
                "run": [
                    "curl -sSL https://dl.bintray.com/emccode/rexray/install | sh",
                    "sudo service rexray start"
                ],
                "validOS": [
                    "ubuntu",
                    "debian",
                    "centos",
                    "redhat"
                ]
            }
        }
    ]
}
```

A simple example of installing a service (Weave as an example):
```
//weave-node-01 (ie. 192.168.10.101)
{
    "extensions": [
        {
            "weave": {}
        }
    ]
}
```

```
//weave-node-02 
{
    "extensions": [
        {
            "weave": {
                "params": {
                    "peer": "192.168.10.101"
                }
            }
        }
    ]
}
```

a more complex example with extensions running in priority:
```
{
    "extensions": [
        {
            "generic": {
                "copy": {
                    "/Users/kcoleman/Desktop/rexconfig.yaml": "/etc/rexray/config_backup1.yaml"
                }
            }
        },
        {
            "weave": {}
        },
        {
            "rexray": {
                "version": "latest",
                "files": {
                    "config.yaml": {
                        "source": "/Users/kcoleman/Desktop/rexconfig.yaml",
                        "destination": "/etc/rexray/config.yaml"
                    }
                }
            }
        },
        {
            "generic": {
                "copy": {
                    "/Users/kcoleman/Desktop/rexconfig2.yaml": "/etc/rexray/config_backup3.yaml"
                }
            }
        }
    ]
}
```

### Deeper Dive
The `.json` file is read from a local resource into memory. 

The name of the extension is checked to see if a registered extension exists. If there is a matching registered extension, the paramters in the `.json` file are passed on.
  - `version` will set the value from `nil` to a `string` in the `extInfo` struct. If it's not specified and the value `nil` is passed along, the extension's `.go` file will have a default or `latest` version specified. Versioning allows consistent deployment of the extension in the case that a specific `docker-engine` version is being specified in the docker machine command. Each extension is responsible for the implementation of installing different versions.
  - `env` is a set of key:value pairs that can call the `utils.go` function `appendEnvFile` to set environment variables on the host. 
  - `params` is any set of key:value pairs that an extension can use for customization. These pairs are passed on to the extension's interface where the extension can use them any way it wishes. Such use cases include commands to run or local variables for configuration.
  - `copy` is a key:value pair that will take the source and destination and transfer the files using `docker-machine scp`. This takes no priority and files can be transferred at will when called.
  - `files` is a nested set of key:value pairs for the transfering of files using the `docker-machine scp` command. This function is used for customizing extensions. The name specified in the `.json` file must match in the function called. This allows you to transfer files at any time in your code. This gives you the freedom and flexibiltiy to transfer files when a function may need them. Files can be transferred from the local system to remote systems or between remote systems. Use cases include configuration files or binaries to be copied. There is a generic function in `utils.go` that will take the `source` and `destination` keys to transfer files.
  - `run` is an array of commands that will be run in sequential order. These are piped into the `provisioner.SSHCommand()`.
  - `validOS` is an array of supported Operating Systems to check. This also allows your to run commands based on the type of OS that is detected from the `.json` file.

Before the installation begins, the `.json` file is parsed to make sure there is valid JSON. This is denoted in the `host.go` file.
```
if h.HostOptions.ExtensionOptions.File != "" {
    if err := extension.ParseExtensionFile(*h.HostOptions.ExtensionOptions); err != nil {
      return err
    }
  }
```

The installation of any extension is performed as the final step during the docker-machine configuration process. This is denoted in the `host.go` file. It checks to see if the `--extension` flag isn't empty:
```
if h.HostOptions.ExtensionOptions.File != "" {
      if err := extension.ExtensionInstall(*h.HostOptions.ExtensionOptions, provisioner); err != nil {
        return err
      }
    }
```

