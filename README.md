# Description

This is a bash shell tool for maintaining AWS credentials in one or more shells.

# Synopsis

```sh
source session-tool.sh
```
This can be added to your `.profile`, `.bashrc` or similar.


# Usage

The session-tool.sh shell function definition file contains commands
for managing your AWS session credentials. This is useful for `terraform` and
`aws` cli commands.

## get_session

`get_session [-h] [-s] [-r] [-l] [-c] [-d] [-u] [-p profile] [MFA token]`

 * `MFA token`    Your one time token. If not provided, and you provided
                  the -s option, the current credentials are stored.
 * `-p profile`   The aws credentials profile to use as an auth base.
                  The provided profile name will be cached, and be the
                  new default for subsequent calls to get_session.
                  Default: awsops
 * `-s`           Save the resulting session to persistent storage
                  for retrieval by other shells. You will be prompted
                  twice for a passphrase to protect the stored credentials.
 * `-r`           Restore previously saved state. You will be prompted for
                  the passphrase you stated when storing the session.
 * `-l`           List currently stored sessions including a best guess on
                  when the session expires based on file modification time.
 * `-c`           Resets session.
 * `-f`           Fetches company-wide roles descriptions to ~/.aws/bf-roles.cfg
                  These entries can be overwritten in ~/.aws/roles.cfg
                  Fetching is done before getting the session token, using only
                  the permissions granted by the profile.
* `-h`           Print this usage.

This command will on a successful authentication return
session credentials for the Basefarm main account. The credentials
are returned in the form of environment variables suitable for the `aws`
cli and `terraform`. The returned session has a duration of 12 hours.

At least one of -s, -r or MFA token needs to be provided.

Session state is stored in: `~/.aws/awsops.aes`

## assume_role

`assume_role [-h] [-l] [role alias]`

* `-h`          Print this usage.
* `-l`          List available role aliases.
* `role alias`  The alias of the role to assume. The alias name will be cached,
                so subsequent calls to get_console_url will use the cached value.

This command will use session credentials stored in the shell
from previous calls to get_session The session credentials are
then used to assume the given role.

This command will also set the AWS_CONSOLE_URL containing a
pre-signed url for console access.

The session credentials for the assumed role will replace the
current session in the shell environment. The only way to retrieve
the current session after an assume_role is to have stored your
session using get_session with the -s option and then to
import them again using get_session -r command.

The assumed role credentials will only be valid for one hour,
this is a limitation in the underlaying AWS assume_role function.

The selected role alias will be cached in the AWS_ROLE_ALIAS environment
variable, so you do not have to provide it on subsequent calls to assume_role.

Roles are configured in ~/.aws/roles.cfg in addition to the company-wide roles.

## get_console_url

`get_console_url [-h] [-l] [role alias]`

* `-h`          Print this usage.
* `-l`          List available role aliases.
* `role alias`  The alias of the role to assume. The alias name will be cached,
                so subsequent calls to get_console_url will use the cached value.

This command will use session credentials stored in the shell
from previous calls to get_session. The session credentials are
then used to assume the given role and finally to create
a pre-signed URL for console access.

## aws-assume-role

`aws-assume-role profile role_alias MFA_token`

This command combines `get_session`, `assume_role` and `get_console_url`.
It is included only to provide backwards compatibility.

# Files

## ~/.aws/bf-roles.cfg

This file contains the predefined roles that you may assume given your Basefarm main
account credentials, assuming you are member of the proper groups.
Updates to this file can be retrieved using `get_session -f`. 

## ~/.aws/roles.cfg
This file contains your personalized overrides and additions to `bf-roles.cfg`
Lines starting with a # are treated as comments. All other
lines must contain a roles definition line. Each line is a space separated
list containing these elements:

```
alias role_arn session_name external_id
```
* `alias`A random name you assign to this role_arn.
* `role_arn` The aws arn of the role you want to assume.
* `session_name` A tag that is added to your login trail.
* `external_id` The external ID assisiated with this role.

The tree first are mandatory, while `external_id` is optional, end the line after
`session_name` if `external_id` is not provided.


# Prerequisites

You must have a user in the Basefarm main account. This user must be able to
assume roles in the needed customer account or other Basefarm accounts. You must
have this credentials profile stored in your `~.aws/credentials` file. This is
usually done using the `aws configure` command.

These commands assume that you have a profile called *awsops*. You might use a
different name, but must then provide the profile name when initializing
a session.

* `openssl`   Used to encrypt/decrypt session state to file.
              Only needed if you use the -s or -r
              options to get_session command.
* `date`      On Max OSX it uses the nativ date command.
              On Linux it assumes a GNU date compatible version.
* `aws`       The aws CLI must be avialable and in the PATH.
* `curl`      Used only for getting console URL.
* `python`    Used for normalizing JSON.
* `json.tool` Python library for parsing JSON.
* `test`, `grep`, `egrep`, `awk` and `sed`.

# Environment variables

The tool export to the current shell a lot of variables. Some are required for
`aws` and `terraform` cli support, others are maintained for the benefit of
the user and some are needed by the tool itself.

