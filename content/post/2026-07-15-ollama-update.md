+++
date = '2026-07-15T21:46:51+02:00'
draft = false
title = '2026 07 15 Ollama Update to v0.30'
+++

Ollama version 0.30 has been released some time ago, bringing a major change: it now use llama.cpp instead of the internal GGML fork (see [v0.30.0](https://github.com/ollama/ollama/releases?page=2#release-v0.30.0))

This has a big impact on my Strix Halo + Rocm setup

## Perfomance on ollama 0.24.0

Using gemma4.31b, it runs 100% on the GPU

```text
NAME          ID              SIZE     PROCESSOR    CONTEXT    UNTIL
gemma4:31b    6316f0629137    47 GB    100% GPU     262144     4 minutes from now
```

And using a simple question `what is json ?' I obtained an answer at ~10 token per second.

```text
total duration:       1m8.796615034s
load duration:        164.085088ms
prompt eval count:    17 token(s)
prompt eval duration: 205.756478ms
prompt eval rate:     82.62 tokens/s
eval count:           702 token(s)
eval duration:        1m8.1488854s
eval rate:            10.30 tokens/s
```

## Performance on ollama v0.30.5

After upgrading to v0.30.5, ollama-rocm now detect both the CPU and the GPU

```text
jui 15 19:18:57 phi25 ollama[1548]: device_info:
jui 15 19:18:57 phi25 ollama[1548]:   - CPU     : AMD RYZEN AI MAX+ 395 w/ Radeon 8060S (31727 MiB, 31727 MiB free)
jui 15 19:18:57 phi25 ollama[1548]:   - ROCm0   : AMD Radeon 8060S Graphics (98304 MiB, 29071 MiB free)
```

And now, the modele is spread across the two devices

```text
NAME          ID              SIZE     PROCESSOR          CONTEXT    UNTIL
gemma4:31b    6316f0629137    34 GB    57%/43% CPU/GPU    262144     4 minutes from now
```

And this has a major impact on perfomance, as the token per second is almsot devided by two.

```text
total duration:       1m44.78409003s
load duration:        309.925333ms
prompt eval count:    17 token(s)
prompt eval duration: 770.637ms
prompt eval rate:     22.06 tokens/s
eval count:           689 token(s)
eval duration:        1m43.701536s
eval rate:            6.64 tokens/s
```

An issue about this is opened on github: [#16462](https://github.com/ollama/ollama/issues/16462)

<!--more-->

## Changing ROCm to Vulkan

As the new llama.cpp backend now support Vulkan, let's try to switch.

To do this, we just need to use the `ollama-vulkan` package and add some environment variables.

```nix
  services = {
    ollama = {
      enable = true;
      host = "0.0.0.0";

      package = pkgs.ollama-vulkan;

      environmentVariables = {
        OLLAMA_DEBUG = "2";
        HIP_VISIBLE_DEVICES = "-1";
        GGML_VK_VISIBLE_DEVICES = "0";
        OLLAMA_VULKAN = "1";
        OLLAMA_IGPU_ENABLE = "1";
      };
    };
```

The GPU is properly detected.

```text
jui 15 20:31:20 phi25 ollama[11181]: Available devices:
jui 15 20:31:20 phi25 ollama[11181]:   Vulkan0: AMD Radeon 8060S Graphics (RADV STRIX_HALO) (114167 MiB, 114001 MiB free)
```

And the modele is running 100% on the GPU

```text
NAME          ID              SIZE     PROCESSOR    CONTEXT    UNTIL
gemma4:31b    6316f0629137    20 GB    100% GPU     262144     4 minutes from now
```

And this bring back the performance, even better than before going from 10.6 to 11 token per second on `gemma4:31b`.

```text
total duration:       1m6.864675587s
load duration:        317.519047ms
prompt eval count:    17 token(s)
prompt eval duration: 428.764ms
prompt eval rate:     39.65 tokens/s
eval count:           728 token(s)
eval duration:        1m6.115625s
eval rate:            11.01 tokens/s
```
