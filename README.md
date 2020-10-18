# nRF5 SDK for mesh 4.0.0 : adding `math` to CMake build breaks SES project generation

Adding `-lm` to the `CMakeLists.txt` of a project in the mesh stack 
(here `light_switch_server` is used), breaks SES project file generation
with `GENERATE_SES_PROJECTS`.

The associated Nordic DevZone Ticket is [here](https://devzone.nordicsemi.com/f/nordic-q-a/53500/adding-math-library-with--lm-to-cmake-build-of-nrf-mesh-breaks-ses-project-generation/217130).

Reproduction sequence is below.

NOTE that the original README text from Nordic was moved to
[README_original.md](./README_original.md).

## Reproduction

1. Install tooling:

    - gcc (this was built with `8.3.1 20190703 (release)`)
    - cmake
    - ninja
    - jinja2
    - markupsafe

    Details outside the scope of this document, see
    https://infocenter.nordicsemi.com/index.jsp?topic=%2Fstruct_sdk%2Fstruct%2Fsdk_mesh_latest.html

1. Clone this repo and set up the build:

        git clone https://github.com/siriobalmelli/nrf_mesh_4.0.0_lm.git
        cd nrf_mesh_4.0.0_lm
        mkdir build && cd build
        cmake -GNinja ..
        ninja nRF5_SDK  # will install nRF52 SDK in the directory above nrf_mesh_4.0.0_lm
        cmake -DGENERATE_SES_PROJECTS=OFF -GNinja ..

1. Build `light_switch_server` successfully:

        $ ninja light_switch_server_nrf52832_xxAA_s132_7.0.1
        # ... some output elided for clarity
           text    data     bss     dec     hex filename
         143248    1152   14672  159072   26d60 examples/light_switch/server/light_switch_server_nrf52832_xxAA_s132_7.0.1.elf

1. Attempting to generate SES projects will fail:

        $ cmake -DGENERATE_SES_PROJECTS=ON -GNinja ..
        # ... some output elided for clarity
        CMake Error at CMake/GenerateSESProject.cmake:23 (get_property):
          get_property could not find TARGET m.  Perhaps it has not yet been created.
        Call Stack (most recent call first):
          examples/light_switch/server/CMakeLists.txt:72 (add_ses_project)

1. This is caused by `m` added to `target_link_libraries`
in `examples/light_switch/server/CMakeLists.txt`:

        target_link_libraries(${target}
            rtt_${PLATFORM}
            uECC_${PLATFORM}
            m)

1. If the `m` link target is removed, SES generation succeeds:

        target_link_libraries(${target}
            rtt_${PLATFORM}
            uECC_${PLATFORM}
        #    m)
            )

        $ cmake -DGENERATE_SES_PROJECTS=ON -GNinja ..
        # ... some output elided for clarity
        -- Configuring done
        -- Generating done

1. ... but this leads to link errors when trying to build `light_switch_server`:

        $ ninja light_switch_server_nrf52832_xxAA_s132_7.0.1
        # ... some output elided for clarity
        in function `main':
        examples/light_switch/server/src/main.c:303: undefined reference to `exp'

1. Attempts to use `set_target_link_options` do not work; both of these fail:

        # try '-lm'
        set_target_link_options(${target}
            ${CMAKE_CURRENT_SOURCE_DIR}/linker/${PLATFORM}_${SOFTDEVICE}
            -lm)

        # try 'lm'
        set_target_link_options(${target}
            ${CMAKE_CURRENT_SOURCE_DIR}/linker/${PLATFORM}_${SOFTDEVICE}
            lm)

## Conclusion

Please consider either:

- Patching the build system to properly generate SES files when linking with `-lm`
    in `target_link_libraries` as above.
- Providing a special target for the math library which plays nice with SES
    project generation.
