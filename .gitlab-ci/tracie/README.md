Tracie - Mesa Traces Continuous Integration System
==================================================

Home of the Mesa trace testing effort.

### Traces definition file

The trace definition file contains information about the git repo/commit to get
the traces from, and a list of the traces to run along with their expected image
checksums on each device. An example:

```yaml
traces-db:
  repo: https://gitlab.freedesktop.org/gfx-ci/tracie/traces-db
  commit: master

traces:
  - path: glmark2/jellyfish.rdc
    expectations:
      - device: gl-intel-0x3185
        checksum: 58359ea4caf6ad44c6b65526881bbd17
      - device: gl-vmware-llvmpipe
        checksum: d82267c25a0decdad7b563c56bb81106
  - path: supertuxkart/supertuxkart-antediluvian-abyss.rdc
    expectations:
      - device: gl-intel-0x3185
        checksum: ff827f7eb069afd87cc305a422cba939
```

The traces-db entry can be absent, in which case it is assumed that the
current directory is the traces-db directory.

Traces that don't have an expectation for the current device are skipped
during trace replay.

Adding a new trace to the list involves commiting the trace to the git repo and
adding an entry to the `traces` list. The reference checksums can be calculated
with the [image_checksum.py](.gitlab-ci/tracie/image_checksum.py) script.
Alternatively, an arbitrary checksum can be used, and during replay (see below)
the scripts will report the mismatch and expected checksum.

### Trace-db repos

The trace-db repos are assumed to be git repositories using LFS for their trace
files. This is so that trace files can be potentially checked out and replayed
individually, thus reducing storage requirements during CI runs.

### Enabling trace testing on a new device

To enable trace testing on a new device:

1. Create a new job in .gitlab-ci.yml. The job will need to be tagged
   to run on runners with the appropriate hardware.

   1. If you mean to test GL traces, use the `.traces-test-gl`
      template jobs as a base, and make sure you set a unique value for the
     `DEVICE_NAME` variable:

   ```yaml
   my-hardware-gl-traces:
     extends: .traces-test-gl
     variables:
       DEVICE_NAME: "gl-myhardware"
   ```

   2. If you mean to test Vulkan traces, use the `.traces-test-vk`
      template jobs as a base, set the `VK_DRIVER` variable, and make
      sure you set a unique value for the `DEVICE_NAME` variable:

   ```yaml
   my-hardware-vk-traces:
     extends: .traces-test-vk
     variables:
       VK_DRIVER: "radeon"
       DEVICE_NAME: "vk-myhardware"
   ```

2. Update the .gitlab-ci/traces.yml file with expectations for the new device.
   Ensure that the device name used in the expectations matches the one
   set in the job. For more information, and tips about how to calculate
   the checksums, see the section describing the trace definition files.

### Trace files

Tracie supports renderdoc (.rdc), apitrace (.trace) and gfxreconstruct
(.gfxr) files. Trace files need to have the correct extension so that
tracie can detect them properly.

The trace files that are contained in public traces-db repositories must be
legally redistributable. This is typically true for FOSS games and
applications. Traces for proprietary games and application are typically not
redistributable, unless specific redistribution rights have been granted by the
publisher.

### Replaying traces

Mesa traces CI uses a set of scripts to replay traces and check the output
against reference checksums.

The high level script [tracie.sh](.gitlab-ci/tracie/tracie.sh) accepts
a traces definition file and the type of traces
(apitrace/renderdoc/gfxreconstruct) to run:

    tracie.sh .gitlab-ci/traces.yml renderdoc

tracie.sh copies produced artifacts to the `$CI_PROJECT_DIR/result`
directory. By default, created images from traces are only stored in case of a
checksum mismatch. The `TRACIE_STORE_IMAGES` CI/environment variable can be set
to `1` to force storing images, e.g., to get a complete set of reference
images.

The `tracie.sh` script requires that the environment variable `DEVICE_NAME` is
properly set for the target machine, and matches the `device` field of the
relevant trace expectations in the used `traces.yml` file.

At a lower level the
[dump_trace_images.py](.gitlab-ci/tracie/dump_trace_images.py) script is
called, which replays a trace, dumping a set of images in the process. By
default only the image corresponding to the last frame of the trace is dumped,
but this can be changed with the `--calls` parameter. The dumped images are
stored in a subdirectory `test/<device-name>` next to the trace file itself,
with names of the form `tracefilename-callnum.png`.  The full log of any
commands used while dumping the images is also saved in a file in the
'test/<device-name>' subdirectory, named after the trace name with '.log'
appended.

Examples:

    python3 dump_traces_images.py --device-name=gl-vmware-llvmpipe mytrace.trace
    python3 dump_traces_images.py --device-name=gl-vmware-llvmpipe --calls=2075,3300 mytrace.trace

### Running the replay scripts locally

It's often useful, especially during development, to be able to run the scripts
locally.

Depending on the target 3D API, the scripts require a recent version
of apitrace being in the path, and also the renderdoc python module
being available, for GL traces.

To ensure python3 can find the renderdoc python module you need to set
`PYTHONPATH` to point to the location of `renderdoc.so` (binary python modules)
and `LD_LIBRARY_PATH` to point to the location of `librenderdoc.so`. In the
renderdoc build tree, both of these are in `renderdoc/<builddir>/lib`. Note
that renderdoc doesn't install the `renderdoc.so` python module.

In the case of Vulkan traces, the scripts need a recent version of
gfxrecon-replay being in the path, and also the
`VK_LAYER_LUNARG_screenshot` Vulkan layer from LunarG's VulkanTools.

To ensure that this layer can be found when running the trace you need
to set `VK_LAYER_PATH` to point to the location of
`VkLayer_screenshot.json` and `LD_LIBRARY_PATH` to point to the
location of `libVkLayer_screenshot.so`.
