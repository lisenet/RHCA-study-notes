# Study notes for EX180 Red Hat Certified Specialist in Containers and Kubernetes
_by Tomas Nevar (tomas@lisenet.com)_

## Exam objectives:

* Implement images using Podman
    * Understand and use FROM (the concept of a base image) instruction.
    * Understand and use RUN instruction.
    * Understand and use ADD instruction.
    * Understand and use COPY instruction.
    * Understand the difference between ADD and COPY instructions.
    * Understand and use WORKDIR and USER instructions.
    * Understand security-related topics.
    * Understand the differences and applicability of CMD vs. ENTRYPOINT instructions.
    * Understand ENTRYPOINT instruction with param.
    * Understand when and how to expose ports from a Docker file.
    * Understand and use environment variables inside images.
    * Understand ENV instruction.
    * Understand container volume.
    * Mount a host directory as a data volume.
    * Understand security and permissions requirements related to this approach.
    * Understand lifecycle and cleanup requirements of this approach.
* Manage images
    * Understand private registry security.
    * Interact with many different registries.
    * Understand and use image tags.
    * Push and pull images from and to registries.
    * Back up an image with its layers and meta data vs. backup a container state.
* Run containers locally using Podman
    * Get container logs.
    * Listen to container events on the container host.
    * Use Podman inspect.
* Basic OpenShift knowledge
* Creating applications in OpenShift
   * Create, manage and delete projects from a template, from source code, and from an image.
   * Customise catalog template parameters.
   * Specifying environment parameters.
   * Expose public applications.
* Troubleshoot applications in OpenShift
   * Understand the description of application resources.
   * Get application logs.
   * Inspect running applications.
   * Connecting to containers running in a pod.
   * Copy resources to/from containers running in a pod.
