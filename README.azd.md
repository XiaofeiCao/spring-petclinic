# Spring Petclinic with azd

The template owns Bicep. The `azure.springboot` extension provides the
`springbootapp` host, prepares the Maven container, PostgreSQL password, and
firewall rules, then deploys the authored Bicep service module and waits for
readiness. Native container services build and publish the image. No deployment
hooks or authored Dockerfile are needed.

## Development prerequisites

This is an unpublished prototype, not a template for an arbitrary released azd.
Use the matching [development azd build and Spring extension](https://github.com/XiaofeiCao/azure-dev/tree/xiaofeicao-spring-boot-template-roadmap/cli/azd/extensions/azure.springboot),
Java 17/Maven Wrapper, and a running Docker-compatible engine. Install the local
extension bundle produced by that development build:

```text
azd extension install <path-to-azure.springboot_0.1.0.zip>
azd auth login
```

The template's version checks do not replace this matching-development-build
prerequisite.

## H2 and PostgreSQL

Start each example in its own empty directory outside the Azure Dev repository.
Select your Azure subscription and location when prompted by `azd up`, or set
`AZURE_SUBSCRIPTION_ID` and `AZURE_LOCATION` in the azd environment beforehand.

```text
mkdir petclinic-h2
cd petclinic-h2
azd init -t https://github.com/XiaofeiCao/spring-petclinic --branch azd-springboot-test . -e petclinic-h2 --no-prompt
azd up
```

In another empty directory:

```text
mkdir petclinic-postgresql
cd petclinic-postgresql
azd init -t https://github.com/XiaofeiCao/spring-petclinic --branch azd-springboot-test . -e petclinic-postgresql --no-prompt
azd env set SPRING_DATABASE_TYPE postgresql
azd up
```

H2 is the default and loses application data when the running instance is
replaced. PostgreSQL keeps data separately. Use a new azd environment to change
database mode. The template owns the Container Apps virtual network and dedicated
infrastructure subnet with its attached network security group, so this topology
requires a fresh azd environment; do not retrofit or switch an already provisioned
Container Apps environment in place. The subnet disables default outbound access,
and its network security group allows public inbound HTTP and HTTPS traffic to the
public Container Apps ingress. PostgreSQL also remains public and is restricted to
observed Container Apps egress addresses, not private database connectivity.

For a new PostgreSQL environment, supply `SPRING_DATABASE_PASSWORD` through the
process/CI environment before the first provision, or let the extension generate
and persist it locally. Reruns reuse the original password. If local secret state
is lost, restore the original password from your secret store through that process
variable before provisioning again; it is never silently regenerated or rotated.
Do not print, commit, or pass real passwords as command-line arguments.

This demo uses the PostgreSQL administrator login, local azd secret state, and
a native Container Apps secret, not Key Vault or passwordless authentication.
PostgreSQL has public networking restricted to observed Container Apps egress
addresses, not private networking. If replacement changes those addresses, retry
`azd deploy` to reconcile them. These are demo trade-offs, not production guidance.
PostgreSQL TLS validates the CA chain using the JRE trust store and checks the
hostname without a per-user `root.crt` file.

Clean up each deployed environment from its own directory:

```text
azd down --force --purge --no-prompt
```

This removes the template-owned virtual network, subnet, and network security
group with the environment. Container Apps platform-managed networking resources
may incur charges while provisioned.

Local azd state, including the password, is retained after cleanup; protect it.

## Source and local checks

Application source is unchanged from `spring-projects/spring-petclinic` commit
`500158f732419217507c7656904b8e6aa1bcc0d6`. `source-revision.json` records its
provenance and selected LF-normalized SHA-256 checksums. The extension E2E harness
pins this testing branch's template commit separately from that upstream commit.

From the development Azure Dev checkout's `cli/azd` directory:

```powershell
go build -o .\azd.exe .
$env:CLI_TEST_AZD_PATH = (Resolve-Path .\azd.exe).Path
$env:AZD_TEST_SPRING_PETCLINIC_INIT = '1'
$env:AZD_TEST_SPRING_PETCLINIC_BUILD = '1'
$env:AZD_TEST_SPRING_PETCLINIC_LIVE = '0'
Set-Location extensions\azure.springboot
go test ./... -timeout 25m
```

These checks install a real local bundle, initialize both modes, and build local
containers without deploying Azure resources. Live deployment tests require
separate Azure-cost authorization and an authenticated disposable test config.
