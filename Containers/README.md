# Containerizing Traefik on Service Fabric
As Service Fabric rolls out more container related features, you may wish to default on containers as your packaging format. However, be aware that containerizing Traefik on Service Fabric does introduce some added complexities. Traefik needs to be able to speak to the Service Fabric API. It typically does this over http[s]://localhost:19080, depending on your container network setup, Traefik running inside a container won't be able to reach this endpoint. You can potentially inject the host IP into the `.toml` at runtime but this can get messy quite quickly. The easiest method is to simply point Traefik at the public IP address of the load balancer handling calls through to the Traefik nodes. If you do go down this route, please ensure you encrypt your data by using TLS.

# Traefik container images
There are 2 Dockerfiles provided in this folder:
* Linux - `Dockerfile`
* Windows - `Dockerfile.win`

Both Dockerfiles are meant as a reference for a simple container deployment, please modify appropriately for production workloads. The Dockerfiles expect to be built in a directory with the following structure:
* Dockerfile
* servicefabric.crt
* servicefabric.key
* traefik.toml

Before you build the image, update your `traefik.toml`, making sure the `clustermanagementurl` will be accessible from inside the container i.e. your Public IP or DNS.

Service Fabric can only reference containers stored on remote container registries. Once you've built this container image locally, tag it appropriately and push it up to a repository on your container registry.

**Warning**: This container image will contain your key and certificate files, please only push this image to a private container registry.

# Public container images
Containous currently ONLY host Traefik for Linux container images on their [Dockerhub repository](https://hub.docker.com/_/traefik/). These images provide a minimum distribution of Traefik and are meant to be extended to add security features like certificates. For simple demonstrations on non-secure clusters, you can use these images directly.

# Service Fabric application configuration
Once you've got your container image stored on your remote registry, you need to create a Service Fabric Container Application. Please refer to the official Service Fabric Documentation for [Windows containers](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started-containers) and [Linux containers](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started-containers-linux).
The essential bits as far as Traefik is concerned are:
* Adding a communication endpoint for Traefik and the dashboard (optional) in the `ServiceManifest.xml`
```
<Endpoints>
      <Endpoint Name="TraefikContainerTypeEndpoint" UriScheme="http" Port="80" Protocol="http" />
      <Endpoint Name="TraefikContainerTypeDashboardEndpoint" UriScheme="http" Port="8080" Protocol="http" />
</Endpoints>
```
* Adding a Port-to-Host port mapping for each defined Traefik endpoint
```
<Policies>
    <ContainerHostPolicies CodePackageRef="Code">
        <PortBinding ContainerPort="8080" EndpointRef="TraefikContainerTypeDashboardEndpoint"/>
        <PortBinding ContainerPort="80" EndpointRef="TraefikContainerTypeEndpoint"/>
    </ContainerHostPolicies>
</Policies>
```
* Configuring authentication to your container registry

Once you've configured your Service Fabric Container Application, you can simply deploy it to your Service Fabric cluster and use it as normal. If you are running in Azure and want to be able to ingress requests to Traefik's dashboard, you'll need to create a new Health Probe and Load Balancing Rule to map an external port to 8080.

**Note** At this present time, Linux container images are only supported on a Linux Service Fabric cluster and Windows Server container images are only supported on a Windows Service Fabric cluster. For Windows clusters, please use the image `WindowsServer2016WithContainers` or later.

