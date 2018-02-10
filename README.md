# go-metadataproxy

The go-metadataproxy is used to allow containers to acquire IAM roles. By metadata we mean [EC2 instance meta data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) which is normally available to EC2 instances. This proxy exposes the meta data to containers inside or outside of EC2 hosts, allowing you to provide scoped IAM roles to individual containers, rather than giving them the full IAM permissions of an IAM role or IAM user.

## Installation

From inside of the repo run the following commands:

```bash
dep ensure
go install
```

## Configuration

The go-metadataproxy has 1 mode of operation, running in AWS where it simply proxies most routes to the real metadata service.

### AWS credentials

go-metadataproxy relies on AWS Go SDK for its AWS credentials. If metadata
IAM credentials are available, it will use this. Otherwise, you'll need to use
.aws/credentials or environment variables to specify the IAM
credentials before the service is started.

### Role assumption

For IAM routes, the go-metadataproxy will use STS to assume roles for containers.
To do so it takes the incoming IP address of metadata requests and finds the
running docker container associated with the IP address. It uses the value of
the container's `IAM_ROLE` environment variable as the role it will assume. It
then assumes the role and gives back STS credentials in the metadata response.

STS-attained credentials are cached and automatically rotated as they expire.

#### Container-specific roles

To specify the role of a container, simply launch it with the `IAM_ROLE`
environment variable set to the IAM role you wish the container to run with.

```shell
docker run -e IAM_ROLE=my-role ubuntu:14.04
```

#### Configurable Behavior

There are a number of environment variables that can be set to tune
metadata proxy's behavior. They can either be exported by the start
script, or set via docker environment variables.

| Variable | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| **DEFAULT\_ROLE** | String | | Role to use if IAM\_ROLE is not set in a container's environment. If unset the container will get no IAM credentials. |
| **DEFAULT\_ACCOUNT\_ID** | String | | The default account ID to assume roles in, if IAM\_ROLE does not contain account information. If unset, go-metadataproxy will attempt to lookup role ARNs using iam:GetRole. |
| DEBUG | Boolean | False | Enable debug mode. You should not do this in production as it will leak IAM credentials into your logs |
| DOCKER\_URL | String | unix://var/run/docker.sock | Url of the docker daemon. The default is to access docker via its socket. |

#### Default Roles

When no role is matched, `go-metadataproxy` will use the role specified in the
`DEFAULT\_ROLE` `go-metadataproxy` environment variable. If no DEFAULT\_ROLE is
specified as a fallback, then your docker container without an `IAM\_ROLE`
environment variable will fail to retrieve credentials.

#### Role Formats

The following are all supported formats for specifying roles:

- By Role:

    ```shell
    IAM_ROLE=my-role
    ```

- By Role@AccountId

    ```shell
    IAM_ROLE=my-role@012345678910
    ```

- By ARN:

    ```shell
    IAM_ROLE=arn:aws:iam::012345678910:role/my-role
    ```

### Role structure

A useful way to deploy this go-metadataproxy is with a two-tier role
structure:

1.  The first tier is the EC2 service role for the instances running
    your containers.  Call it `DockerHostRole`.  Your instances must
    be launched with a policy that assigns this role.

2.  The second tier is the role that each container will use.  These
    roles must trust your own account ("Role for Cross-Account
    Access" in AWS terms).  Call it `ContainerRole1`.

3.  go-metadataproxy needs to query and assume the container role.  So
    the `DockerHostRole` policy must permit this for each container
    role.  For example:
    ```
    "Statement": [ {
        "Effect": "Allow",
        "Action": [
            "iam:GetRole",
            "sts:AssumeRole"
        ],
        "Resource": [
            "arn:aws:iam::012345678901:role/ContainerRole1",
            "arn:aws:iam::012345678901:role/ContainerRole2"
        ]
    } ]
    ```

4. Now customize `ContainerRole1` & friends as you like

Note: The `ContainerRole1` role should have a trust relationship that allows it to be assumed by the `user` which is associated to the host machine running the `sts:AssumeRole` command.  An example trust relationship for `ContainRole1` may look like:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::012345678901:root",
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Routing container traffic to go-metadataproxy

Using iptables, we can forward traffic meant to 169.254.169.254 from docker0 to
the go-metadataproxy. The following example assumes the go-metadataproxy is run on
the host, and not in a container:

```
/sbin/iptables \
  --append PREROUTING \
  --destination 169.254.169.254 \
  --protocol tcp \
  --dport 80 \
  --in-interface docker0 \
  --jump DNAT \
  --table nat \
  --to-destination 127.0.0.1:8000 \
  --wait
```

If you'd like to start the go-metadataproxy in a container, it's recommended to
use host-only networking. Also, it's necessary to volume mount in the docker
socket, as go-metadataproxy must be able to interact with docker.

Be aware that non-host-mode containers will not be able to contact
127.0.0.1 in the host network stack.  As an alternative, you can use
the meta-data service to find the local address.  In this case, you
probably want to restrict proxy access to the docker0 interface!

```
LOCAL_IPV4=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)

/sbin/iptables \
  --append PREROUTING \
  --destination 169.254.169.254 \
  --protocol tcp \
  --dport 80 \
  --in-interface docker0 \
  --jump DNAT \
  --table nat \
  --to-destination $LOCAL_IPV4:8000 \
  --wait

/sbin/iptables \
  --wait \
  --insert INPUT 1 \
  --protocol tcp \
  --dport 80 \
  \! \
  --in-interface docker0 \
  --jump DROP
```

## Run go-metadataproxy without docker

In the following we assume \_my\_config\_ is a bash file with exports for all of
the necessary settings discussed in the configuration section.

```
source my_config
cd /srv/go-metadataproxy
go run main.go
```

## Run go-metadataproxy with docker

For production purposes, you'll want to kick up a container to run.
You can build one with the included Dockerfile.  To run, do something like:
```bash
docker run --net=host \
    -v /var/run/docker.sock:/var/run/docker.sock \
    jippi/go-metadataproxy
```

## Attribution

This project is a ~1:1 port of [lyft/metadataproxy](https://github.com/lyft/metadataproxy), done in Go.

## Contributing

### File issues in Github

In general all enhancements or bugs should be tracked via github issues before
PRs are submitted. We don't require them, but it'll help us plan and track.

When submitting bugs through issues, please try to be as descriptive as
possible. It'll make it easier and quicker for everyone if the developers can
easily reproduce your bug.

### Submit pull requests

Our only method of accepting code changes is through github pull requests.