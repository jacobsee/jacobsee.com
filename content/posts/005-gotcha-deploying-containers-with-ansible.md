---
title: "Gotcha: Configuring Containers with Ansible"
description: "Be careful with this sneaky reason that your mounted files might not behave as expected!"
date: 2021-04-07
---

If you're running serious container infrastructure these days, chances are strong that you're using an orchestration system like Kubernetes or OpenShift. But, what if your workloads are a little less... _clustered_? A little more traditional? Maybe you have containers as a part of your workload, but don't need to migrate the entire system to them? Running one-off containers on normal Linux hosts is still _very much a thing_, it's just more on you to manage them - perhaps using an automation tool like Ansible. I was doing this recently, and hit a bit of a snag, so I thought I'd write about it here!

To kick things off - running a container on a target machine in Ansible is a pretty easy task. Depending on your target container type, you can use the [docker_container](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_container_module.html) or [podman_container](https://docs.ansible.com/ansible/latest/collections/containers/podman/podman_container_module.html) modules in a task like so:

```yaml
- name: Run the container
  docker_container:
    name: my_container
    image: "{{ my_exporter_image }}:{{ my_exporter_image_version }}"
    state: "started"
```

Fair enough.

Reading through the documentation for those modules above, we see that we can also mount files from the host into the container as volumes - an important thing to do when a container takes some kind of a configuration file in order to run. Since it's likely that we're also creating that config file with Ansible, let's take a look at what writing that file and then mounting it into the container would look like:

```yaml
- name: write the config file
  template:
    src: config.yml.j2
    dest: "/home/{{ ansible_user }}/my_config.yml"
            # ^ we're writing the config file here

- name: Run the container
  docker_container:
    name: my_container
    image: "{{ my_exporter_image }}:{{ my_exporter_image_version }}"
    state: started
    volumes:
    - "/home/{{ ansible_user }}/my_config.yml:/config.yml"
        # ^ and mounting it into the container here
```

And there we go! The `my_config.yml` file in the home directory of whichever user is running this task is now mounted into the container at `/config.yml`.

This works nicely... **once**. This is where the "gotcha" is!

If we were to run this task a second time with different values for the config file, we would notice some interesting behavior. The file on the host (at `/home/{{ ansible_user }}/my_config.yml`) would appear to have our new configuration in it, but the application in the container would not be performing as expected! In fact, if we were to `ssh` into the remote machine and run `docker exec my_container cat /config.yml` to see the contents of the config file _from the container's perspective_, we would see that it has retained the old config data from before we ran the task a second time! _Why have the file on the host and the file in the container diverged?_

The answer lies in a combination of how Ansible writes files, and how Docker mounts them. Files in a UNIX-based filesystem are represented by `inodes`, which are records in a table that contain metadata about those files (including where they physically exist on a disk). If you're familiar with hard links and symbolic links - congratulations! You're already manipulating `inodes`! Hard links are paths on the filesystem that point at an underlying `inode`, and symbolic links are paths on the filesystem that act as shortcuts to other paths.

Two things are important to know at this point, and they are that:

- Containers mount a file by referencing that file's `inode` at the container's creation time, and:
- When creating files, Ansible first writes the data to a temporary file, and then _moves_ that temporary file into its final location

This is important because, on the filesystem level, Ansible is actually creating a _new_ `inode` every time it writes this data - even though, intuitively, it seems like we're writing to the same path every time. It then updates the hard link at that path to point at this newly created `inode`. This makes sense as it is actually a safer way for Ansible to write data remotely, but it means that any already-running container with that path (and therefore the old `inode`) mounted **do not get updated with our newly-written config data**.

There are two easy fixes for this.

First, we could force a container restart every time we make a change to the config file. The Ansible module has a `restart` flag to make this easy:

```yaml
- name: write the config file
  template:
    src: config.yml.j2
    dest: "/home/{{ ansible_user }}/my_config.yml"
            # ^ we're writing the config file here

- name: Run the container
  docker_container:
    name: my_container
    image: "{{ my_exporter_image }}:{{ my_exporter_image_version }}"
    state: started
    restart: yes
    volumes:
    - "/home/{{ ansible_user }}/my_config.yml:/config.yml"
        # ^ and mounting it into the container here
```

However, if our application can live-reload its config and we don't want to force a restart every time we make a change to the config file, we have to get slightly more creative. Notice how earlier we said that _moving_ a file changes the `inode` referenced by a certain path. _Copying_ a file does not have this same effect. We can use this to our advantage!

```yaml
- name: write the config file (temporary)
  template:
    src: config.yml.j2
    dest: "/home/{{ ansible_user }}/my_config.yml.tmp"

- name: copy the config file to the correct path (so that the inode stays intact)
  command: "cp /home/{{ ansible_user }}/my_config.yml.tmp /home/{{ ansible_user }}/config.yml"

- name: remove the temporary config file
  file:
    path: "/home/{{ ansible_user }}/my_config.yml.tmp"
    state: absent
```

By doing this, we have ensured that the `inode` at the path mounted into the container is not modified, and our new configuration is ready to use in the container immediately!

Happy automating!
