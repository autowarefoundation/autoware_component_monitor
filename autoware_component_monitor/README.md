<!-- cspell:ignore LXCFS, sysinfo -->

# autoware_component_monitor

The `autoware_component_monitor` package allows monitoring system usage of component containers.
The composable node inside the package is attached to a component container, and it publishes CPU and memory usage of
the container.

## Inputs / Outputs

### Input

None.

### Output

| Name                       | Type                                               | Description            |
| -------------------------- | -------------------------------------------------- | ---------------------- |
| `~/component_system_usage` | `autoware_internal_msgs::msg::ResourceUsageReport` | CPU, Memory usage etc. |

## Parameters

### Core Parameters

| Parameter      | Type   | Default | Description        |
| -------------- | ------ | ------- | ------------------ |
| `publish_rate` | number | `5.0`   | Publish rate in Hz |

The [parameter schema](schema/component_monitor.schema.json) and the [parameter file](config/component_monitor.param.yaml) define this default.

## How to use

Add it as a composable node in your launch file:

```xml

<launch>
  <group>
    <push-ros-namespace namespace="your_namespace"/>
    ...

    <load_composable_node target="$(var container_name)">
      <composable_node pkg="autoware_component_monitor"
                       plugin="autoware::component_monitor::ComponentMonitor"
                       name="component_monitor">
        <param from="$(find-pkg-share autoware_component_monitor)/config/component_monitor.param.yaml"/>
      </composable_node>
    </load_composable_node>

    ...
  </group>
</launch>
```

### Quick testing

You can test the package by running the following command:

```bash
ros2 component load <container_name> autoware_component_monitor autoware::component_monitor::ComponentMonitor -p publish_rate:=10.0 --node-namespace <namespace>

# Example usage
ros2 component load /pointcloud_container autoware_component_monitor autoware::component_monitor::ComponentMonitor -p publish_rate:=10.0 --node-namespace /pointcloud_container
```

## How it works

The package uses the `top` command under the hood.
`top -b -n 1 -p PID` command is run at `publish_rate` (5 Hz by default) to get the system usage of the process.

- `-b` activates the batch mode. By default, `top` doesn't exit and prints to stdout periodically. Batch mode allows
  exiting the program.
- `-n` number of times should `top` prints the system usage in batch mode.
- `-p` specifies the PID of the process to monitor.

Here is a sample output:

```text
top - 10:25:47 up 18:24,  1 user,  load average: 1.34, 1.38, 1.43
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  6.7 us,  2.1 sy,  0.0 ni, 91.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem : 128731.0 total,  49006.2 free,  24414.5 used,  56026.2 buff/cache
MiB Swap: 150464.0 total, 150464.0 free,      0.0 used. 104316.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
3230885 mfc       20   0   10408   3804   3424 S   0.0   0.0   0:00.00 bash
```

We get 5th, 8th fields from the last line, which are RES, %CPU respectively.

The total and free system memory come from the [`sysinfo(2)`](https://man7.org/linux/man-pages/man2/sysinfo.2.html) system call, not from the `top` output.
Under [LXCFS](https://github.com/lxc/lxcfs), for example in LXC or LXD containers, `sysinfo()` reports the memory of the host, not of the container.