Required for cli access to aws resources:
* `AWS_SESSION_TOKEN`
* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY_ID`
* `AWS_PROFILE` The name of the credentials profile. Not needed to auth with the above credentials.

Maintained for the user and need the for the tool itself:
* `AWS_EXPIRATION_LOCAL` The time when the current session expires in the current locale.
* `AWS_CONSOLE_URL` The URL to the aws console, created by the assume_role command.
* `AWS_CONSOLE_URL_EXPIRATION_LOCAL` The time when the aws console url session expires in the current locale.
* `AWS_USER` The arn of the current authenticated user.
* `AWS_SERIAL` The arn of the MFA instance for the current user.
* `AWS_ROLE_ALIAS` The alias of the last used role.
* `AWS_EXPIRATION` The time when the current session expires as received from aws (in Zulu time).
* `AWS_EXPIRATION_S` The time when the current session expires converted to seconds since the epoch.
* `AWS_CONSOLE_URL_EXPIRATION` The time when the aws console url session expires as received from aws (in Zulu time).
* `AWS_CONSOLE_URL_EXPIRATION_S` The time when the aws console url session expires converted to seconds since the epoch.

# Example workflow

These examples assume that you already have added the session-tool.sh to your
.profile (or similar) and that the AWS CLI is installed.

Initial setup consists of configuring an AWS profile and adding credentials to it:

```sh
aws configure --profile awsops
```

The user starts by initializing a session, providing his MFA token:

```sh
get_session -f 123456
```

Note that the `-f` flag is used to ensure that company-wide roles are updated.
The user now has his environment populated with AWS variables that are
suitable to for example run terraform (with assume_role).

The user then needs to open another terminal and have the credentials follow him.
First, the user must then store the existing credentials to file:

```sh
get_session -s
```
The user is prompted twice for the passphrase to protect the stored credentials.

> The user can also provide the `-s` option during the initial authentication
> (using his MFA token), saving him this step.

Then the user can open *another terminal* and restore/import the stored
credentials:
```sh
get_session -r
```

Now the user want's to assume a role within the Basefarm lab environment and
perform some `aws` cli commands:
```sh
assume_role -l
assume_role awsopslab-admin
aws iam list-users
```
Then he need to access the AWS management console:
```sh
get_console_url
```
The returned URL can then be pasted into a browser to gain temporary access to
the management console in the context of the assumed account.

Because the user first ran the assume_role command, the get_console_url will
just echo the AWS_CONSOLE_URL environment variable and the console session will
only last until the assume_role session expires.

At any time (both for the Basefarm main account session and the assume_role
session) the user can query the AWS_EXPIRATION_LOCAL variable to get the end
time of the current session.

Once the assume_role session is expired (after one hour), the credentials are no
longer valid and the user must either re-authenticate or restore a previously
saved session.

## (Long) Example with multiple calls to assume_role

Since we store a copy of the credentials returned by get_session, we can re-use them for doing
multiple calls of assume_role and get_console_url:
```sh
[bent@c7vm ~]$ get_session -c
[bent@c7vm ~]$ get_session 123456
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"basefarm-operations"
[bent@c7vm ~]$ assume_role bf-awsopslab-admin
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"bf-awsopslab"
[bent@c7vm ~]$ assume_role bf-awsops-admin
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"basefarm-operations"
[bent@c7vm ~]$ assume_role bf-awsopslab-admin
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"bf-awsopslab"
[bent@c7vm ~]$ get_console_url bf-awsops-admin
https://signin.aws.amazon.com/federation?Action=login&Issuer=&Destination=https%3a%2f%2fconsole.aws.amazon.com%2f&SigninToken=xmnh8ELFeXJRaz-qV9jOOVE_m1kqBOu-l1LyabMK7Hc1Sr3EM1HungasdhaskdhjkoBmFObn0DfkJ9Kko.....
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"bf-awsopslab"
[bent@c7vm ~]$ assume_role bf-awsopslab-failtest

An error occurred (AccessDenied) when calling the AssumeRole operation: Not authorized to perform sts:AssumeRole
ERROR: Unable to obtain session
[bent@c7vm ~]$ aws iam list-account-aliases | jq ".AccountAliases|.[]"
"bf-awsopslab"
[bent@c7vm ~]
```
# Known issues

* If you do not have an awsops profile or you change the profile name to one that does not exists in your credentials file, aws cli commands will fail. You need to unset the AWS_PROFILE variable or use these tools to set a new value: `get_session -p <profile> <mfa>`.
* The assume_role command is only able to create sessions that last for one hour. This is an AWS limitation. Once the session has expired, you must re-authenticate or manually restore a previously saved session.
* It is considered best practice to use the build in assume role support in terraform, so you would for terraform purposes only use the get_session command.
* Some AWS CLI commands require Python 3 (https://www.linkedin.com/pulse/aws-cli-requires-python3-bent-terp)

# Authors
Initial work by [Daniel Abrahamsson](https://github.com/danabr) and [Bent Terp](https://github.com/bentterp), adapted and re-worked by
[Bjørn Røgeberg](https://github.com/bjornrog) and [Bent Terp](https://github.com/bentterp).
